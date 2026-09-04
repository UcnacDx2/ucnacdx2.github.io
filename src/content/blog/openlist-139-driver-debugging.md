---
title: "OpenList 139 云盘鉴权重构：我到底改了哪四个文件"
description: "从一个真实的 Cookie / 登录问题出发，拆开 OpenList 139 驱动的凭证选择、Cookie 复用、密码 fallback、重定向处理和测试证据。"
pubDate: 2026-09-04
tags:
  - OpenList
  - Go
  - HTTP
  - 鉴权
draft: false
---

这篇不再从“Cookie 很复杂”这种泛泛结论开始，而是直接写我在 OpenList 139 云盘驱动里真正改过什么。

证据主要来自我提交并已经合并的 OpenList PR #2067：

<https://github.com/OpenListTeam/OpenList/pull/2067>

这个 PR 最终改了四个文件：

```text
drivers/139/driver.go
drivers/139/meta.go
drivers/139/util.go
drivers/139/util_test.go
```

后面还有一个继续改进凭证续期的 PR #3029，但截至本文整理时它仍然是 **open、未合并**：

<https://github.com/OpenListTeam/OpenList/pull/3029>

所以本文会把“已经进入上游的 #2067”和“后续仍在 review 的 #3029”分开写。

## 起点：为什么复制了一大串浏览器 Cookie 仍然会出问题

当时表面现象很像配置问题：浏览器已经登录，拿到一长串 Cookie，塞进驱动后却仍可能登录失败、反复走密码登录，甚至触发邮箱侧风控。

真正的问题不是“Cookie 字符串不够长”，而是驱动原本没有把不同凭证的优先级和生命周期拆清楚。

在 #2067 之前，`driver.go` 的初始化逻辑大致是：

```go
if Authorization == "" && !isShare() {
    if Username != "" && Password != "" {
        loginWithPassword()
    } else {
        return error
    }
}
```

也就是说，只要 `Authorization` 为空，就很容易直接进入完整密码登录。

这对 139 邮箱这种带风控、Cookie 会续期、还存在 SSO 链路的服务来说并不理想：**能复用已有凭证时却重新登录，既慢，也更容易触发风控。**

## 第一处改动：把“有没有字符串”改成明确的凭证状态机

我在 `driver.go` 里把原来直接判断用户名密码的逻辑收口成：

```go
if !d.isShare() {
    if err := d.validateAndInitCredentials(); err != nil {
        return err
    }
}
```

真正的选择逻辑放进 `util.go`。

当时明确区分了三种状态：

```go
const (
    credentialStateAuthorization credentialState = iota
    credentialStateFullLogin
    credentialStateCookiesOnly
)
```

这一步很重要，因为从此初始化逻辑不再是“Authorization 没有就登录”，而是先回答：**我现在手上到底是哪一种可用凭证？**

优先级变成了：

```text
已有 Authorization
        ↓
可复用的 MailCookies fast path
        ↓
用户名 + 密码完整登录 fallback
```

如果用户名和密码只填了一半，则直接给明确配置错误，而不是让后面的 HTTP 请求以奇怪方式失败。

## 第二处改动：MailCookies 不是“有值就能 fast login”

139 邮箱 Cookie 里会混着大量页面状态、统计字段和临时值。

在 #2067 里，我把 fast login 的有效条件收紧成：必须同时存在两个关键 Cookie：

```text
Os_SSo_Sid
RMKEY
```

如果只是从浏览器复制了一大串 Cookie，但缺少其中一个，驱动不会再假装这是一份完整 fast-login 凭证。

这也解释了我当时遇到的一个现象：**Cookie 很长并不代表登录态完整。**

我做真实账号 smoke test 时，输入 Cookie 就缺少 `RMKEY`。结果是 fast path 被正确跳过，随后完整密码链路继续执行并成功。

## 第三处改动：清理 Cookie，不把旧 JSESSIONID 带进新登录

`util.go` 里新增了 Cookie 解析、清理和合并逻辑。

原因之一是 `JSESSIONID`。

旧的 `JSESSIONID` 如果继续带到一次新的密码登录中，可能让 `mail.10086.cn` 把这个请求判成异常会话，进一步进入风控。

所以密码登录前会处理 Cookie：

```go
func sanitizeMailLoginCookies(existing, newJSessionID string) string
```

核心原则是：

- 页面统计类 Cookie 不参与登录；
- `Os_SSo_Sid` / `RMKEY` 等不应该被当作普通原始上下文胡乱拼接；
- 如果拿不到新的 `JSESSIONID`，旧值宁可丢掉，也不继续发送。

这不是为了“Cookie 越少越好”，而是为了让请求携带的状态和当前登录阶段一致。

## 第四处改动：不再通过 Resty 的错误字符串判断 302

密码登录流程里另一个很脆弱的地方是重定向。

以前代码会依赖 Resty 在禁止重定向时返回的错误文本，再通过字符串匹配猜测是否遇到 302。

