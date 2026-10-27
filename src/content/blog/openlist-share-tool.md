---
title: "把 Android 分享菜单接到 OpenList：重名、分片和断网后该问谁"
description: "从系统分享面板上传到 OpenList 并复制 /d/ 直链。真正麻烦的不是上传，而是同名文件、服务端的分片规则，以及网络抖动时先查状态再重试。"
pubDate: 2026-10-27
tags:
  - Android
  - Java
  - OpenList
  - Upload
draft: false
---

这篇对应实际项目：

<https://github.com/UcnacDx2/openlist-share-tool>

目标很小：在任意 App 里点“分享”，选这个工具，文件传到 OpenList 的指定目录，完成后通知栏能直接复制直链。

它不是一个网盘客户端，没有文件列表，也不做浏览。**只有上传和取链。**

## 为什么不用“上传完再分享”

Android 的分享面板给的是 `content://` URI，不是文件路径。很多应用会把这个 URI 直接交给 `HttpURLConnection`，然后在某些 Provider 上失败，或者对大文件一次性读进内存。

这里的做法是：

```text
ShareReceiverActivity 接收 URI
        ↓
启动 UploadService（前台服务）
        ↓
流式读取 + 分片上传
        ↓
通知栏显示进度 / 完成后复制直链
```

所有上传都在前台服务里跑，退出应用不会中断，通知栏能看到当前文件和整体进度。

## 重名：`Overwrite: false` 不是“保留两个文件”

OpenList 的上传接口在 `overwrite=false` 时的行为是：目标已存在就拒绝上传。它不会自动改名，也不会保留两份。

所以“重名”必须在客户端解决。流程是：

```java
String target = joinPath(dir, originalName);
synchronized (reservedTargets) {
    if (reservedTargets.contains(target)) {
        target = localUniqueTarget(...);   // 同一批里的重名
    }
    reservedTargets.add(target);
}
if (!overwrite && exists(base, token, target)) {
    target = timestampTarget(...);          // 服务端已存在的重名
}
```

两层判断缺一不可：

- `reservedTargets` 是内存里的并发集合，解决“同一次分享里选了两个同名文件”；
- `exists()` 查服务端，解决“目录里已经有同名文件”。

改名策略是加人类可读的时间戳，而不是 `file(1).txt`：

```text
report.pdf
report (2026-10-27 14-03-21).pdf
report (2026-10-27 14-03-21) #2.pdf
```

同一秒内再次冲突时追加序号，避免两个并发线程生成同一个名字。

## 大文件：以服务端返回的分片规则为准

超过阈值（默认 100 MB）走 multipart，小于阈值走单次 PUT。分片参数可以配，代码里的默认值是：

```text
chunk size      10 MiB
chunk 并发      3
文件并发        2
```

但真正开始上传前，先看初始化接口返回了什么：

```java
long chunkSize = session.optLong("chunk_size", requestedChunkSize);
int totalChunks = session.optInt("total_chunks", 0);
Set<Integer> received = parseReceived(session);
```

服务端可以改 `chunk_size`，也可以告诉你哪些分片已经收到了。客户端不能假设“我按 10 MiB 切就一定对”。

`received` 是一组区间，解析后跳过已收分片：

```java
for (int i = 0; i < totalChunks; i++) {
    if (!received.contains(i)) missing.add(i);
}
if (missing.isEmpty()) {
    completeAndWait(...);   // 上次其实已经传完，只差收尾
    return;
}
```

这让“传到一半被杀进程”变成可恢复的：重新进来时，只补缺的分片。

## 断网：先问状态，再决定重试

这是整个项目里最重要的一个设计。

分片请求抛出 `IOException` 时，不能直接重试——**请求可能已经到达服务端，只是响应没回来**。盲目重传会浪费流量，也可能把会话搞乱。

所以先查：

```text
GET /api/fs/multipart/status?upload_id=...
```

然后分三种处理：

| 状态 | 处理 |
| --- | --- |
| 404 或 `completed` | 上传已经结束，直接收尾 |
| `failed` / `aborted` | 抛错，不再重试 |
| 其他 | 退避后重试当前分片 |

```java
StatusResult status = probeStatus(base, token, uploadId);
if (status.httpCode == 404 || isCompleted(status.data)) {
    completedEarly.set(true);
    return;
}
if (isFailed(status.data)) {
    throw new IOException("OpenList 分片上传失败");
}
```

传输层错误和 HTTP 业务错误也被分开了：只有 `IOException` 才算网络问题，4xx/5xx 响应走另一条判定。把两者混在一起，就会出现“服务端明确拒绝，但客户端还在重试”的死循环。

## 直链：用 OpenList 的 `/d/`，不用上游 `raw_url`

上传完成后调 `/api/fs/get` 拿文件信息，但复制给用户的链接是自己拼的：

```java
String sign = data.optString("sign", "");
String link = base + "/d" + Uri.encode(target, "/");
if (!sign.isEmpty()) {
    link += "?sign=" + Uri.encode(sign);
}
```

原因很实际：上游网盘的 `raw_url` 通常带时效，过期后链接就废了。OpenList 的 `/d/...` 入口由服务端决定是 302 还是代理，生命周期由自己的配置控制。

签名是否出现、有效期多长，取决于服务端设置（例如 `sign_all` 和 `link_expiration`）。客户端只负责原样带上，不猜。

## 进度与通知

前台服务的通知不能每条进度都更新一次，否则系统会限流。这里的做法是：

- 维护一个批次的进度对象；
- 按固定间隔发布（`startProgressScheduler`），而不是每个分片都发；
- 状态写入 SharedPreferences，进程被杀后仍能看到最近一次直链。

通知里只有一个动作：复制直链。没有做“取消上传”——中途终止会留下一个未完成的分片会话，而清理只能等服务端过期；与其给一个语义不完整的按钮，不如不做。

## 边界

- **Token 存在 SharedPreferences 里。** 这是本地工具的取舍，不是安全方案；真要更严格，应该用 Keystore。
- **只支持单账号。** 多账号会把目录、Token、历史记录全部搞复杂，暂时不做。
- **分片依赖存储驱动。** 有些驱动不支持 multipart，初始化会明确返回不可用，这时只能退回单次上传或换目录。
- **不做断点续传的跨进程持久化。** 进程重启后靠服务端的 `received` 恢复，而不是本地记录分片状态——服务端才是事实来源。

## 这个项目真正解决的是什么

看起来是“分享一个文件”，实际要处理的是三件容易被忽略的事：

```text
同名冲突  → 客户端先占位，再查服务端
分片规则  → 以服务端返回为准，不假设
网络失败  → 先查状态，再决定是否重试
```

把这三件事做对，上传才不是“大部分时候能用”。
