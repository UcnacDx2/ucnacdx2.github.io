---
title: "在小米电视上换掉原厂桌面：TvService、Binder 事务号和读回确认"
description: "一台把设置入口藏起来的电视，一个只能以 uid 0 执行的诊断域，一组会随 ROM 漂移的事务号。换桌面这件事，最后靠的是每一步都读回确认。"
pubDate: 2026-10-27
tags:
  - Android TV
  - Android
  - Binder
  - Root
draft: false
---

这篇对应实际项目：

<https://github.com/UcnacDx2/mitv-home-bridge>

目标：把一台小米电视的原厂桌面换掉，换成能自己控制的第三方桌面；顺带处理原厂升级组件和安装器限制。

听起来像“装个 Launcher 就行”，实际不是。原厂桌面是系统组件，设置入口被藏起来，卸载入口也没有。能改的地方只有两条：`TvService` 和 `su`。

## 两条路径，优先级不能反

这台 ROM 上，`/system/bin/service call TvService 4400 ...` 会让 `misysdiagnose` 域以 `uid=0` 执行一段脚本。这条路不需要用户授权 root，也不弹 Magisk 提示。

`su` 是第二条路，只在 TvService 失败时用。

之所以强调顺序：`su` 第一次调用可能卡在 Magisk 的授权弹窗上。如果主流程排在它后面，用户看到的就是“点开应用，桌面卡住不动”。所以代码里把安装器限制那条放到独立线程：

```java
new Thread(() -> {
    disableUpgrade();
    if (fromTvHome) runOnUiThread(this::prepareAlternativeHome);
    else openTvHome();
}, "MiTvHomeBridge").start();

new Thread(this::removeInstallerRestriction, "MiTvHomeBridge-Installer").start();
```

两条线程共享的东西只有 TvService，后面的文件命名问题就出在这里。

## TvService 调用长什么样

```java
new ProcessBuilder("/system/bin/service", "call", "TvService",
        "4400", "s16", "s", "s16", scriptFile.getAbsolutePath())
```

`4400` 是这台 ROM 上的事务号，参数是一个脚本路径。脚本最终被复制到 `/data/diagnosis/command.sh` 执行，这也是一个坑：**脚本无法通过 `$0` 推导自己的目录**，所以结果文件路径必须由 App 写死在脚本里传进去。

诊断域能访问的外部存储是 `/sdcard`，而 App 看到的是自己的 `Download` 目录，两边指向同一个位置。

## 没有 stdout，所以用文件当返回值

`service call` 的回复永远是一个空 Parcel，脚本里的 `echo` 不会回到调用方。要对结果有把握，只能让脚本把结论写进文件：

```sh
V=/sdcard/Download/mitv-home-bridge.3.result
OUT=$(/system/bin/service call appops 31 i32 23 i32 10123 s16 com.android.packageinstaller i32 2 2>&1)
case "$OUT" in
  *Error*|*'does not exist'*) echo "FAIL call=[$OUT]" > $V; exit;;
esac
```

App 侧不睡固定时间，而是轮询这个文件：

```java
for (int attempt = 0; attempt < (waitForVerdict ? 20 : 3); attempt++) {
    String line = readFirstLine(verdictFile);
    if (line != null) return line.trim();
    Thread.sleep(150);
}
```

调用前先把结果文件清空。这样脚本中途死掉时读到的是空，而不是上一次操作留下的 `OK`。

## 并发踩过的坑：固定文件名会互相删

最早脚本和结果文件用的是固定名字。结果在电视上复现出一个很难查的问题：force-stop 的结果读回来是空的。

原因不是 force-stop 失败，而是**另一条线程（换桌面）在同一时刻清理了自己的文件**——两条线程共用 TvService，也共用文件名：

```text
线程 A 写 command.sh，执行
线程 B 写 command.sh，覆盖 A 的脚本
A 执行完，删除 command.sh 和 result
B 去读 result，已经被 A 删了
```

修法很朴素：每次操作生成唯一序号，脚本和结果文件成对命名，`finally` 里只删自己那一对。

```java
String operation = FILE_PREFIX + OPERATION_SEQUENCE.incrementAndGet();
```

这类 bug 在单线程测试里永远不会出现，只有在真机上两条流程同时跑才会露头。

## `pm disable` 不等于 `disable-user`

