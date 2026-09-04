---
title: "AI 参与真实开源 PR：以 OpenList 139 驱动为例，我怎么验证它不是在写 Demo"
description: "用一个已经合并的 OpenList PR 拆解 AI 辅助开发：具体改了哪些文件、测试怎么跑、哪些结果由真实账号验证，以及为什么最后责任仍然在人。"
pubDate: 2026-09-04
tags:
  - AI
  - Open Source
  - OpenList
  - Engineering
draft: false
---

我确实会用 AI 写代码，但如果文章只写：

> AI 帮我读代码、生成代码、补测试，我再人工检查。

那几乎没有信息量。

所以这篇直接拿一个真实 PR 做证据：

<https://github.com/OpenListTeam/OpenList/pull/2067>

PR 标题是：

```text
fix(drivers/139): optimize login flow with cookie reuse and robust fallback
```

它已经合并到 OpenList，上游 merge commit 是：

```text
376cada3d4917f9ac9136eb2ca313c8384557dcf
```

我在 PR 里也明确做了 AI Disclosure：使用了 ChatGPT 和 Codex，范围包括代码生成、重构、文档、测试和 review assistance。

重点不是“AI 用得多”，而是：**这些 AI 产物最后怎么被验证。**

## 这个任务不是从 prompt 开始，而是从一个真实失败链路开始

139 云盘驱动的问题不是新建一个函数，而是修改已有鉴权状态机。

它涉及：

```text
Authorization
MailCookies
Username / Password
SSO Cookie
HTTP Redirect
Token Refresh
邮箱风控
```

这类任务里，AI 最容易犯的错误不是语法错，而是生成一套“看起来合理、实际上不符合现有协议状态”的代码。

所以我的第一层约束是：**尽量让改动落在现有结构里，而不是让模型重新设计一套驱动。**

最后 #2067 实际只改了四个文件：

```text
drivers/139/driver.go
drivers/139/meta.go
drivers/139/util.go
drivers/139/util_test.go
```

这个 diff 边界本身就是一种约束。

## AI 可以帮我快速提出方案，但我需要先确定“状态”

原始逻辑在 `driver.go` 里对凭证的判断比较隐式。

改造后，初始化入口收敛成：

```go
if !d.isShare() {
    if err := d.validateAndInitCredentials(); err != nil {
        return err
    }
}
```

并在 `util.go` 里明确区分：

```go
const (
    credentialStateAuthorization credentialState = iota
    credentialStateFullLogin
    credentialStateCookiesOnly
)
```

这类改造 AI 很适合协助做，因为它需要把散落的 if/else 归并成一套明确状态。

但真正需要人确认的是：

```text
哪些状态真的被上游服务支持？
优先级应该是什么？
某个 Cookie 有值是否真的代表可复用？
```

例如 MailCookies fast login 并不是“Cookie 非空就走”，而是要检查：

```text
Os_SSo_Sid
RMKEY
```

两个关键 Cookie 是否同时存在。

这个判断最终必须来自实际协议行为和测试，不是语言模型对 Cookie 名字的猜测。

## 我要求 AI 生成测试，而不是只生成实现

#2067 新增 / 修改了 `util_test.go`，覆盖的不是“函数能调用”这种弱测试，而是几个关键规则：

- credential state 选择；
- Cookie sanitize；
- Cookie merge；
- redirect status 处理。

实际跑的命令是：

```sh
go test ./drivers/139
```

这一步对 AI 辅助开发特别重要。

因为如果只让 AI 改 implementation，很容易形成一个自洽故事：它先假设某个行为正确，再按照这个假设写代码。

测试至少逼迫改动变成可以被机械检查的条件。

## 单元测试通过还不够：我又跑了真实账号 smoke test

这个驱动最关键的问题是第三方真实认证链路，所以本地 test 不能证明全部事情。

我用临时环境变量提供真实凭证，没把真实 Cookie、token、用户名密码放进代码或 PR 描述。

当时有一个非常具体的输入：

```text
现有 MailCookies 缺少 RMKEY
```

预期行为应该是：

```text
不能走 fast login
→ 自动 fallback 到密码登录
```

实际验证结果是：

```text
fast path 被跳过
→ Step 1
→ Step 2
→ Step 3
→ 生成 Authorization
```

这个测试把一个“AI 觉得逻辑合理”的 patch，变成了一个有真实服务端响应支撑的结果。

## 我还专门检查了敏感日志

鉴权代码非常容易在 debug 时顺手输出：

```text
Cookie
Authorization
token
password hash
服务端解密后的响应
```

AI 在排错时也很喜欢建议“多打印一点”。

但在真正准备 PR 时，我做的方向恰好相反：把敏感登录值从日志中移掉。

例如 fallback 后只记录：

```go
log.Debugf(
  "139yun: password fallback generated authorization: %t",
  newAuth != "",
)
```

只留下“有没有生成”的布尔状态，不留下真正凭证。

这也是我认为“AI 能写”和“代码能合并”之间很现实的一道边界。

## PR 描述本身也是验证清单

#2067 的 Testing 部分不是一句：

```text
Tested locally.
```

而是写清：

- 跑了哪个 test command；
- 哪个真实输入缺少 `RMKEY`；
- fast path 为什么被跳过；
- 密码链路实际跑到 Step 1 → 2 → 3；
- 敏感信息没有被提交。

这其实也是我现在越来越喜欢的 AI 使用方式：

> 让 AI 帮我把“我声称做对了”变成一份可检查的 checklist。

## 后续 PR 更能说明为什么不能把 AI patch 当最终答案

后来我继续做了 PR #3029：

<https://github.com/OpenListTeam/OpenList/pull/3029>

它进一步尝试解决：第一次用户名密码登录不应该要求预先抓 MailCookies、短信风控如何续上、刷新后的 Cookie 如何复用等问题。

这版又新增了非常具体的约束：

```go
SetRetryCount(0)
```

用于登录和短信 POST，避免 Resty 自动重试非幂等请求。

同时验证了：

```text
stale MailCookies
→ fast login 失败
→ password fallback
→ 刷新 MailCookies
→ 后续 fast login 可复用
```

但截至我整理这篇文章时，#3029 **仍然是 open，尚未合并**。

所以即使代码已经跑过真实账号回归，我也不能把它描述成“OpenList 当前正式实现”。

这就是另一个很重要的置信度边界：

```text
本地验证成功 ≠ upstream 已接受
```

## 我现在会怎么定义“AI 参与了一个真实工程任务”

至少需要四层证据：

```text
1. Diff
   AI 最后到底改了哪些文件、哪些函数

2. Automated test
   哪条命令能重复跑

3. Real behavior
   真实输入是否跑过真实外部系统

4. Review / upstream state
   已合并、仍 open，还是只是我自己的 branch
```

如果只有第一层，那更像代码生成。

如果四层都能说清，我才愿意把它写成“AI 辅助工程实践”。

AI 在这里最有价值的确实是缩短探索和修改循环；但**置信度不是模型给的，而是 diff、测试、真实运行结果和 upstream 状态共同给的。**