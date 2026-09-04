---
title: "MiTV OTA 机型库怎么落地：ADB 采集、在线验证和敏感字段加密"
description: "不是简单收集电视型号：我实际怎样从 getprop 生成 OTA 参数、用小米 OTA 接口验证贡献，再把 SN / 设备身份加密后用于持续监测。"
pubDate: 2026-09-04
tags:
  - Android TV
  - OTA
  - ADB
  - Cloudflare
draft: false
---

这篇对应实际项目：

<https://github.com/UcnacDx2/mitv-ota-monitor>

之前我把它写成了“想做一个电视固件地图”，但那样没有说清楚最有价值的部分：**机型数据不是用户填完表单就直接进库，而是先从设备采参数，再拿这组参数去请求小米 OTA，验证它真的能返回固件后才接受。**

## 问题不是少一个“型号”字段

电视商品页上的型号，和 OTA 服务真正使用的设备身份不是一回事。

项目里至少要处理这些值：

```text
displayName
product
codename
device
module
currentVersion
lang
serial
deviceIdentity
```

其中 `serial` 和 `deviceIdentity` 还属于敏感设备信息，不能跟普通机型元数据一样直接公开。

所以我把流程拆成了：

```text
ADB 读取
  ↓
生成规范化参数
  ↓
服务端格式校验
  ↓
真实请求 Xiaomi OTA
  ↓
确认能拿到版本和 package
  ↓
机型公开入库
  ↓
SN / deviceIdentity 加密保存，仅供后续监测
```

## 第一步：不是让贡献者手抄十个 `getprop`

实际代码在：

```text
components/contribution-form.tsx
```

页面直接生成了一条 ADB 命令，一次输出表单需要的完整字段：

```sh
adb shell 'product=$(getprop ro.short_assm_mn); codename=$(getprop ro.product.device); printf "displayName=%s\nproduct=%s\ncodename=%s\ndevice=%s.%s\nmodule=%s.%s.firmware\nminimumKnownVersion=%s\nlang=%s\nserial=%s\ndeviceIdentity=%s\n" "$(getprop ro.product.model)" "$product" "$codename" "$product" "$codename" "$product" "$codename" "$(getprop ro.build.version.incremental)" "$(getprop ro.product.locale)" "$(getprop ro.serialno)" "$(getprop mitv.factory.mac)"'
```

这条命令实际读取的是：

| 输出字段 | Android 属性 |
| --- | --- |
| `displayName` | `ro.product.model` |
| `product` | `ro.short_assm_mn` |
| `codename` | `ro.product.device` |
| `minimumKnownVersion` | `ro.build.version.incremental` |
| `lang` | `ro.product.locale` |
| `serial` | `ro.serialno` |
| `deviceIdentity` | `mitv.factory.mac` |

另外两个值默认按规则拼出来：

```text
device = product.codename
module = product.codename.firmware
```

例如我当时实机验证过的组合是：

```text
codename = finch
product  = OBPCN1N
```

重点是：贡献者不用理解 OTA 参数怎么拼，只需要打开 ADB、运行一条只读命令、把输出交给表单。

## 第二步：表单提交后先做格式校验

提交接口在：

```text
app/api/contribute/route.ts
```

服务端不会直接信任用户输入。

它先对几个设备标识做正则约束：

```ts
const MODEL_VALUE = /^[A-Za-z0-9._-]{2,96}$/;
const VERSION_VALUE = /^[A-Za-z0-9._-]{3,96}$/;
const LANG_VALUE = /^[A-Za-z]{2}_[A-Za-z]{2}$/;
```

如果用户没有手工覆盖，服务端也会重新生成默认值：

```ts
const device = input.device || `${product}.${codename}`;
const moduleName = input.module || `${product}.${codename}.firmware`;
const lang = input.lang || 'zh_CN';
```

也就是说，前端的默认拼接只是用户体验，**真正落库前服务端会再做一次相同约束。**

