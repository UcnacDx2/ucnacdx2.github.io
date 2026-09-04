---
title: "从单商户到多租户：一次上门服务项目的真实架构演进"
description: "把设计稿和实际代码分开看：加盟商为什么不能只是角色、Partner/OperatingArea/Offering 怎么落表，以及后来 Tenant、套餐、额度、审计真正实现到了哪一步。"
pubDate: 2026-09-04
tags:
  - 项目复盘
  - SaaS
  - NestJS
  - Prisma
  - 多租户
draft: false
---

> 本文已对公司、人物、业务名称和本地环境信息脱敏。文中只保留可以从项目代码、提交记录或测试中确认的工程事实。

这个项目最开始是一个单商户上门服务系统：用户端、师傅端、运营后台，加上一套 NestJS + Prisma + MySQL 后端。

真正困难的阶段，是业务从“一个商户自己经营”变成“总部 + 加盟商 + 区域 + 门店 + 师傅”，后来又继续要求做成可以商业化售卖的多租户 SaaS。

我以前总结这段经历时，容易写成一句：

> 多租户不是加一个 `tenantId`。

这句话没错，但太空。

这篇我直接按两类证据来写：

```text
设计阶段：当时明确决定系统应该怎么拆
实现阶段：后来代码里到底真正落了哪些模型、Service 和测试
```

两者不混在一起。

## 1. 第一次真正的转折：加盟商不能只是一个后台角色

早期系统天然接近：

```text
用户 → 订单 → 师傅
```

因为只有一个经营主体，所以很多代码根本不需要回答：

```text
这张订单属于谁？
这个师傅是谁的人？
这个价格是谁定的？
这个投诉最后算到谁头上？
```

加盟模式出现后，我一开始最容易想到的做法其实很简单：

```text
给 Admin 加一个 franchisee 角色
```

后来在架构重构阶段，我明确把这个方向否掉了。

设计稿里直接写了几条禁止项：

```text
不要把加盟商简单做成 Admin.role = franchisee
不要让订单归属某个管理员账号，而要归属 partnerId
不要让师傅归属创建人，而要归属 partnerId
不要把订单、履约工单、支付、结算全部塞进一个 Order
```

原因是：

> **账号是“谁在操作”，加盟商是“谁在经营”。**

这两个概念不是一回事。

后来 Prisma 里也确实落成了独立 `Partner` 实体，而不是一个角色枚举。

简化后大概是：

```prisma
model Partner {
  id       String
  tenantId String? @unique

  admins             Admin[]
  partnerAdmins      PartnerAdmin[]
  settlementAccounts PartnerSettlementAccount[]
  operatingAreas     PartnerOperatingArea[]
  serviceOfferings   PartnerServiceOffering[]
  orders             Order[]
  workers            Worker[]
  serviceJobs        ServiceJob[]
  settlements        Settlement[]
  settlementItems    SettlementItem[]
  commissionRules    CommissionRule[]
  complaints         Complaint[]
  stores             Store[]
}
```

这就是第一个比较重要的落地：

```text
经营主体 → Partner
操作身份 → Admin / PartnerAdmin
```

而不是把两者揉成一个 `role`。

## 2. 权限也从“角色”变成了“角色 + 数据范围”

只做到 `Partner` 还不够。

假设两个加盟商都有一个“管理员”，两个人角色完全一样：

```text
role = partner_admin
```

但显然 A 不能看到 B 的订单。

所以项目后来给加盟商管理员关系加了数据范围：

```prisma
model PartnerAdmin {
  partnerId String
  adminId   String
  dataScope DataScope @default(partner_only)
}
```

在设计阶段还明确区分过：

```text
platform_all
partner_only
area_limited
```

这让我后来对 RBAC 有一个很具体的认识：

> **“你是什么角色”只解决能不能执行某种动作；“你能操作哪些数据”是另一个维度。**

这两件事如果不分开，做多租户后台时很容易到处写 `if (admin.partnerId === ...)`。

## 3. “服务区域”后来也没有继续只做成一个字符串

上门服务有一个很现实的问题：用户地址只是一个位置，但业务需要知道这个位置由谁经营。

最终代码里把区域单独建模成了 `OperatingArea`：

```prisma
model OperatingArea {
  id           String
  name         String
  cityCode     String
  districtCode String?
  boundary     Json?
  status       OperatingAreaStatus
}
```

加盟商和区域之间再通过关系表连接：

```prisma
model PartnerOperatingArea {
  partnerId       String
  operatingAreaId String
  active          Boolean

  @@unique([partnerId, operatingAreaId])
}
```

