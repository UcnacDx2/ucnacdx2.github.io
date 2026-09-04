---
title: "NetCertFix 适配 1.2026.237：不是类名变了，而是旧类名被 R8 复用了"
description: "从 ChatGPT Android 1.2026.230 到 1.2026.237 的真实 Hook 失效案例：旧短类名仍存在却已指向别的类，导致 DexKit fallback 永远不执行。"
pubDate: 2026-09-04
tags:
  - Android
  - Xposed
  - DexKit
  - 逆向
draft: false
---

这篇文章对应我自己的 NetCertFix 仓库：

<https://github.com/UcnacDx2/NetCertFix>

关键修复 commit：

<https://github.com/UcnacDx2/NetCertFix/commit/ae4985ea2085f311d7c808c89942f6a902dd672b>

问题发生在 ChatGPT Android 从：

```text
1.2026.230  versionCode 2623032
```

升级到：

```text
1.2026.237  versionCode 2623716
```

以后。

旧版本工作正常，新版本 Hook 失效。

最容易给出的解释是“App 混淆升级了，类名变了”。但这次真正的问题反而更坑：**旧类名没有消失。**

## 先说 NetCertFix 原本 Hook 的是什么

这个模块不是关闭 TLS 校验，也不是绕过证书 pinning。

它只处理 ChatGPT Android 在某些可正常工作的代理 / VPN 环境里误触发的：

```text
untrusted network certificate
```

全屏错误状态。

我的目标是找到 certificate status 对应的 StateFlow，并阻止那个“坏证书状态”写进去。

TLS 验证、证书 pinning 和普通网络错误路径都保持原样。

初版针对 `1.2026.230` 验证过的混淆目标是：

```text
service class: i6w
bad state:     f6w
flow method:   m
```

同时我已经写了一个自适应 fallback：如果这些硬编码类找不到，就使用 DexKit，根据字符串：

```text
NetworkCertificateStatusService
```

去找真正的 service class。

听上去已经“适配混淆”了，但 1.2026.237 还是挂了。

## 真正的 bug：`i6w/f6w` 在新版本里居然还存在

旧逻辑的顺序是：

```text
1. 先 findClass("i6w") / findClass("f6w")
2. 如果找不到，才执行 DexKit
```

问题在于 R8 的短类名会被重新分配。

到了 `1.2026.237`，`i6w/f6w` 这两个名字仍然能被 ClassLoader 找到，但它们已经是**无关的媒体类**。

于是程序发生了一个很典型的“假成功”：

```text
findClass("i6w") 成功
        ↓
代码认为旧目标仍然有效
        ↓
DexKit fallback 不执行
        ↓
Hook 安装在错误对象上 / 功能失效
```

所以这次问题不是：

> 找不到旧类名。

而是：

> **找得到旧类名，但它已经不是原来的语义。**

这个区别非常重要。

## 修改一：把 DexKit 从 fallback 改成 primary path

修复后，顺序直接反过来。

旧逻辑：

```java
Class<?> serviceClass = findFirst(appCl, SERVICE_CANDIDATES);
badStateClass = findFirst(appCl, BAD_STATE_CANDIDATES);

if (serviceClass == null) {
    serviceClass = resolveWithDexKit(lpparam);
}
```

新逻辑变成：

```java
Class<?> serviceClass = resolveWithDexKit(lpparam);
badStateClass = null;

if (serviceClass == null) {
    Class<?>[] fallback = resolveVerifiedFallback(appCl);
    if (fallback != null) {
        serviceClass = fallback[0];
        badStateClass = fallback[1];
    }
}
```

也就是：

```text
稳定语义锚点 / DexKit
        ↓
结构验证
        ↓
实在不行才用版本配对的短类名
```

而不是反过来。

## 修改二：短类名必须按“版本配对”使用

原来 service 和 bad-state 类是两组独立候选：

```java
SERVICE_CANDIDATES = { "i6w" }
BAD_STATE_CANDIDATES = { "f6w" }
```

这会有一个问题：即使将来分别碰巧找到两个同名类，也不代表它们属于同一个版本、同一条业务链。

所以我改成了配对 fallback：

```java
private static final String[][] VERIFIED_TARGETS = {
    { "wcy", "tcy" }, // 1.2026.237
    { "i6w", "f6w" } // 1.2026.230
};
```

这样 fallback 的语义变成：

```text
要么命中一整组已验证组合
要么不启用
```

而不是从不同版本的 R8 短名里拼一个“看起来存在”的组合。

## 修改三：类名命中以后还要做结构验证

仅仅 `findClass()` 成功已经不能证明它就是目标。

所以我加了：

```java
hasFlowLikeField(serviceClass)
```

它会遍历 service 的非 static 字段，再检查字段类型里是否存在类似 StateFlow 写入实现的结构特征。

代码里的判断核心是：

```java
m.getParameterCount() == 2
&& m.getParameterTypes()[0] == Object.class
&& m.getParameterTypes()[1] == Object.class
&& m.getReturnType() == boolean.class
```

这不是完美的“语义识别”，但它至少建立了第二层条件：

```text
名字 / DexKit 候选命中
        +
字段结构像我要找的 flow
```

而不是“名字存在 = 目标正确”。

## 修改四：DexKit 搜出来的候选也不能无脑接受

DexKit 通过字符串锚点：

```text
NetworkCertificateStatusService
```

找到候选 class 后，我没有直接拿第一个结果，而是也通过同一个 `hasFlowLikeField()` 做结构验证：

```java
Class<?> candidate = cd.getInstance(lpparam.classLoader);
if (hasFlowLikeField(candidate)) {
    service = candidate;
    break;
}
```

这相当于把解析器从单特征：

```text
字符串命中
```

变成：

```text
字符串命中 + 结构命中
```

## 1.2026.237 最终解析到了什么

新版本验证到的 fallback 名称是：

```text
service:   wcy
bad state: tcy
```

而自适应模式下，我不再依赖一个固定 bad-state class，而是把 certificate flow 固定到初始化时的正常值。

成功时日志会类似：

```text
NetCertFix: ready. service=wcy, badState=<pin-to-initial>
NetCertFix: blocked untrusted-network-certificate state write
```

旧版 1.2026.230 对应的是：

```text
NetCertFix: ready. service=i6w, badState=f6w
```

## 如果都找不到，宁可不 Hook

跨版本 Hook 最危险的不是“功能没生效”，而是**误命中完全无关的类还继续执行**。

所以解析失败时模块会直接保持 inactive，并写日志：

```text
NetCertFix: could not resolve NetworkCertificateStatusService; module inactive
```

这就是我更愿意接受的失败模式：

```text
无法证明目标正确
→ 不执行
```

而不是：

```text
有一个类名碰巧存在
→ 赌它还是原来的类
```

## 这次改动到底解决了什么

如果把这个 commit 压成一句话，不是“增加 1.2026.237 类名”。

而是把解析策略从：

```text
硬编码短类名优先
短类名不存在才自适应
```

改成：

```text
稳定字符串语义锚点优先
        ↓
结构验证
        ↓
已验证的版本组合 fallback
        ↓
无法确认则停止
```

这次让我对 R8 混淆适配真正警惕的点也不是“名字会变化”，而是：

> **短名字可能在下一版继续存在，但被复用给完全不同的类。**

所以“ClassLoader 能找到它”本身几乎没有版本兼容性的证明力。