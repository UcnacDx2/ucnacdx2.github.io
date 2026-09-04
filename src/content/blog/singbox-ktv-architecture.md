---
title: "SingBox KTV 从 Demo 到 0.9.12：我把状态、解析、播放和 Android 系统边界拆到了哪里"
description: "用真实目录、运行拓扑和修复 commit 复盘一个 Android TV KTV 系统：唯一状态源、DLNA/WebUI/Media3、Resolver 边界，以及 Android 12 开机启动和逐字字幕的具体坑。"
pubDate: 2026-09-04
tags:
  - Android TV
  - TypeScript
  - Kotlin
  - 架构
draft: false
---

项目仓库：

<https://github.com/UcnacDx2/singbox-ktv>

截至这篇整理时，仓库已经准备到 `0.9.12`。

这篇不再写“电视负责播放、手机负责交互”这种抽象总结，而是直接看现在代码到底被拆成了什么。

## 当前运行拓扑长什么样

项目的 `docs/ARCHITECTURE.md` 里已经把运行拓扑写得很明确：

```text
手机 DLNA 客户端 ── SSDP/SOAP ─┐
                              ├── Android TV KTV Runtime ── Media3 ── HDMI
手机浏览器 ── HTTP/SSE WebUI ──┘          │
                                          ├── 本地热点（可选）
                                          ├── APK 本地解析器
                                          ├── Resolver Gateway（失败回退）
                                          └── WSS ── Cloudflare Worker ── 异地 WebUI
```

这里最关键的不是协议多，而是：**电视端只有一个权威状态源 `KtvRuntime`。**

WebUI、React Native TV UI、DLNA SOAP、Media3 callback 最后都读写同一份队列状态。

我不希望出现这种情况：

```text
DLNA 客户端认为正在播 A
WebUI 队列认为正在播 B
Media3 实际又是 C
```

所以“统一状态源”是比 UI 怎么画更早确定的设计。

## TypeScript 和 Kotlin 是怎么分工的

现在仓库结构不是一个大 Android 工程，而是：

```text
apps/
  tv/                 React Native TV / TypeScript
  web-lite/           手机 WebUI
  mock-tv-server/     Node.js Host / 协议回测
  resolver-worker/    Cloudflare Worker / 远程控制

android-shell/        Kotlin Android TV runtime

packages/
  shared/             AppState / QueueItem / 字幕 cue / capability
  core/               队列、DLNA、链接分类、字幕转换

test/                 Node E2E + Android 静态契约测试
```

Kotlin 只承担 Android 平台必须承担的东西，例如：

```text
KtvRuntime
EmbeddedKtvServer
DlnaSsdpServer
HotspotController
KtvForegroundService
Media3
```

而队列数据结构、协议业务逻辑、WebUI、Resolver 这类高变化部分尽量留在 TypeScript。

这不是为了追求“全栈统一语言”，而是因为 Android 的热点、组播、前台服务、Media3 这些边界没必要硬绕开原生 API。

## 为什么先做 `mock-tv-server`

桌面 Host 可以直接这样跑：

```sh
npm install
npm test
HOST=0.0.0.0 PORT=8090 npm start
```

然后：

```text
http://机器IP:8090/tv
http://机器IP:8090/control
```

它的意义不是“再写一个服务端”。

而是让我可以在没有 Android TV 真机的情况下，先验证：

- 队列 API；
- WebUI；
- Resolver；
- 字幕转换；
- SSE 状态同步；
- 控制协议。

换句话说，Host 是一个协议回测器。

如果这些都被焊进 APK，任何一次协议问题都得重新构建、安装、拿遥控器操作，反馈循环会慢很多。

## Resolver 为什么不能直接和播放器绑死

APK 现在有本地解析：

```text
Bilibili API
YouTube / NewPipeExtractor
网易云接口
```

失败时还可以回退 Gateway：

```text
Android APK
  └→ HTTPS / LAN Resolver Gateway
       ├→ yt-dlp
       └→ 自托管兼容 API
```

但无论来源是什么，最后都要归一成类似：

```json
{
  "title": "歌曲名",
  "artist": "歌手",
  "mediaUrl": "https://...",
  "mediaKind": "video",
  "artworkUrl": "https://...",
  "subtitle": {"format": "karaoke", "cues": []},
  "expiresAt": "..."
}
```