这样“某个城市 / 区域现在由谁负责”就不再是散落在订单代码里的判断。

它成为了一个可以查询、停用、审计和以后扩展边界多边形的业务关系。

## 4. 价格也不能只挂在总部服务项目上

单商户时期，一项洗车服务有一个价格基本就够了。

加盟以后就会出现：

```text
总部定义服务模板
        ↓
加盟商决定自己在哪个区域卖
        ↓
这个区域实际卖多少钱
```

最后真正落表的是 `PartnerServiceOffering`：

```prisma
model PartnerServiceOffering {
  partnerId       String
  serviceId       String
  operatingAreaId String
  price           Decimal
  enabled         Boolean
  capacityPerDay  Int?

  @@unique([partnerId, operatingAreaId, serviceId])
}
```

这比在 `CarService` 上继续加各种“城市价格”“加盟价”字段干净很多。

它表达的是一个完整业务事实：

> 某加盟商，在某经营区域，启用了某项服务，并以这个价格销售。

## 5. 为什么我后来强调订单要保存“当时的事实”

项目里 `OrderItem` 最后没有只关联商品 ID，还保留了金额和快照：

```prisma
model OrderItem {
  originalAmount Decimal
  discountAmount Decimal
  payableAmount  Decimal
  snapshot       Json?
}
```

这是我后来很认可的一点。

如果订单查询时永远去读“当前服务价格”，那总部或加盟商一改价，历史订单就无法解释。

订单需要回答的不是：

```text
这个服务现在多少钱？
```

而是：

```text
用户下单那一刻，看到的是什么商品、什么价格、什么优惠？
```

所以价格快照、订单项快照这类看起来有点重复的数据，其实是为了保存历史事实。

## 6. Order、ServiceJob、Payment、Settlement 为什么要拆

这一部分我要区分“设计”和“实际完成度”。

在 DDD 重构阶段，我明确规划了：

```text
Order       = 交易订单
ServiceJob  = 履约工单
Payment     = 支付事实
Settlement  = 结算事实
```

理由很简单：一笔订单完成以后，可能还有完全不同生命周期的事情：

```text
支付成功
→ 派师傅
→ 上门履约
→ 完成
→ 退款 / 售后
→ 生成结算明细
→ 后续打款
```

如果所有东西都塞进 `Order.status`，状态机会越来越难解释。

当前 Prisma schema 里，`Partner` 已经分别关联：

```text
orders
serviceJobs
settlements
settlementItems
commissionRules
```

也有单独的 `PartnerSettlementAccount`。

**但这里不能写成“自动分账、提现已经完整上线”。**

当时的架构规划里，自动打款、复杂分账、加盟商提现本来就被列在“可暂缓”范围。

所以更准确的表述是：

> 数据模型已经为支付、履约、结算拆开边界，并预留 / 实现了部分结算实体；我不能据此声称完整自动提现链路已经完成。

这个边界就是以前那篇文章缺失的置信度信息。

## 7. 后来做 SaaS 时，我才真正给 Partner 上面再加了一层 Tenant

“加盟商”解决的是经营主体问题，但商业化 SaaS 还要处理另一组事情：

```text
套餐
订阅
试用期
成员
白标品牌
自定义域名
资源额度
账单
审计
```

这些不应该继续全部塞进 `Partner`。

后来一次真正的 SaaS 升级提交里，Prisma 新增了：

```text
SaasPlan
Tenant
TenantSubscription
TenantMember
TenantDomain
TenantUsage
TenantInvoice
SaasAuditLog
```

同时 `Partner` 才增加：

```prisma
tenantId String? @unique
```

也就是说，最终关系更像：

```text
Tenant：SaaS 合同 / 套餐 / 品牌 / 成员 / 额度边界
   ↓
Partner：实际经营主体
   ↓
OperatingArea / Store / Worker / Order / ServiceJob
```

这比“所有业务表直接加 tenantId”更符合当时实际需求。

## 8. `provisionTenant()` 里真正做了什么

SaaS 模块不是只加了几张表。

后端 `saas.service.ts` 里有一个比较典型的租户开通事务：

```ts
provisionTenant(input, actorAdminId)
```

它会在同一个事务里做这些检查和写入：

```text
1. 按 planCode 找套餐
2. 套餐不存在 / 已下架 → 拒绝
3. 如果绑定 Partner：
   - Partner 不存在 → 拒绝
   - Partner 已经绑定 Tenant → 拒绝
4. 创建 Tenant
5. 创建 Subscription
6. 创建 owner 成员
7. 把 Partner.tenantId 指向新 Tenant
8. 写一条 SaasAuditLog
```

