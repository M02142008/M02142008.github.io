# DOM型 XSS-X 通关笔记
> 和上一关几乎一样，唯一的区别是：**数据来源从"输入框"变成了"URL 参数"**。
>
> 这一关还多了一个动作：payload 要先提交一次，再点一下链接，才会触发。
---
## 一、环境与目标
| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | Cross-Site Scripting → DOM型xss-x |
| 页面地址 | `http://pikachu/vul/xss/xss_dom_x.php` |
| 页面形式 | 输入框 + 「请说出你的伤心往事」按钮 |
| 目标 | 让页面弹出一个对话框 |
页面流程：
```
[__________]  [请说出你的伤心往事]
        ↓ 提交
（出现一行链接：有些费尽心机想要忘记的事情...）
        ↓ 点击
（触发 JS）
```
---
## 二、我的解题步骤
### 步骤 1：先随便输点东西，看看流程
输入 `test`，提交。
页面下方出现一行可点击的文字：
> 有些费尽心机想要忘记的事情,后来真的就忘掉了
点它，**没反应**。
### 步骤 2：看源码，找到数据来源
```javascript
function domxss(){
    var str = window.location.search;
    var txss = decodeURIComponent(str.split("text=")[1]);
    var xss = txss.replace(/\+/g,' ');
    document.getElementById("dom").innerHTML = "<a href='"+xss+"'>就让往事都随风,都随风吧</a>";
}
```
关键：
- `window.location.search` → 取的是 **URL 里的查询字符串**（`?text=xxx`）
- `decodeURIComponent()` → 解码一次
- `innerHTML` → 危险 sink
**也就是说：输入框里的内容根本没用，真正被读取的是 URL。**
### 步骤 3：直接改 URL，绕过输入框
访问：
```
http://pikachu/vul/xss/xss_dom_x.php?text=%27%3E%3Cimg%20src%3Dx%20onerror%3Dalert(1)%3E
```
编码对照：
| 编码 | 原字符 |
| --- | --- |
| `%27` | `'` |
| `%3E` | `>` |
| `%3C` | `<` |
| `%20` | 空格 |
| `%3D` | `=` |
### 步骤 4：点那行链接
点「有些费尽心机想要忘记的事情...」。
**弹窗了。**
---
## 三、原理分析
### 3.1 完整源码
**前端：**
```javascript
function domxss(){
    var str = window.location.search;
    var txss = decodeURIComponent(str.split("text=")[1]);
    var xss = txss.replace(/\+/g,' ');
    document.getElementById("dom").innerHTML = "<a href='"+xss+"'>就让往事都随风,都随风吧</a>";
}
```
**服务端：**
```php
if(isset($_GET['text'])){
    $html.= "<a href='#' onclick='domxss()'>有些费尽心机想要忘记的事情,后来真的就忘掉了</a>";
}
```
服务端只做了一件事：**如果 URL 里有 `text` 参数，就渲染一个链接，点击时调用 `domxss()`。**
### 3.2 payload 代入后发生了什么
模板：
```javascript
"<a href='"+xss+"'>就让往事都随风,都随风吧</a>"
```
`xss` 经过 `decodeURIComponent` 之后是 `'><img src=x onerror=alert(1)>`：
```html
<a href=''><img src=x onerror=alert(1)>'>就让往事都随风,都随风吧</a>
```
- `'` 闭合 `href`
- `>` 闭合 `<a>`
- `<img onerror>` 触发执行
### 3.3 为什么 payload 要 URL 编码
因为 `location.search` 拿到的是**原始字符串**：
```
?text=%27%3E%3Cimg%20src%3Dx%20onerror%3Dalert(1)%3E
```
代码里有 `decodeURIComponent()`，会把它解码回：
```
'><img src=x onerror=alert(1)>
```
**如果你不编码直接写**：
```
?text='><img src=x onerror=alert(1)>
```
浏览器地址栏会把 `<` `>` 自动编码，大部分情况也行。但 **`'` 和空格容易出问题**，所以养成手动编码的习惯更稳。
---
## 四、知识点与笔记
### 4.1 和上一关的唯一区别
| | DOM型xss | DOM型xss-x |
| --- | --- | --- |
| Source | `document.getElementById("text").value` | `window.location.search` |
| 需要交互吗 | 点按钮 | 提交一次 + 点链接 |
| payload 放哪 | 输入框 | **URL 参数** |
| Sink | `innerHTML` | `innerHTML`（一样） |
**Sink 没变，只是 Source 变了。** 这就是 DOM XSS 的典型变体。
### 4.2 `location.search` 是什么
```javascript
// 假设 URL 是 http://a.com/x.php?text=abc&id=1
location.search      // "?text=abc&id=1"
location.href        // 完整 URL
location.hash        // "#top"（# 后面的部分）
```
### 4.3 为什么 payload 里不能有 `&`
代码用的是：
```javascript
str.split("text=")[1]
```
它会取 `text=` **之后的所有内容**。
如果 payload 里有 `&`，比如：
```
?text=abc&payload
```
`split("text=")[1]` 得到的是 `abc&payload`，本身没问题。但如果是多参数：
```
?text=1&other=2
```
取到的就是 `1&other=2`。
**更常见的坑是**：如果后面还要拼别的参数，`&` 会把 payload 截断。所以 payload 里尽量别用 `&`。
### 4.4 `decodeURIComponent` 只解码一次
```javascript
decodeURIComponent("%2541")   // "%41"，不是 "A"
```
**双重编码可以绕过只解码一次的过滤。**
### 4.5 这类关卡在真实场景中的形态
很多单页应用（SPA）会这样做：
```javascript
var name = location.hash.slice(1);
document.getElementById("welcome").innerHTML = "欢迎你，" + name;
```
用户访问 `#<img src=x onerror=alert(1)>` 就中招。
**这就是典型的 DOM XSS，服务器完全不知情。**
---
## 五、和前面几关的区别
| 关卡 | Source | 有没有经过服务器 |
| --- | --- | --- |
| 反射型 GET | URL 参数（服务器读取） | **经过** |
| DOM型xss | 输入框 | 不经过 |
| **DOM型xss-x** | **URL 参数（前端读取）** | **不经过** |
注意最后一行的微妙之处：
> **同样是 URL 参数，反射型是"服务器读"，DOM 型是"前端读"。**
抓包时两者看起来都像"URL 里有参数"，但服务器返回的 HTML 完全不同。
---
## 六、防御怎么写
### 6.1 用 `textContent` 代替 `innerHTML`
```javascript
document.getElementById("dom").textContent = xss;
```
### 6.2 必须构造链接时，手动创建元素
```javascript
var a = document.createElement("a");
a.href = "#";
a.textContent = "就让往事都随风";
document.getElementById("dom").appendChild(a);
```
### 6.3 不要信任 `location.*`
```javascript
// 危险
el.innerHTML = location.search;
// 安全
el.textContent = new URLSearchParams(location.search).get("text") || "";
```
---
## 七、总结
| 要点 | 内容 |
| --- | --- |
| 这关教什么 | DOM XSS 的 source 可能来自 URL |
| 关键点 | 输入框没用，真正被读的是 `location.search` |
| 触发方式 | 先带 payload 访问，再点链接 |
| 防御 | 同样是用 `textContent`，不拼 HTML |
**一句话**：
> **反射型是"服务器把参数拼进页面"，DOM 型是"前端自己把参数拼进页面"——看起来像，根子不同。**
---
## 参考
- [OWASP: DOM based XSS Prevention](https://cheatsheetseries.owasp.org/cheatsheets/DOM_based_XSS_Prevention_Cheat_Sheet.html)
- [MDN: Location.search](https://developer.mozilla.org/zh-CN/docs/Web/API/Location/search)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
