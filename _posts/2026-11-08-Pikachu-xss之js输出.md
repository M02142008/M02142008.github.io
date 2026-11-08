# XSS 之 js 输出 通关笔记
> 这关教的是最容易做错的一类：
>
> **输入被塞进了 JavaScript 字符串里。**
>
> 这时候你用 HTML 转义，不但防不住，还会把功能搞坏。
---
## 一、环境与目标
| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | Cross-Site Scripting → xss之js输出 |
| 页面地址 | `http://pikachu/vul/xss/xss_04.php` |
| 页面形式 | 输入框，问「which NBA player do you like?」 |
| 目标 | 让页面弹窗 |
---
## 二、我的解题步骤
### 步骤 1：先看正常功能
根据页面提示，输入 `tmac`。
提交后，下方显示：
```
tmac确实厉害,看那小眼神..
```
说明这个功能是：**你输入球员名，页面回一句评价。**
### 步骤 2：看源码，找到输出位置
```php
if(isset($_GET['submit']) && $_GET['message'] !=null){
    $jsvar=$_GET['message'];
}
```
```html
<script>
    $ms='<?php echo $jsvar;?>';
    if($ms.length != 0){
        if($ms == 'tmac'){
            $('#fromjs').text('tmac确实厉害,看那小眼神..')
        }else {
            $('#fromjs').text('无论如何不要放弃心中所爱..')
        }
    }
</script>
```
**`$jsvar` 被直接拼进了 JS 字符串字面量 `'...'` 里，没有任何转义。**
### 步骤 3：先试常规 payload（会失败）
输入：
```html
<script>alert(1)</script>
```
提交后，页面显示的是……**"无论如何不要放弃心中所爱.."**
因为 `$ms` 只是变成了字符串 `<script>alert(1)</script>`，它不等于 `'tmac'`，所以走了 else 分支。
**它没有执行，因为它现在只是"字符串内容"，不是"标签"。**
### 步骤 4：闭合字符串
既然是 `$ms='...'`，那就把引号闭合掉：
```javascript
';alert(1);//
```
提交后，**页面弹窗。**
---
## 三、原理分析
### 3.1 payload 代入后发生了什么
模板：
```javascript
$ms='[payload]';
```
payload 是 `';alert(1);//`：
```javascript
$ms='';alert(1);//';
```
拆解：
| 部分 | 作用 |
| --- | --- |
| `'` | 闭合字符串 |
| `;` | 结束语句 |
| `alert(1)` | 执行 |
| `//` | 注释掉后面的 `';` |
**这是一段完全合法的 JS，所以浏览器直接执行了。**
### 3.2 我验证过的真实输出
服务器返回的 script 块：
```html
<script>
    $ms='';alert(1);//';
    if($ms.length != 0){
        if($ms == 'tmac'){
            $('#fromjs').text('tmac确实厉害,看那小眼神..')
        }else {
            $('#fromjs').text('无论如何不要放弃心中所爱..')
        }
    }
</script>
```
`$ms='';` 之后立刻执行 `alert(1)`，后面的代码被 `//` 注释掉。
**弹窗后，页面上原本的评价文字不会出现**——因为后续逻辑被注释掉了。
### 3.3 为什么用 `//` 注释
如果不注释：
```javascript
$ms='';alert(1);';      // 末尾的 ' 会造成语法错误
```
报错可能导致 `alert` 之后的代码不执行，但那行 `alert(1)` 已经执行了。
**加 `//` 更干净，避免干扰。**
---
## 四、知识点与笔记
### 4.1 这是第几种输出位置
到这里，XSS 的四种典型输出位置就齐了：
| # | 输出位置 | 例子 | 需要防什么 |
| --- | --- | --- | --- |
| 1 | HTML 正文 | `<p>{$x}</p>` | `<` `>` |
| 2 | HTML 属性 | `<input value='{$x}'>` | 引号 |
| 3 | URL 属性 | `<a href='{$x}'>` | 协议 |
| 4 | **JS 字符串** | `var s='{$x}';` | **引号、反斜杠、换行、`</script>`** |
**四种位置，四种防御，四种 payload。**
### 4.2 为什么 HTML 转义在这里没用
作者在源码注释里写得很清楚：
> javascript里面是不会对tag和字符实体进行解释的，所以需要进行js转义
>
> 这里如果进行html的实体编码，虽然可以解决XSS的问题，但是实体编码后的内容，在JS里面不会进行翻译，这样会导致前端的功能无法使用。
实际效果：
```javascript
// 输入 ' onclick=alert(1)
// HTML 转义后变成 &#039; onclick=alert(1)
<script>
$ms='&#039; onclick=alert(1)';
</script>
```
- `&#039;` 在 HTML 里是单引号
- **但在 JS 字符串里，它就是 6 个普通字符 `& # 0 3 9 ;`**
- 所以字符串没被闭合，XSS 确实防住了
- **但用户输入的单引号也永远显示成 `&#039;` 了**
**结果：安全了，但功能坏了。**
### 4.3 JS 字符串里要防什么
| 字符 | 为什么要防 |
| --- | --- |
| `'` `"` | 闭合字符串 |
| `\` | 转义字符，可以构造 `\'` |
| 换行 | 字符串不能跨行 |
| `</script>` | 会提前结束 script 块（HTML 解析层） |
| `<!--` | 可能被当注释 |
### 4.4 正确做法一：json_encode（推荐）
```php
$jsvar = json_encode($_GET['message']);
```
```html
<script>
    var $ms = <?php echo $jsvar; ?>;   // 注意：不要加引号
</script>
```
`json_encode` 会输出：
```
"';alert(1);//"
```
带双引号的字符串字面量，引号和反斜杠都被转义了。
**注意 `</script>` 的问题**：JSON 里的 `</script>` 会提前结束 script 标签，所以 PHP 里要用：
```php
json_encode($data, JSON_HEX_TAG | JSON_HEX_AMP | JSON_HEX_APOS | JSON_HEX_QUOT)
```
或者用 `JSON_UNESCAPED_SLASHES` 之外的安全选项。
### 4.5 正确做法二：不要拼进 JS
更好的思路是：**把数据放到 data 属性或 API 里，用 JS 去读。**
```html
<div id="fromjs" data-msg="<?php echo htmlspecialchars($jsvar, ENT_QUOTES); ?>"></div>
```
```javascript
var msg = document.getElementById("fromjs").dataset.msg;
document.getElementById("fromjs").textContent = msg;
```
**数据走 HTML 属性（用 HTML 转义），而不是拼进 JS 代码。**
### 4.6 正确做法三：直接用 textContent
```javascript
// 危险
$('#fromjs').html(data);
// 安全
$('#fromjs').text(data);
```
---
## 五、和前面几关的区别
| 关卡 | 输出位置 | payload | 防御 |
| --- | --- | --- | --- |
| htmlspecialchars | HTML 属性 | `' onclick='alert(1)` | `ENT_QUOTES` |
| href输出 | URL 属性 | `javascript:alert(1)` | 协议白名单 |
| **js输出** | **JS 字符串** | `';alert(1);//` | **JS 转义 / json_encode** |
三关的共同主题：**输出位置决定防御方式。**
而且这一关还多了一个反例：
> **用错位置的防御，不但没用，还会破坏功能。**
---
## 六、防御怎么写
### 6.1 标准写法
```php
<script>
    var $ms = <?php echo json_encode($_GET['message'], JSON_HEX_TAG | JSON_HEX_AMP | JSON_HEX_APOS | JSON_HEX_QUOT); ?>;
</script>
```
### 6.2 更推荐：数据与代码分离
```html
<div id="app" data-message="<?php echo htmlspecialchars($msg, ENT_QUOTES, 'UTF-8'); ?>"></div>
```
```javascript
const el = document.getElementById('app');
el.textContent = el.dataset.message;
```
### 6.3 前端渲染时
| 写法 | 安全 |
| --- | --- |
| `el.textContent = x` | 安全 |
| `el.innerText = x` | 安全 |
| `el.innerHTML = x` | 危险 |
| jQuery `.text(x)` | 安全 |
| jQuery `.html(x)` | 危险 |
| React `{x}` | 安全 |
| React `dangerouslySetInnerHTML` | 危险 |
| Vue `{{ x }}` | 安全 |
| Vue `v-html` | 危险 |
---
## 七、总结
| 要点 | 内容 |
| --- | --- |
| 这关教什么 | JS 上下文的 XSS |
| 关键点 | 输入被拼进 `var s='...'` |
| payload | `';alert(1);//` |
| 为什么 HTML 转义没用 | JS 不解析 HTML 实体，会破坏功能 |
| 修复 | `json_encode` + 正确的 flags |
**一句话**：
> **转义要"看上下文"。HTML 转义防不住 JS 字符串，就像锁装错了门。**
---
## 参考
- [OWASP: XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [PHP: json_encode](https://www.php.net/manual/zh/function.json-encode.php)
- [MDN: textContent](https://developer.mozilla.org/zh-CN/docs/Web/API/Node/textContent)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
