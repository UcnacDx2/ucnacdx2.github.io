---
title: "额度恢复之后：用 app-server 协议把 Codex 的受限线程接回来"
description: "不重启会话、不猜 --last：看门狗怎样读 Goal 状态、判断配额恢复、处理 active writer 和卡住的 turn，以及在不确定时停手。"
pubDate: 2026-10-27
tags:
  - Codex
  - Automation
  - JSON-RPC
  - Windows
draft: false
---

这篇对应实际项目：

<https://github.com/UcnacDx2/codex-quota-resumer>

长任务跑到一半撞上额度上限，是最难受的一种中断：上下文还在，Goal 还在，但线程被标成 `usageLimited`，不会自己恢复。手动重开一个会话，等于把前面所有铺垫丢掉。

我做了一个只走 Codex `app-server` 协议的小看门狗：扫出处于配额限制的线程，等账号额度真的恢复，再用原线程继续。

## 先划三条不做的事

这类工具很容易滑向“模拟用户操作”，所以我先把边界写死：

- 不调用 `--last`。它恢复的是“最近一个会话”，不是“被限制的那个 Goal”。
- 不兑换 reset credit，不改审批或沙箱策略。
- 不依赖插件 hook，也不去点 UI。

它只做一件事：**通过官方 app-server 协议读取状态、恢复线程、启动 turn。**

## app-server 是什么形态

它不是 HTTP，而是 `codex app-server` 进程上的 JSONL（每行一个 JSON-RPC 消息）：

```text
watchdog ──stdin──▶ codex app-server
         ◀─stdout─  responses + notifications
```

所以客户端至少要做三件事：

1. 给每个请求编号，按 `id` 把响应分发给对应的等待者；
2. 单独消费没有 `id` 的 notification（`turn/completed`、`thread/goal/updated` 等）；
3. app-server 退出时，把所有挂起的请求立刻失败掉，而不是等超时。

`cqr.py` 里的 `AppServer` 就是这么拆的：两个读线程分别读 stdout/stderr，`call()` 用 `queue.Queue` 等响应，notification 走另一条队列。

```python
self._seq += 1
request_id = self._seq
self._pending[request_id] = response_queue
...
target = self._pending.get(message.get("id"))
(target.put(message) if target is not None else self.notifications.put(message))
```

`install.ps1` 在 Windows 上装的是同逻辑的 Node 实现 `cqr.mjs`，Python 那份更适合当参考实现读。

## 状态机只有四步

持久化文件 `state.json` 里，每个待恢复线程是一条记录，`stage` 只有四种：

```text
queued    ── 扫描到，等额度
starting  ── 正在 resume + 起 turn
running   ── turn 已启动，等 turn/completed
completed ── 从 blocked 里移除
```

失败不会删除记录，只会退回 `queued` 并写下 `lastError`。这样重启看门狗之后，之前没处理完的线程还在。

## 判断一：什么叫“额度恢复了”

不能看“上次失败了，这次应该好了”。我的判断是三条同时成立：

```python
recovered = (
    bool(windows)
    and not rate_limits.get("rateLimitReachedType")
    and len(numeric) == len(windows)
    and all(value < 100 for value in numeric)
)
```

其中 `windows` 来自 `account/rateLimits/read` 的 `primary` / `secondary`，兼容几种返回形态。少一个窗口、字段类型不对、或者还有 `rateLimitReachedType`，都算没恢复。

宁可多等一个周期，也不要在半恢复状态下把线程再撞一次。

## 判断二：`usageLimited → active` 不是取消

这是整个项目里最容易写错的一处。

Codex 在配额重置后，会把 Goal 状态从 `usageLimited` 改回 `active`，**但那不等于用户取消了任务，也不等于已经恢复执行**。它只是状态回到可用。

所以恢复逻辑里单独处理了这种情况：

