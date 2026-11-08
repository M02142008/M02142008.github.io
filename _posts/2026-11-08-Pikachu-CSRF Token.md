# CSRF Token 通关笔记
> 前两关都是"没有防护"，这一关终于加上了 token。
>
> 但我在测的时候发现了一个**边界情况**：如果受害者的 session 里还没生成过 token，攻击者不带 token 参数发请求，`null == null` 居然成立，照样能改。
---
## 一、环境与目标
| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | CSRF → CSRF Token |
| 登录页 | `http://pikachu/vul/csrf/csrftoken/token_get_login.php` |
| 会员中心 | `http://pikachu/vul/csrf/csrftoken/token_get.php` |
| 修改页 | `http://pikachu/vul/csrf/csrftoken/token_get_edit.php` |
| 测试账号 | `vince` / `123456` |
| 目标 | 尝试用前两关的方式伪造请求，看 token 能不能拦住 |
---
## 二、我的解题步骤
### 步骤 1：登录，打开修改页
登录 `vince` / `123456`，点「修改个人信息」。
按 `F12` 看表单，发现多了一个隐藏字段：
```html
<input type="hidden" name="token" value="774046af079eea08d1390967549" />
```
**这就是 token。**
### 步骤 2：先按前两关的方式打——不带 token
构造和 CSRF(get) 一样的请求：
```
http://pikachu/vul/csrf/csrftoken/token_get_edit.php?sex=hacked3&phonenum=330330&add=HACKED3&email=h3@evil.com&submit=submit
```
**结果：居然改成功了。**
| 字段 | 修改前 | 修改后 |
| --- | --- | --- |
| 性别 | boy | **hacked3** |
| 手机 | 18626545453 | **330330** |
| 住址 | chain | **HACKED3** |
| 邮箱 | vince@pikachu.com | **h3@evil.com** |
### 步骤 3：再试一次，这次失败了
把同样的请求再发一次（还是不带 token），换成新的值。
**这次没改成功。**
### 步骤 4：读源码，找原因
```php
if(isset($_GET['submit'])){
    if($_GET['sex']!=null && $_GET['phonenum']!=null && $_GET['add']!=null && $_GET['email']!=null && $_GET['token']==$_SESSION['token']){
        // 执行 update
    }
}
//生成token
set_token();
```
关键在最后那个条件：
```php
$_GET['token'] == $_SESSION['token']
```
**第一次为什么能过？**
- 刚登录，从没打开过修改页 → `$_SESSION['token']` **还没被设置**，是 `null`
- 请求里也没带 token → `$_GET['token']` 也是 `null`
- **`null == null` → true** → 条件成立 → 执行 update
**第二次为什么过不了？**
- 第一次请求末尾执行了 `set_token()`，session 里已经有 token 了
- 再不带 token → `null == "774046..."` → false → 不执行
### 步骤 5：带正确 token 才能改
从修改页的隐藏字段里读出 token，带上它再发请求：
```
token_get_edit.php?sex=...&phonenum=...&add=...&email=...&token=774046af...&submit=submit
```
**改成功了。**
---
## 三、原理分析
### 3.1 源码
```php
if(isset($_GET['submit'])){
    if($_GET['sex']!=null && $_GET['phonenum']!=null && $_GET['add']!=null && $_GET['email']!=null && $_GET['token']==$_SESSION['token']){
        $getdata=escape($link, $_GET);
        $query="update member set sex='{$getdata['sex']}',phonenum='{$getdata['phonenum']}',address='{$getdata['add']}',email='{$getdata['email']}' where username='{$_SESSION['csrf']['username']}'";
        $result=execute($link, $query);
        if(mysqli_affected_rows($link)==1 || mysqli_affected_rows($link)==0){
            header("location:token_get.php");
        }
    }
}
//生成token
set_token();
```
`set_token()` 的实现（`inc/function.php`）：
```php
function set_token(){
    if(isset($_SESSION['token'])){
       unset($_SESSION['token']);
    }
    $_SESSION['token']=str_replace('.','',uniqid(mt_rand(10000,99999),true));
}
```
### 3.2 我验证过的四种情况
| # | 请求 | session 里有 token 吗 | 结果 |
| --- | --- | --- | --- |
| 1 | **不带 token** | **没有**（刚登录） | **改成功**（null == null） |
| 2 | 不带 token | 有 | 没改（null != "774046..."） |
| 3 | `token=`（空字符串） | 有 | 没改（"" != "774046..."） |
| 4 | 带正确 token | 有 | 改成功 |
**只有第 1 种情况是漏洞。**
### 3.3 这个漏洞在真实场景中成立吗
**成立，但有前提。**
攻击者需要满足：
1. 受害者已经登录
2. 受害者**还没有打开过**「修改个人信息」页面（session 里没生成过 token）
这个条件并不苛刻：
- 用户登录后通常先看会员中心，不一定马上点"修改信息"
- 攻击者可以在用户刚登录的时间窗口内投递
**一旦用户打开过一次修改页，这个路就走不通了。**
### 3.4 但 token 本身是有效的
对于正常的 CSRF（前两关那种），token 完全防得住：
```
攻击者的页面
    ↓ 想读目标站的 token
    ↓ 同源策略阻止（读不到别的域的 DOM）
    ↓ 拿不到 token
    ↓ 请求被拒绝
```
**这就是 token 能防 CSRF 的根本原因：它防的不是"猜"，而是"读不到"。**
---
## 四、知识点与笔记
### 4.1 CSRF Token 的设计要点
| 要点 | 说明 |
| --- | --- |
| **不可预测** | 随机生成，不能用时间戳、递增 ID |
| **绑定会话** | 存在 session 里，跟用户绑定 |
| **不可被第三方读取** | 靠同源策略保证 |
| **一次性（可选）** | 用完即换，防重放 |
| **每个敏感操作都校验** | 不能只在登录时校验一次 |
### 4.2 为什么 token 能防 CSRF
关键在于**同源策略**：
| 攻击者想做的事 | 能做到吗 |
| --- | --- |
| 让浏览器发请求到目标站 | ✅ 能 |
| 读目标站返回的内容 | ❌ 不能 |
| 读目标站页面里的 token | ❌ 不能 |
**CSRF 的本质是"能发不能读"，token 正好利用了这一点。**
### 4.3 这个 token 实现的问题
| 问题 | 说明 |
| --- | --- |
| **`==` 松散比较** | 应该用 `hash_equals()`，防时序攻击 |
| **null == null** | 未初始化的 session token 会放过空 token |
| **token 放在 URL 里** | GET 表单，token 会出现在地址栏、日志、Referer 里 |
| **没有一次性校验** | 校验完不销毁（`set_token()` 在渲染时换，但逻辑上不清爽） |
**其中最要命的是第 2 条**——未初始化时 `null == null`。
### 4.4 正确的 token 校验
```php
// 1. 先判断 token 是否存在（关键，防 null == null）
if (!isset($_SESSION['token']) || !isset($_POST['token'])) {
    exit('非法请求');
}
// 2. 用 hash_equals 做恒定时间比较
if (!hash_equals($_SESSION['token'], $_POST['token'])) {
    exit('非法请求');
}
// 3. 一次性
unset($_SESSION['token']);
```
**第 1 步的 `isset` 检查，就是为了堵住我这个发现的口子。**
---
## 五、和前面几关的区别
| | CSRF(get) | CSRF(post) | CSRF Token |
| --- | --- | --- | --- |
| token | 无 | 无 | **有** |
| 直接伪造 | ✅ 成功 | ✅ 成功 | ❌ 失败 |
| 我的实测 | 改成功 | 改成功 | **第 1 次成功，之后失败** |
| 根本原因 | 没防护 | 没防护 | **token 判空有缺陷** |
**这一关是本模块唯一有"防护"的一关，也是唯一需要动脑子绕的一关。**
---
## 六、防御怎么写
### 6.1 标准实现
```php
// 生成（登录后 / 每次页面渲染）
function csrf_token() {
    if (empty($_SESSION['csrf_token'])) {
        $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
    }
    return $_SESSION['csrf_token'];
}
// 校验
function csrf_check($token) {
    if (empty($_SESSION['csrf_token']) || empty($token)) {
        return false;   // 关键：空值直接拒绝
    }
    return hash_equals($_SESSION['csrf_token'], $token);
}
```
### 6.2 框架怎么做的
| 框架 | 机制 |
| --- | --- |
| Laravel | `@csrf` 指令，中间件自动校验 |
| Django | `{% csrf_token %}`，中间件校验 |
| Spring Security | `CsrfFilter` |
| Express (csurf) | 中间件 |
**框架默认都做了，但要注意：**
- 不要为了"方便"关掉
- API 接口如果不用 Cookie 认证（用 Bearer Token），可以不做 CSRF 防护
### 6.3 多层防御
| 层 | 措施 |
| --- | --- |
| Cookie | `SameSite=Lax` 或 `Strict` |
| 请求头 | 校验 `Origin` / `Referer` |
| 参数 | CSRF Token |
| 业务 | 关键操作二次确认 |
| 传输 | 敏感操作强制 HTTPS |
**单靠一层都可能被绕过，多层才稳。**
---
## 七、总结
| 要点 | 内容 |
| --- | --- |
| 这关教什么 | token 是防 CSRF 的正确方向 |
| token 为什么有效 | 同源策略让攻击者读不到它 |
| 我发现的边界问题 | 未初始化时 `null == null` 绕过 |
| 正确写法 | `isset` 检查 + `hash_equals` + 一次性 |
| 部署位置 | 所有敏感操作都要校验 |
**一句话**：
> **token 防的不是"猜"，是"读不到"。所以只要判空写错，这道墙就漏了。**
---
## 参考
- [OWASP: CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [PHP: hash_equals](https://www.php.net/manual/zh/function.hash-equals.php)
- [MDN: 同源策略](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