播放器只理解标准化的 `QueueItem`，不用知道这首歌到底来自 Bilibili、YouTube 还是网易云。

这条边界后来变得很重要，因为解析源的变化速度明显高于 Media3 播放层。

## DLNA 我没有实现成“完整认证栈”

当前目标是最小可互通 MediaRenderer。

已经实现的范围包括：

```text
SSDP
AVTransport
RenderingControl
ConnectionManager
GENA subscribe / renew / unsubscribe / NOTIFY
```

但项目文档明确承认：不是完整 DLNA 认证实现。

例如不同厂商客户端如果要求更完整的 DIDL-Lite metadata 或 LastChange 字段，仍要真机抓包再补。

这也是我现在比较看重的一种工程写法：**把支持边界写出来，而不是把“能和某几个客户端互通”描述成完整协议实现。**

## 一个真实 Android 坑：开机自启把整个进程搞崩了

项目加 `BOOT_COMPLETED` 后，最初 `KtvBootReceiver` 会直接启动前台服务。

核心逻辑类似：

```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    context.startForegroundService(serviceIntent)
} else {
    context.startService(serviceIntent)
}
```

在 Android 12+，后台广播直接启动 foreground service 可能抛：

```text
ForegroundServiceStartNotAllowedException
```

结果不是“服务没启动”，而是 receiver 抛异常，整个启动流程被打断。

对应修复 commit：

<https://github.com/UcnacDx2/singbox-ktv/commit/322aa9cf827c611ad62b5e4dc21b5b9123ab7d88>

我把 service start 包进 `try/catch`：

```kotlin
try {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        context.startForegroundService(serviceIntent)
    } else {
        context.startService(serviceIntent)
    }
} catch (error: Throwable) {
    Log.w(
      "KtvBootReceiver",
      "启动前台服务被拒绝，改为直接拉起界面",
      error,
    )
}
```

然后仍然启动 `MainActivity`。

前台 Activity起来之后，再按正常允许的路径启动服务。

我还同步给 `test/android-scaffold.test.mjs` 加了静态契约检查，确认 BootReceiver 里不仅有 service start，也有异常 fallback 和 `startActivity`。

这就是一个很具体的“为什么 Android 系统边界留在 Kotlin”案例：这种生命周期限制如果只从 Web / Node 层想象，很难设计对。

## 另一个真实坑：逐字歌词“同步正确”但肉眼很难看

KTV 字幕不是普通 LRC 一行一行切。

网易云 YRC / karaoke cue 做逐字描色后，遇到 seek、position resync 或快节奏歌词时，早期实现会直接：

```text
timeline = 当前真实 position
```

结果就是前几个字突然整段跳色。

对应修复 commit：

<https://github.com/UcnacDx2/singbox-ktv/commit/98424d4eed20028ddda473d2825e84a5acabbe60>

现在先计算：

```ts
const catchUpMs = Math.abs(positionMs - timelineMs.value);
```

如果偏差超过 40ms，不瞬移，而是用一个 90～320ms 的短动画追到真实进度：

```ts
timelineMs.value = withTiming(positionMs, {
  duration: Math.min(320, Math.max(90, catchUpMs)),
  easing: ReanimatedEasing.linear,
});
```

追上以后再继续按 cue 的 `endMs` 正常推进。

同一个 commit 还做了两件事：

- 去掉 `active` 门控，让待唱阶段的时间轴也连续推进；
- 双行歌词固定行位，新 cue 原位淡入，不再上下跳。

这个坑对我挺有意思：

> 时间戳“数学上更准确”，不等于动画“视觉上更正确”。

## 为什么我现在不把“架构”理解成画模块图

这个项目真正让我觉得架构有用的地方，是它不断帮我限定改动范围。

例如：

```text
来源解析出问题
→ Resolver / Gateway

队列状态错乱
→ KtvRuntime / core state machine

电视播放行为
→ Media3 / Kotlin runtime

遥控器焦点
→ apps/tv

异地控制
→ Worker relay
```

而不是每次功能变化都同时动 APK、WebUI、Node Host 和协议层。

所以如果要用一句话总结这个项目，我现在会写：

> **不是“我做了一个 Android TV 点歌 App”，而是我把一个天然跨设备的 KTV 场景拆成一个权威状态源、多个协议入口和几个明确的平台边界，并且这些边界经受了真实 Android 生命周期和字幕同步问题的修改。**