```python
auto_reactivated = status == "active"
if not is_usage_status(goal) and not auto_reactivated \
        and stage not in {"starting", "running"} \
        and not allowed_active_recovery:
    cancel()
```

反过来说，如果线程已经被手动取消、Goal 变成终态，就把它从队列里删掉，不再尝试。

## 判断三：Desktop 可能已经接手了

如果 Codex Desktop 自己恢复了线程，看门狗再调 `thread/resume` 会拿到 “active writer” 之类的错误。

这时候不能重试到死，也不该抢回控制权。我的做法是去看 rollout 文件：

```text
~/.codex/sessions/**/*<threadId>*.jsonl
~/.codex/archived_sessions/**/*<threadId>*.jsonl
```

如果最近 10 分钟内这个文件有新写入，说明已经有人在跑，直接判定为“外部已恢复”，把记录清掉：

```python
if activity and activity[1] > observed_at and now_ms - activity[1] < 10 * 60_000:
    add_history(data, "external_recovery_observed", thread_id)
    data["blocked"].pop(thread_id, None)
    return False
```

这不是猜，是拿文件时间戳当证据。

## 判断四：上一轮 turn 是不是卡住了

有些线程在中断时留下一个 `inProgress` 的 turn。直接 `turn/start` 会和它打架，所以恢复前先 `thread/read`：

- 没有活动 turn，且线程不是 active：可以恢复；
- 活动 turn 的 `startedAt` 晚于我们第一次发现它的时间：说明线程还在动，跳过；
- 活动 turn 早于发现时间：判断为陈旧，`turn/interrupt` 之后再启动。

时间戳要兼容秒、毫秒和 ISO 字符串三种形态，这段代码比看起来长。

## 恢复本身只调四个方法

```text
thread/goal/get     确认 Goal 还在、状态是什么
thread/read         看 turn 列表
thread/resume       绑定到原线程
turn/start          用固定 prompt 继续
```

如果 Goal 还是 `usageLimited`，会先 `thread/goal/set` 成 `active`，并记录 `watchdogActivated`，方便之后区分“我改过”和“系统自己变的”。

启动 turn 时带上原线程的 `cwd`，避免恢复出来的任务在错误目录里继续。

提示词是固定的：

> 额度已经恢复。继续此前因额度限制中断的同一任务，从最后一个安全点开始；先核对现状，不要重复已经完成的操作。

## 结果由 notification 决定，不由 HTTP 决定

turn 启动成功只代表“开始了”。真正的结局在 `turn/completed`：

- error 里是 usage limit：退回 `queued`，等下一轮；
- `status == completed` 且没有 error：删除记录；
- 其他：记 `lastError`，同样退回 `queued`。

这样一次失败不会把线程丢掉，也不会无限重试。

## 单实例和失败关闭

Windows 和 Unix 的锁写法不同，但目的相同：只允许一个看门狗。

```python
if os.name == "nt":
    msvcrt.locking(self.stream.fileno(), msvcrt.LK_NBLCK, 1)
else:
    fcntl.flock(self.stream.fileno(), fcntl.LOCK_EX | fcntl.LOCK_NB)
```

拿不到锁就退出，不去写 pid 文件，也不碰状态文件。

计划任务本身也配了 `MultipleInstances IgnoreNew`、崩溃后每分钟重启、电池上照常运行。登录时启动，是因为它需要当前用户的 Codex 登录态。

## 这个项目真正解决的是什么

不是“自动点继续”，而是把一次不确定的中断变成一条有证据的状态迁移：

```text
扫到 usageLimited
  → 记到磁盘
  → 等所有额度窗口都低于 100%
  → 确认没有别的 writer
  → 清掉陈旧 turn
  → resume + turn/start
  → 用 turn/completed 收尾
```

每一步都能在 `watchdog.log.jsonl` 里查到，失败也停在安全的一侧。协议一旦变动，它会报错退出，而不是替你做一个看起来很聪明的决定。