## 第三步：贡献频率不是靠前端按钮防刷

接口会取：

```text
cf-connecting-ip
user-agent
```

然后做 SHA-256：

```ts
createHash('sha256')
  .update(`${ip}\n${userAgent}`)
  .digest('hex')
```

再交给：

```ts
claimContributionWindow(...)
```

做提交窗口限制。

如果提交过快，直接返回：

```text
429
提交过于频繁，请一分钟后再试
```

这个 fingerprint 不是拿来当用户身份，而只是一个不直接保存 IP 文本的简单限流键。

## 第四步：最关键的一步——拿贡献参数去真实请求 Xiaomi OTA

服务端先构造公开配置：

```ts
const config = {
  displayName,
  product,
  device,
  module: moduleName,
  lang,
  currentVersion,
};
```

然后把敏感字段只在请求阶段合进去：

```ts
const status = await checkXiaomiOta({
  ...config,
  serial,
  deviceIdentity,
});
```

接下来不是“请求没报错就算成功”，而是至少满足：

```ts
status.ok
status.latestVersion
status.packages.length > 0
```

否则接口直接返回 `422`：

```text
小米 OTA 未确认该组信息可获得更新，未加入机型库
```

这一步就是整个机型库质量控制的核心。

贡献者写了一个看起来很像真的 `product/codename` 并没有用；**小米 OTA 服务自己必须能认这组身份。**

## 第五步：验证成功后到底写了哪些东西

如果 OTA 验证通过，会继续执行：

```ts
await upsertCommunityModel(env.DB, config, status);
await archiveOtaObservation(env.DB, config, status);
```

也就是：

- 更新社区机型记录；
- 保存这一次 OTA 观察结果。

如果 OTA 返回的最新版本和贡献者当前版本不同，还会记录一次版本对：

```ts
if (status.latestVersion !== currentVersion) {
  await recordVersionProbe(...)
}
```

这也是为什么项目后来不仅能展示“最新版本”，还能逐步积累：

```text
某个源版本
→ 被 OTA 服务导向哪个目标版本
```

这种历史关系。

## 第六步：SN 和设备身份不能跟机型数据一起公开

OTA 查询又确实需要：

```text
serial
deviceIdentity
```

所以项目没有把它们放到公开 GitHub 数据里。

接口要求服务器配置 `CHECK_TOKEN`，然后：

```ts
const encrypted = await encryptMonitorCredentials(
  env.CHECK_TOKEN,
  { serial, deviceIdentity },
);

await saveMonitorCredentials(
  env.DB,
  config,
  encrypted.iv,
  encrypted.ciphertext,
);
```

也就是说网页可以公开：

```text
这个机型是什么
当前看到什么版本
有哪些 OTA 包
```

但持续查询需要的设备身份是加密保存的，不显示在网页，也不提交进 Git 仓库。

## 前端为什么显示“验证并贡献机型”而不是“提交”

`ContributionForm` 的按钮实际写的是：

```text
验证并贡献机型
```

因为提交成功的含义不是：

> 我收到了你的数据。

而是：

> **我拿你的数据问过 Xiaomi OTA，它确实返回了有效更新信息，然后我才把它纳入机型库。**

成功响应里会带：

```text
latestVersion
packageCount
monitoring: true
```

前端收到后显示：

```text
验证通过，已加入机型库。最新版本：...
```

## 这个项目真正解决的是什么

如果只是做一个“电视型号 Wiki”，其实没有必要写这么多后端逻辑。

这个项目更像一条数据验证流水线：

```text
真实设备属性
      ↓
可复现的 ADB 采集
      ↓
规范化 OTA 参数
      ↓
官方 OTA 服务在线验证
      ↓
公开机型元数据 + OTA 观察历史
      ↓
敏感设备身份加密后持续监测
```

所以我后来觉得它最有价值的地方，不是网页上多了多少型号，而是：**每一条社区数据都尽可能带着“这组参数真的被 OTA 服务接受过”的证据。**