---
title: "验证码是道算术题：用 LLM 跑通校园签到，以及几个必须停手的边界"
description: "登录重试、图片验证码识别、token 藏在响应头、签到要连续调四个接口，以及哪些失败我故意不让它重试。"
pubDate: 2026-10-27
tags:
  - Go
  - LLM
  - Automation
  - HTTP
draft: false
---

这篇对应实际项目：

<https://github.com/UcnacDx2/wisestu-auto-sign>

实习期间每天要在学校系统里签到，时间点固定，偶尔会忘。系统登录要过一道图片验证码，内容是道算术题。

这个工具只做一件事：到点登录、找到签到任务、提交、再用查询接口确认真的成功。

> 它只用于自己的账号。批量签到、替别人签到都不在目标里，也没做任何绕过风控的设计。

## 整体流程

```text
检查登录态
  ├─ token 还有效 ──▶ 直接签到
  └─ 失效 ──▶ 登录循环
                ├─ 取验证码图片
                ├─ LLM 识别算式
                ├─ 提交账号 + 验证码
                └─ 从响应头取 token
                        ↓
              查询未签到列表
                        ↓
              进入签到 → 提交位置 → 查询结果
```

代码不长，真正花时间的是把每一步的“成功”定义清楚。

## 验证码：为什么用多模态模型，而不是 OCR

这类验证码不是扭曲字母，而是一张带算式的图片：`7 + 5 =`。传统 OCR 能读出字符，但你还要自己解析表达式、处理全角符号、判断有没有认错。

多模态模型刚好适合这种“识别 + 计算”合并的任务。提示词里把输出格式定死：

```text
识别下图中的数学算式并计算出结果，输出 JSON：
{"expression": "...", "result": 8, "error": null}
```

请求参数只开很小的自由度：

```go
"max_tokens":  100,
"temperature": 0.1,
"stream":      false,
```

响应里再解析 `choices[0].message.content` 这段 JSON。解析失败就当识别失败，进入下一次重试——**不做“猜一个数字”的兜底**，因为验证码错一次就是一次失败登录。

## 登录循环：重试，但要知道什么时候不该重试

登录最多 5 次，每次间隔可配。循环里任何一步失败都 `continue`，直到拿到 token：

```go
for i := 0; i < 5; i++ {
    verif := getVerification()      // 取图
    answer := llm.SolveCaptcha(...) // 识别
    resp := login(verif, answer)    // 提交
    ...
}
```

但有一个错误码必须立刻退出：

```go
if baseResp.Code == 1002 {
    return "", fmt.Errorf("登录失败：密码错误 (code: 1002)")
}
```

密码错了，重试 5 次只会让账号更危险。**能被重试解决的（验证码错、网络抖动）和不能被重试解决的（密码错），必须分开。**

还有一种容易漏的情况：接口返回 `code == 0` 表示登录成功，但 token 不在 body 里，而在响应头：

```go
token := resp.Header().Get("token")
if token == "" {
    // 仍然算失败，继续下一轮
}
```

之后的请求统一把 token 放进 `Authorization` 头，不做 cookie 会话。

## 签到不是一次请求

很多自动化脚本在这里翻车：调一个“签到接口”，看到 HTTP 200 就认为成功。这个系统实际要按顺序走四步：

| 步骤 | action | 作用 |
| --- | --- | --- |
| 1 | `getUnSigninList` | 拿到未签到任务 |
| 2 | `getSigninDetails` | 进入签到页，建立本次签到上下文 |
| 3 | `updateLocationSignin` | 提交位置与范围标记 |
| 4 | `getSigninSuccess` | 查询签到结果 |

少一步，前面都白做。所以代码里是显式串行的，每一步都检查业务 `code`：

```go
if err := s.getSigninDetails(signinID, batchNo); err != nil { ... }
if err := s.updateLocationSignin(signinID, batchNo); err != nil { ... }
if err := s.getSigninSuccess(signinID, batchNo); err != nil { ... }
```

选择任务时优先找类型为“实习”的那条；找不到才退回列表第一个，并在日志里明确写出来。这种“兜底”必须留痕，否则哪天签错任务都不知道。

## 位置字段不是一个经纬度

`updateLocationSignin` 要的 `signin_location` 是一个 JSON 字符串，里面既有坐标，也有结构化的地址：

```json
{
  "point": { "lng": 0, "lat": 0 },
  "address": "...",
  "addressComponents": {
    "province": "...",
    "city": "...",
    "district": "...",
    "street": "...",
    "streetNumber": "..."
  }
}
```

外勤标记 `outsideFlag` 也是接口要求的固定值。经纬度来自配置，但地址组件最初是写死在代码里的——**这是这个项目做得最差的一处**：换一个实习点就得改代码重编译，还把真实地点永久留在了提交历史里。

后来改成整段地址都进配置，并把 `address` 当必填项校验：没填就直接报错退出，而不是把空地址悄悄发给服务端。顺带把日志文件移出了版本库。

## 调度：跑一次还是常驻

命令行分两个子命令：

```text
zhxg-signin run     执行一次
zhxg-signin daemon  按 cron 常驻
```

常驻模式用 `robfig/cron`，并且显式加载时区：

```go
loc, _ := time.LoadLocation(cfg.Scheduler.Timezone)
c := cron.New(cron.WithLocation(loc))
```

不指定时区的话，容器或 CI 里跑出来就是 UTC，签到时间会整体偏 8 小时——这种 bug 很安静，日志里一切正常。

## 我故意没做的

- **没有并发签到。** 一个账号、一个任务，串行最稳。
- **没有把验证码识别结果缓存复用。** 每次登录都要新图，旧答案没有意义。
- **没有无限重试。** 5 次之后退出，让调度器下个时间点再来。
- **没有绕过任何设备校验。** 请求里带客户端标识是为了让接口接受请求，不是为了伪造设备。

## 这类工具真正难的地方

不是“调通接口”，而是定义失败：

```text
验证码识别失败   → 重试
密码错误         → 立即停
token 缺失       → 重试
签到接口 code≠0  → 停，记录原因
查询结果失败     → 停，不要谎报成功
```

把这五种情况分清楚，比多写几个功能重要。否则它会在某天悄悄失败，而你以为它一直在工作。
