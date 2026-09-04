---
title: "Termux + root 排查小米应用商店忽略更新：从 market_2.db 开始"
description: "把一次真实操作重新按证据链整理：我能确认改过哪个数据库、怎么进入和检查；无法从旧记录恢复的字段与 SQL 明确标注未知。"
pubDate: 2026-09-04
tags:
  - Android
  - SQLite
  - Debug
draft: false
---

这篇文章是对我之前一次小米应用商店“忽略更新”操作的重新整理。

先把证据边界写在前面，避免把推测写成事实。

## 我现在能确认什么

我能确认的原始操作信息有三点：

1. 当时是在 **Termux** 里操作；
2. 通过 `su` 获取 root shell；
3. 修改的是小米应用商店的 **`market_2.db`**，目的就是改变某些应用的更新提示状态。

公开的 Android 取证资料也能交叉验证：小米应用商店包名为 `com.xiaomi.market`，其数据库中确实存在 `market_2.db`，常见位置是：

```text
/data/data/com.xiaomi.market/databases/market_2.db
```

部分取证镜像会把同一路径表示成：

```text
/data/com.xiaomi.market/databases/market_2.db
```

公开资料还能确认，这个数据库会保存应用相关记录，例如历史安装信息、包名、安装路径等。

**但我目前无法从旧聊天记录里恢复当时具体修改的表名、字段名和最终那条 `UPDATE` SQL。**

所以这里不会编一个看起来合理的字段名充数。下面只写我现在有把握复现的排查路径。

## 1. 先确认应用商店和数据库

先看包是否存在：

```sh
pm path com.xiaomi.market
```

然后进入 root shell：

```sh
su
```

确认数据库：

```sh
ls -lah /data/data/com.xiaomi.market/databases/
```

目标文件是：

```text
market_2.db
```

这一步的意义是先确认“我改的是哪个持久化状态”，而不是一上来猜 SharedPreferences 或改 APK。

## 2. 用 Termux 的 sqlite3 直接看库

Termux 里先安装 SQLite：

```sh
pkg install sqlite
```

Termux 的 sqlite3 通常位于：

```text
/data/data/com.termux/files/usr/bin/sqlite3
```

进入 root 后可以直接指定完整路径：

```sh
DB=/data/data/com.xiaomi.market/databases/market_2.db
SQLITE=/data/data/com.termux/files/usr/bin/sqlite3

$SQLITE "$DB" '.tables'
```

接着看所有建表语句：

```sh
$SQLITE "$DB" '.schema'
```

这一步比直接抄网上某条 SQL 更重要，因为系统应用商店版本变化后，数据库 schema 也可能变化。

## 3. 我会怎么定位目标应用对应的记录

假设要忽略更新的应用包名是：

```text
com.example.target
```

最直接的办法不是先猜表名，而是先 dump 后全文找包名：

```sh
$SQLITE "$DB" '.dump' | grep -nF 'com.example.target'
```

如果命中了某张表，就继续针对那张表检查：

```sql
.schema <命中的表名>
```

然后再查询目标记录：

```sql
SELECT *
FROM <命中的表名>
WHERE <包名字段> = 'com.example.target';
```

到这里，才能知道这个版本实际使用的是哪些字段，而不是凭经验猜 `ignore`、`ignored`、`versionCode` 之类的名字。

## 4. 修改前先停进程和备份

SQLite 可能同时存在 WAL 文件。如果应用商店正在运行，直接复制或修改主库可能得到不一致状态。

我现在会先停掉应用商店：

```sh
am force-stop com.xiaomi.market
```

然后备份数据库及可能存在的 WAL/SHM：

```sh
cp -a "$DB" /sdcard/Download/market_2.db.bak

[ -f "${DB}-wal" ] && cp -a "${DB}-wal" /sdcard/Download/market_2.db-wal.bak
[ -f "${DB}-shm" ] && cp -a "${DB}-shm" /sdcard/Download/market_2.db-shm.bak
```

至少这样改错后还有回退点。

## 5. 真正的修改应该建立在“schema 已确认”之后

这里是我之前那篇文章最大的问题：只写了“理解数据模型”，却没有说明到底如何落到修改。

实际操作一定会走到类似下面这一步：

```sql
UPDATE <实际表名>
SET <实际状态字段> = <目标值>
WHERE <包名字段> = 'com.example.target';
```

但我要强调：**上面的尖括号不是教程里的省略号，而是我现在确实没有证据恢复出的旧版本字段。**

我只记得最终是通过修改 `market_2.db` 的目标记录达到忽略更新的效果；如果我现在直接写成：

```sql
UPDATE apps SET ignored = 1 ...
```

那就是编造。

## 6. 如果重新做一次，我会用“前后 diff”把字段找出来

这是现在回头看最稳的方法。

如果当前版本还能找到任意一个可由 UI 改变的“忽略/恢复更新”状态，可以做一个控制实验：

```sh
$SQLITE "$DB" '.dump' > /sdcard/Download/market.before.sql
```

在 UI 中改变一个测试应用的状态，再停掉应用商店，然后：

```sh
$SQLITE "$DB" '.dump' > /sdcard/Download/market.after.sql

diff -u \
  /sdcard/Download/market.before.sql \
  /sdcard/Download/market.after.sql
```

这样可以直接看到：

- 哪张表变了；
- 哪一行变了；
- 哪个字段承载这个状态；
- 它是按包名记录，还是按 `package + version` 记录。

如果 UI 完全没有对应入口，也可以先通过包名定位相关记录，再结合 schema 和应用行为逐步验证，但此时更应该记录每次修改前后的实际值。

## 7. 验证不能只看 SQL 执行成功

真正的验收应该是行为层面的。

至少要做：

```sh
am force-stop com.xiaomi.market
```

重新打开应用商店，重新进入“应用升级”页面，并触发一次更新检查。

要确认的是：

- 目标应用是否从待更新列表消失；
- 重启应用商店后状态是否仍保留；
- 刷新服务端数据后是否会被覆盖；
- 新版本发布后是继续忽略，还是只忽略当前版本。

SQL 改成功，只能证明数据库写进去了；**UI 和下一次同步后的行为才是最终结果。**

## 这次我真正学到的东西

这件事有价值的地方并不是“root 后可以改数据库”。

而是一个比较通用的排查路径：

```text
UI 没入口
  ↓
确定负责该功能的系统应用
  ↓
定位持久化文件
  ↓
查看 schema，而不是猜字段
  ↓
按包名定位真实记录
  ↓
修改前备份
  ↓
修改后做行为验证
```

以及另一个对写技术记录更重要的教训：

> 没有保存下来的字段名和 SQL，就应该明确写“未知”，而不是几年后根据经验补一个看起来合理的答案。

## 可交叉验证的公开资料

- 小米官方隐私政策说明应用商店会读取已安装应用及版本，用于检查应用更新：<https://privacy.mi.com/xiaomi-market/zh_CN/xiaomi-market%E9%9A%90%E7%A7%81%E6%94%BF%E7%AD%96.pdf>
- 公开 Android 取证记录中可见 `com.xiaomi.market/databases/market_2.db`，且该库包含应用安装历史等信息：<https://www.cnblogs.com/WXjzc/p/17728788.html>
- 另一份取证记录同样使用 `market_2.db` 的 `history` 表分析应用记录：<https://blog.csdn.net/2401_87205029/article/details/142332392>

这些资料只能用于交叉验证数据库的存在和用途，**不能替代我当时具体版本的 schema 证据**。