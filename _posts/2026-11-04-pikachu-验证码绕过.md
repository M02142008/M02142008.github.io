# 验证码在 Cookie 里：Pikachu「验证码绕过(on server)」复盘
> 90 个请求跑完，Length 那一列整整齐齐全是 `35376`，一条都没中。我一开始以为是 Session 过期了——毕竟社区版 Intruder 一秒一个请求，90 个跑了快两分钟。后来做了个对照实验才发现，**根因跟时间完全无关，是验证码在我按下 Forward 的那一刻就已经被换掉了。**
## 一、环境与目标
| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 运行环境 | phpStudy（Apache 2.4.39 + MySQL 5.7.26 + PHP 7.3.4 NTS） |
| 站点根目录 | `D:\Pikachu` |
| 访问方式 | `http://pikachu/` |
| 关卡位置 | 暴力破解 → 验证码绕过(on server) |
| 页面地址 | `http://pikachu/vul/burteforce/bf_server.php` |
| 工具 | Burp Suite 社区版 |
目标：用 `admin` 爆破出密码，让页面回显 `login success`。
## 二、这道题在考什么
### 2.1 和上一关的差别
页面比「基于表单的暴力破解」多了一个验证码输入框和一张扭曲的图片。表面上看，验证码把「无限次尝试」这条路堵死了。
但源码里有两处细节，把这层防护彻底废掉了。
### 2.2 先看验证码是怎么发出来的
`inc/showvcode.php` 全文只有 8 行，核心是这三句：
```php
session_start();
$_SESSION['vcode'] = vcodex();          // 生成验证码，存进 session
setcookie('bf[vcode]', $_SESSION['vcode']);  // 又把同一个值明文塞进 Cookie
```
作者自己在旁边留了一行注释：
> 服务端将验证码字符串以明文 COOKIE 的方式给了前端，那验证码还有什么鸟意义。。。
也就是说：**你根本不需要认那张图，答案就在 Cookie 里。**
### 2.3 再看校验逻辑
`bf_server.php` 的校验部分：
```php
if (strtolower($_POST['vcode']) != strtolower($_SESSION['vcode'])) {
    $html .= "<p class='notice'>验证码输入错误哦！</p>";
    //应该在验证完成后,销毁该$_SESSION['vcode']      ← 作者自己标的 TODO
} else {
    // 验证码通过，开始查数据库
    $sql = "select * from users where username=? and password=md5(?)";
    ...
    if ($line_pre->num_rows() == 1) {
        $html .= '<p> login success</p>';
    } else {
        $html .= '<p> username or password is not exists～</p>';
    }
}
```
校验通过之后，**没有 `unset($_SESSION['vcode'])`**。所以同一个验证码可以反复用，直到有人把图片重新加载一次。
### 2.4 两个洞叠加
| 缺陷 | 单独存在的后果 |
| --- | --- |
| 验证码明文写进 Cookie | 验证码等于没有，直接读 Cookie 就行 |
| 校验后不销毁 session | 一次有效 → 永久有效，可以直接拿去爆破 |
两个都在，再加上没有失败次数限制，这套验证码就彻底形同虚设。
## 三、常规解法
标准流程和上一关几乎一样，唯一的区别是 `vcode` 必须固定住。
1. Burp 开代理，用内置浏览器打开目标页；
2. 从 Cookie 里读出 `bf[vcode]` 的值（或者直接认图）；
3. 填 `admin` / 任意密码 / 验证码，点 Login；
4. 抓到 `POST /vul/burteforce/bf_server.php`，右键 → Send to Intruder；
5. Positions 里只把 `password` 的值包上 `§`，`vcode` 保持原样；
6. Payloads 挂字典，Settings 里加 Grep-Match `login success`；
7. Start attack，看结果。
## 四、我的实际操作
### 4.1 第一阶段：抓包
1. Burp → `Proxy` → `Intercept` → 点 `Open browser` 打开内置浏览器；
2. 把 `Intercept off` 点成 `Intercept on`，开始拦截；
3. 在浏览器里填 `admin` / `123456` / 看着图填了验证码 `zk1wsn`，点 Login；
4. 浏览器转圈，Burp 里成功拦到 POST 请求；
5. 右键 → `Send to Intruder`；
6. 点 `Forward` 放行，然后把 Intercept 关掉，让浏览器恢复正常。
第 6 步看起来只是收尾，实际上**就是它把我的验证码弄废了**——这一点当时完全没意识到，后面第五节详细说。
### 4.2 第二阶段：配置 Intruder
1. 切到 `Intruder → Positions`；
2. 点 `Clear §`——点不动。因为默认没有标记，这个按钮只有在已有 `§` 时才起作用；
3. 手动标记了两个位置：
   - `username=admin` 里的 `admin`
   - `password=123456` 里的 `123456`