这是这个 ROM 另一个反直觉的地方。

设置组件状态时，`PackageManager.COMPONENT_ENABLED_STATE_DISABLED` 是 `2`，`DISABLED_USER` 是 `3`。用 `3` 去禁用原厂桌面，即使 `uid=0` 也会被拒绝：

```text
invalid new component state: 3
```

更糟的是，`pm` 对框架拒绝的修改依然返回退出码 0。只看退出码，会得到“成功”的假象。

所以 su 路径统一用 `pm disable`（状态 2），并且在执行后读回真实状态：

```java
int applied = getComponentState(component);
return applied == state;
```

**退出码只能证明命令跑过，不能证明状态变了。**

## 换桌面的护栏

直接 `set-home-activity` 有个致命风险：如果新桌面起不来，而原厂桌面已经被禁用，电视会停在一个没有桌面的状态。

所以流程是“先准备退路，再切换”：

```text
1. 禁用 FallbackHome（原厂兜底桌面）
2. 枚举能响应 HOME 的第三方桌面
3. 没有候选 → 恢复 FallbackHome，保留原厂桌面，退出
4. 多个候选 → 提示用户只保留一个，恢复原状
5. 校验候选桌面真的能启动
6. 禁用原厂 TvHome 组件
7. 设置默认 Home 并启动
```

任何一步失败，都回到“恢复 FallbackHome + 启用原厂桌面”。

注意第 6 步的顺序：这台 ROM 在原厂 Home 组件仍启用时，会忽略 `setHomeActivity`。所以必须先禁用原厂组件，再设默认桌面——但这也意味着第 5 步的“能启动”校验必须先做完。

## 安装器限制

原厂安装器有自己的拦截开关，存在系统设置 `pi_config` 里：

```json
{"pi_intercept_switch": false, "app_pi_control": false}
```

只要它被写回去，侧载就会被拦。处理思路是三步：

1. 拒绝安装器的 `WRITE_SETTINGS` appop；
2. 删除并重写 `pi_config`；
3. force-stop 安装器，让它重新读取。

有 root 时三步一起做；没有 root 时，appops 和 force-stop 可以走 TvService，但 `settings` 写入只能留给用户或另一个带 root 服务的模块。

这里同样有事务号漂移的问题：`IAppOpsService.setMode` 是 31，`IActivityManager.forceStopPackage` 是 83，`OP_WRITE_SETTINGS` 是 23——**这些数字是在特定 ROM 版本上测出来的，换版本可能就变**。所以脚本里加了失败识别：

```sh
case "$OUT" in
  *Error*|*'does not exist'*) echo "FAIL call=[$OUT]" > $V; exit;;
esac
```

事务号不对时，Binder 会回 “Not a data message”，而不是执行一个错误操作。

## 为什么每一步都要读回

这个项目里几乎没有“调用成功即完成”的操作：

| 操作 | 读回方式 |
| --- | --- |
| 禁用组件 | `getComponentEnabledSetting` |
| 禁用包 | `getApplicationEnabledSetting` |
| 设置 Home | `resolveActivity` 能否启动 |
| appop | `dumpsys appops` 里该包的 WRITE_SETTINGS 状态 |

框架可以接受一次调用却不改变状态，`pm` 还会返回 0。只有读回的值能作为结论。

## 边界

- **只对验证过的 ROM 负责。** 事务号、被藏起来的组件名、FallbackHome 的行为都和 ROM 版本绑定。换机型就得重新测。
- **不保留“直接调 PackageManager”的路径。** 这个 ROM 上普通应用没有 `CHANGE_COMPONENT_ENABLED_STATE`，那条路必然失败，留着只会误导。
- **不做常驻服务。** 它是一个短生命周期的控制器，点开、执行、退出，不驻留后台。
- **失败一律回到原厂状态。** 宁可不换，也不留一个起不来的桌面。

## 这个项目真正解决的是什么

不是“换桌面”这个动作，而是把一组不可靠的操作变成可验证的步骤：

```text
调用可能被拒 → 读回状态
脚本可能没跑 → 检查 verdict 文件
事务号可能漂移 → 识别错误输出
并发可能互删 → 每次操作独立命名
```

在系统边界上做事，最重要的不是权限有多大，而是**每一步都能证明它真的发生了**。