这种写法的问题很直接：

```text
HTTP 语义稳定
错误文案不稳定
```

所以 #2067 把这部分改成直接读取 HTTP response 的状态码，按 `3xx` 处理，而不是匹配类似 `auto redirect is disabled` 这样的错误字符串。

这是一个看起来很小、但明显降低维护成本的改动：**协议层判断就用协议层的数据，不用库的报错文案当 API。**

## 第五处改动：Authorization 失效后也要有明确 fallback

`refreshToken()` 不只处理“token 快过期”，还会遇到：

- Base64 解码失败；
- Authorization 结构不完整；
- expiration 非法；
- token 已过期；
- refresh API 返回失败。

这些情况现在统一转到：

```go
loginAfterAuthorizationFailure(...)
```

再进入密码 fallback。

而不是在不同分支里各自返回一个错误，让用户手工重新配一遍驱动。

## 第六处改动：把敏感值从日志里拿掉

原实现里有一些调试日志会打印新生成的 Authorization 或登录链路中的中间值。

#2067 同时清理了这些日志，避免输出：

- password / password hash；
- Cookie；
- passId / token / dycpwd / authToken；
- 解密后的认证响应正文。

保留的日志更偏“状态”，例如：

```go
log.Debugf("139yun: password fallback generated authorization: %t", newAuth != "")
```

只记录“有没有成功生成”，而不是把真正的凭证打出来。

## `meta.go` 改的是什么

代码改了，配置 UI 的说明也必须跟着改。

`drivers/139/meta.go` 里对这些字段的 help text 做了同步：

```text
Authorization
Username
Password
SmsCode
MailCookies
```

目的不是换文案，而是让 UI 准确表达哪些组合被支持。

否则后端已经支持多条凭证路径，用户界面却仍暗示“每个字段都必填”，最终还是会把人带到错误配置上。

## 我怎么验证 #2067 不是“逻辑看起来对”

PR 里至少有两层验证。

### 1. 驱动单元测试

实际执行：

```sh
go test ./drivers/139
```

测试覆盖了：

- credential state 选择；
- Cookie 清理；
- Cookie merge；
- HTTP redirect 状态处理。

### 2. 真实账号 smoke test

我用临时环境变量提供真实凭证，没有把凭证写进代码或 PR 描述。

实际验证到：

```text
输入 MailCookies 缺少 RMKEY
        ↓
fast path 正确跳过
        ↓
进入完整密码登录
        ↓
Step 1 → Step 2 → Step 3
        ↓
成功得到 Authorization
```

所以这个 PR 的证据不是“理论上应该能登录”，而是有真实链路跑通。

## 后续 #3029：继续解决“第一次登录为什么还要先准备 Cookie”

#2067 合并后，我继续往下看，发现还可以再减少一个人为前置条件：**用户名 + 密码第一次登录，理论上不应该要求用户先去浏览器抓一份 MailCookies。**

这就是后续 PR #3029 做的事情。

注意：下面这些截至本文整理时 **还没有合并上游**。

它进一步把 `mail_cookies` 变成账号密码登录时的可选值，并做了几件具体事：

1. 密码登录响应里的新 Cookie merge 回驱动状态并持久化；
2. 后续登录复用刷新后的 MailCookies；
3. 遇到支持的短信风控时，保存临时 Cookie，填写 `sms_code` 后继续链路；
4. 登录和短信 POST 显式设置：

```go
SetRetryCount(0)
```

避免 Resty 自动重试这种非幂等请求，导致重复登录或重复发验证码；
5. 密码登录停在第一个 `302`，直接从响应中提取 `sid`，不继续跟随跳转。

这一版的回归测试实际跑的是：

```sh
go test ./drivers/139 -count=1
```

真实账号回归还验证了：

```text
旧 MailCookies fast login 失败
        ↓
password fallback 成功
        ↓
刷新 MailCookies
        ↓
下一次可直接走 Step 2 → Step 3 fast login
```

以及零初始 Cookie 的短信风控 bootstrap。

但再次强调：**#3029 仍在 review，不能把它写成“OpenList 当前正式版已经如此”。**

## 这次真正学到的不是“Cookie 很复杂”

如果只把这次经历总结成“Web Cookie 很复杂”，信息量太低。

我真正改掉的是几个具体的工程问题：

```text
隐式 if/else 凭证判断
→ 明确 credential state

Cookie 有值就尝试复用
→ 验证 Os_SSo_Sid + RMKEY

旧 JSESSIONID 继续发送
→ 登录前清理会话上下文

匹配 Resty 错误字符串识别重定向
→ 直接看 HTTP 3xx

失败后让用户重新配置
→ Authorization refresh 失败自动 fallback

调试日志打印敏感认证值
→ 只记录状态，不记录凭证
```

这才是我觉得这次 139 驱动改造真正值得记录的部分。