4. `vcode=zk1wsn` 没有标记，保持原样；
5. 攻击类型改成 **Cluster bomb**（想着"用户名 × 密码"的组合）。
最终是这样：
```
username=§admin§&password=§123456§&vcode=zk1wsn&submit=Login
```
### 4.3 第三阶段：装 Payloads
1. 切到 `Payloads`；
2. `Payload position` 选 `1 - admin`，下面粘贴一个用户名；
3. 切到 `2 - 123456`，粘贴密码字典（`123456`、`admin888` 等，大约 90 条）。
中间还出了个小插曲：切到位置 2 时手滑把用户名又粘了一遍，导致第 2 个位置的 payload 类型不对，删掉重贴才好。
### 4.4 第四阶段：开火
1. 点 `Start attack`；
2. 社区版弹了个限速提示框，点掉之后，Burp 开始以**大约每秒 1 个请求**的速度慢慢跑；
3. 90 条字典跑了一分多钟。
### 4.5 结果：全军覆没
跑完之后看 `Length` 那一列：
```
35376
35376
35376
...（全是 35376）
```
一条都没有变化，`login success` 那个 Grep 列全空。
## 五、踩坑记录：为什么 90 个请求长得一模一样
### 5.1 我一开始的判断（错的）
回到浏览器手动提交一次，页面提示 **「验证码输入错误哦！」**。
我的第一反应是：**Session 过期了**。
理由听起来很合理——社区版一秒一个请求，90 个跑了一分多钟，等我跑到第 55 个（正好是 `admin/123456`）时，服务器早就不认最开始抓的那个 `zk1wsn` 了。所以后面全是"验证码错误"，长度自然完全一样。
**这个判断是错的。** 而且错得很典型：我把"慢"当成了"过期"的原因。
### 5.2 真正的原因：验证码被轮换了
问题出在第四节的第 6 步——**点 Forward 之后**。
登录请求被放行后，服务器返回的是完整的结果页面，而 `bf_server.php` 的 HTML 里始终带着这张图：
```html
<img src="../../inc/showvcode.php" onclick="this.src='../../inc/showvcode.php?'+new Date().getTime();" />
```
浏览器渲染这个响应时，会**立刻再去请求一次 `showvcode.php`**。而那个文件的第一件事就是：
```php
$_SESSION['vcode'] = vcodex();   // 重新生成，把旧值覆盖掉
```
所以时间线是这样的：
```
t0  打开页面        → showvcode.php 执行 → session.vcode = zk1wsn
t1  填表、提交      → 请求被 Burp 拦下（此时的 vcode 还是 zk1wsn）
t2  Send to Intruder
t3  点 Forward      → 登录请求到达服务器，用 zk1wsn 校验，通过
t4  浏览器渲染响应  → <img> 触发 showvcode.php → session.vcode 变成新值
t5  Start attack    → 90 个请求全部带着 zk1wsn，而 session 里早就是新值了
                    → 全部返回「验证码输入错误」
```
**验证码在 t4 就已经死了，不是在一分钟后才过期。**
### 5.3 一个关键的响应头
我又去看了一下 `showvcode.php` 的响应头：
```
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Expires: Thu, 19 Nov 1981 08:52:00 GMT
```
这个头的意思是：**浏览器绝对不许缓存这张图**。所以每次渲染页面，浏览器都必然重新请求它，session 里的 vcode 也就必然被刷新。这解释了为什么这个坑"一定会踩"，不是偶然。
### 5.4 对照实验（三分钟验证）
为了确认，我写了个小脚本，固定会话，只改密码，看服务端怎么回：
| 步骤 | 操作 | 结果 |
| --- | --- | --- |
| 1 | 请求 `showvcode.php`，拿到 vcode = `k8w1g6` | — |
| 2 | 用 `k8w1g6` 提交（密码故意错） | 密码错误（**验证码通过了**） |
| 3 | 再用 `k8w1g6` 提交（换个错密码） | 密码错误（**验证码还能用，没被销毁**） |
| 4 | 再请求一次 `showvcode.php` | vcode = `ycugzx`（**和上面不同**） |
| 5 | 用旧的 `k8w1g6` 提交 | **验证码输入错误** |
| 6 | 用新的 `ycugzx` 提交 | 密码错误（验证码又通过了） |
三个结论一次坐实：
1. 验证码**不会被销毁**（第 3 步）——所以"一次有效"这条是成立的；
2. 但只要**重新请求图片**，session 里的值就变（第 4 步）；
3. 旧值随即作废（第 5 步）。
所以我失败的原因是"验证码被换掉了"，不是"过期了"。
### 5.5 踩坑清单
| 现象 | 我当时的判断 | 真正原因 | 解决 |
| --- | --- | --- | --- |
| 90 个请求 Length 全是 35376 | Session 过期 | Forward 后浏览器重新加载了验证码图片，session 里的 vcode 被覆盖 | 别让图片重新加载，或用脚本固定 vcode |
| `Clear §` 点不动 | 按钮坏了 | 没有已标记的位置时它本来就没反应 | 先手动 Add §，再 Clear 才有意义 |
| 用 Cluster bomb | 觉得"用户名 × 密码"更正规 | 只有一个用户名时它会做笛卡尔积，容易配错 payload | 只有一个变量时用 Sniper 就够了 |
| 位置 2 粘进了用户名 | 手滑 | 切位置时没注意当前选中的是哪一项 | 粘贴前先确认右上角 `Payload position` |
## 六、我是怎么解决的
核心思路只有一句：**让 `showvcode.php` 在爆破期间不要再被执行。**
### 方案 A：在 Burp 里把图片请求 Drop 掉
点 `Forward` 放行登录包之后，浏览器会立刻发出那条 `GET /inc/showvcode.php`。因为 Intercept 还开着，这条请求也会被拦下来。
**这时候不要 Forward，直接点 `Drop`。**
请求根本没到服务器，`$_SESSION['vcode']` 就不会被刷新，手里那个值继续有效。然后再去 Intruder 开跑，90 个请求全部用的是同一份有效的 session。
要点：
- 放行登录包之后**盯住 Burp**，别急着关 Intercept；
- 看到 `GET /inc/showvcode.php` 就 Drop，不是 Forward；
- 之后不要刷新页面、不要关浏览器标签、不要点验证码图片。
### 方案 B：用脚本，绕开页面渲染
页面渲染是"重新请求图片"的唯一触发条件。那就干脆不渲染——直接按 HTTP 说话：
1. `GET /inc/showvcode.php`，从响应头里同时拿到 `PHPSESSID` 和 `bf[vcode]`；
2. 带着这两个 Cookie，直接循环 `POST /vul/burteforce/bf_server.php`；
3. 只换 `password`，`vcode` 全程不变。
因为中间没有浏览器、没有 `<img>`，session 里的 vcode 就不会被覆盖。用这个方式跑 90 条字典，0.06 秒出结果：
```
[*] PHPSESSID = m3lalq6km85n81fn0i1945mkpj
[*] 服务端明文下发的验证码 bf[vcode] = 6lizax
[*] 验证码复用测试: PASS_CAPTCHA_WRONG_PWD
[+] 命中！第 1 次尝试  password = 123456
[*] 耗时 0.06s
[*] 用 123456 重新登录 -> SUCCESS
```
第一刀就中，因为 `123456` 在任何弱口令字典里都排第一。
### 顺手对比
| 方案 | 优点 | 缺点 |
| --- | --- | --- |
| Burp + Drop 图片请求 | 纯手工，能看清每一步 | 手要快，容易点错 |
| 脚本 | 快、可复用、不受社区版限速 | 需要写代码，看不到"攻击过程" |
学原理用前者，出效率用后者。
## 七、我学到了什么
### 7.1 验证码要安全，必须同时满足三条
1. **机密**：答案不能以任何形式下发给客户端（Cookie、HTML 注释、隐藏字段都不行）；
2. **一次性**：校验完不管成败，立刻销毁；
3. **不可枚举**：字符集和长度要够，不能让攻击者撞。
这题把第 1 条和第 2 条同时违反了。
### 7.2 "明文 Cookie"这种错误比想象中常见
看到 `setcookie('bf[vcode]', $_SESSION['vcode'])` 时我愣了一下——这等于把门锁的钥匙挂在门上。
类似的问题在真实系统里也出现过：验证码答案放进隐藏字段、把校验结果写进可读写的 Cookie、SSO token 明文放进 URL。共同点都是**把服务端该藏着的东西交给了客户端**。
### 7.3 别把"慢"当成"失效"
这是这次最值钱的一条。
90 个请求跑了一分多钟，我下意识地认为"时间长了 session 就过期了"。但实际上：
- PHP 默认 `session.gc_maxlifetime` 是 1440 秒，90 秒远远不够；
- 真正的失效发生在 **t4**，也就是 Forward 之后的那一瞬间，跟时间毫无关系。
**错误的时间假设会让人去修错的地方。** 如果我当时真去调 session 超时配置，只会越走越偏。
正确做法是：先做对照实验，把"变量"隔离出来——固定会话、只改密码，看服务端到底在返回什么。
### 7.4 失败时先看"错误类型"，再看"响应长度"
所有响应长度一样，只能说明**所有请求得到了同一种结果**，不能说明"字典里没有正确密码"。
这题里，那 90 个 `35376` 全是「验证码输入错误」页面。如果我早点把失败页面单独提交一次、看一眼提示文字，就能立刻发现是验证码没通过，而不是密码不对。
**先确认"失败的原因是什么"，再怀疑"字典对不对"。**
### 7.5 响应头也是信息
`Cache-Control: no-store, no-cache, must-revalidate` 这个头，直接解释了"为什么浏览器一定会重新请求图片"。
以前看响应头只关注 `Set-Cookie` 和 `Content-Type`，这次意识到：缓存策略也会影响攻击面。
## 八、防御怎么写才对
```php
// 1. 验证码只存 session，绝不下发到 Cookie / 页面
$_SESSION['vcode'] = vcodex();
// 2. 校验时用严格比较
if (!isset($_SESSION['vcode']) || strtolower($_POST['vcode']) !== strtolower($_SESSION['vcode'])) {
    // 校验失败
}
// 3. 不管成败，立即销毁，一次一用
unset($_SESSION['vcode']);
// 4. 再配合失败次数限制
```
另外三条容易漏的：
| 措施 | 说明 |
| --- | --- |
| 验证码绑定会话 | 不能跨会话复用 |
| 设置有效期 | 比如 2 分钟，过期必须重新获取 |
| 失败计数 | 同一账号/IP 连续失败 N 次就锁定或加验证 |
## 九、下一关
- [x] 基于表单的暴力破解
- [x] 验证码绕过（on server）
- [ ] 验证码绕过（on client）
- [ ] token 防爆破？
下一关 `on client` 是另一类问题：验证码只在前端 JS 里校验，服务端根本不验。那就不需要跟 session 斗智斗勇了，抓包直接改就行。
## 参考
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
- [小皮面板官网](https://www.xp.cn/)
- [Burp Suite 官方文档](https://portswigger.net/burp/documentation)
---
**一句话总结**：这题的验证码有两个洞——答案明文写在 Cookie 里，校验完又不销毁。但我第一次爆破全失败，不是因为这两个洞不存在，而是因为**我在按下 Forward 的那一刻，亲手把验证码刷掉了**。搞清楚"验证码是什么时候失效的"，比搞清楚"字典里有没有正确密码"重要得多。