试用期也不是任意值，而是做了上限：

```ts
const trialDays = Math.min(input.trialDays ?? 14, 90);
```

租户状态根据是否有试用期进入：

```text
trialing
或
active
```

这个函数让我后来觉得“多租户开通”本身也应该是一个业务事务，而不应该让管理员手工往四五张表里各插一行。

## 9. 套餐不是只显示价格，还真正控制资源额度

`SaasPlan` 里有两个 JSON：

```text
features
limits
```

比如可以表达：

```json
{
  "features": {
    "orders": true,
    "workers": true,
    "stores": true
  },
  "limits": {
    "stores": 10,
    "workers": 100,
    "orders": 10000
  }
}
```

`SaasService.entitlements()` 会把当前租户、订阅、套餐和当前周期使用量一起解析成权限状态。

真正计量时走的是：

```ts
recordUsage(adminId, input)
```

逻辑不是简单 `usage++`，而是先判断：

```text
Tenant 状态是否 trialing / active
Subscription 状态是否 trialing / active
当前 metric 是否有额度
本次增加后会不会超过额度
```

核心判断类似：

```ts
const limit = rights.limits[input.metric];
const next = (rights.usage[input.metric] || 0) + input.quantity;

if (Number.isFinite(limit) && limit >= 0 && next > limit) {
  throw new ForbiddenException(`已达到套餐额度：${input.metric}`);
}
```

然后再按自然月做 `TenantUsage` upsert。

这里 `-1` 被定义成不限量。

## 10. 我后来还专门给“额度”写了失败测试

这部分至少有自动化证据，不是只看代码觉得合理。

`saas.service.spec.ts` 里覆盖了几个反例：

### 非法功能配置

```ts
features: { orders: "yes" }
```

应该抛 `BadRequestException`。

### 非法额度

```ts
limits: { stores: -2 }
```

因为只允许 `-1` 或非负整数，所以也应该拒绝。

### 被暂停的租户不能继续计量

```text
tenantStatus = suspended
subscriptionStatus = active
```

调用 `recordUsage()` 应该抛 `ForbiddenException`。

### 已经到上限后不能再加

```text
stores limit = 2
stores usage = 2
再 +1
```

同样应该拒绝。

这类测试比“页面上看起来显示了套餐”更能证明额度逻辑至少被编码成了业务约束。

## 11. 自定义域名这里还有一个很值得保留的未完成点

SaaS 模块已经实现了添加租户域名。

`addDomain()` 会：

```text
规范化 hostname
→ 生成 24-byte 随机 verification token
→ 创建 TenantDomain
→ 返回需要配置的 DNS TXT 记录
```

形式类似：

```text
TXT
_ctxc.example.com
<verification-token>
```

`resolveByHostname()` 也只会解析：

```text
Tenant 状态有效
+
TenantDomain.status = verified
```

的域名。

但是当前 `verifyDomain()` 本身做的是：

```ts
update({
  status: "verified",
  verifiedAt: new Date(),
})
```

**它并没有在这个函数里真的请求 DNS 去验证 TXT。**

所以这部分如果写成：

> 已实现完整自定义域名 DNS 验证。

就是过度描述。

更准确的是：

> 已完成域名模型、verification token、verified 状态和按 hostname 解析租户的主流程，但当前验证动作仍缺真正 DNS 查询这一层。

这种“代码已经有 80%，但最后 20% 还没闭环”的地方，我觉得反而比把文章写得很圆满更有复盘价值。

## 12. 我现在会怎么总结这次演进

这次项目真正发生过的架构变化，不是：

```text
单商户表
→ 每张表加 tenantId
```

而更接近：

```text
单商户默认世界
        ↓
识别出经营主体 Partner
        ↓
把区域 OperatingArea 独立出来
        ↓
用 PartnerServiceOffering 表达本地服务和价格
        ↓
订单 / 履约 / 支付 / 结算拆生命周期
        ↓
角色权限增加 DataScope
        ↓
商业化后再引入 Tenant
        ↓
套餐 / 订阅 / 成员 / 品牌 / 域名 / 额度 / 审计
```

其中有些是设计决策，有些后来真正进入了 Prisma schema、NestJS Service 和测试；自动打款、提现、完整 DNS 验证等则不能因为设计过就算“完成”。

这也是我现在写项目复盘最想保留的一点：

> **项目的价值不只在“最后做出了什么”，也在于哪些边界被真正落进代码，哪些只停留在设计，哪些直到项目结束都还没有闭环。**

如果这三种状态不分开，技术复盘就很容易从经验总结变成一篇看起来很完整、但无法验证的故事。