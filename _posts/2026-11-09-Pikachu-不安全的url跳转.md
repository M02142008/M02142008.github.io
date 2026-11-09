# 不安全的URL跳转 通关笔记

> 菜单里叫 **不安全的URL跳转**。
>
> 页面问「你到底是下面哪一种类型的人」，四个选项。
>
> 但真正决定跳去哪的，是 URL 里的 `url` 参数。

---

## 一、环境与目标

| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | URL重定向 → 不安全的URL跳转 |
| 页面地址 | `http://pikachu/vul/urlredirect/urlredirect.php` |
| 提交方式 | GET |
| 参数名 | `url` |
| 目标 | 让页面跳转到外部任意地址 |

---

## 二、我的解题步骤

### 步骤 1：先看正常选项

页面上有四个链接：

```html
<a href="urlredirect.php">像春天的花一样的少年</a>
<a href="urlredirect.php">像夏天的雨一样的少年</a>
<a href="urlredirect.php?url=unsafere.php">像秋天的风一样的少年</a>
<a href="urlredirect.php?url=i">我就是我,放荡不羁的我</a>
```

点第三个，跳到 `unsafere.php`（概述页）。

**跳转地址是 URL 参数里的 `url`。**

### 步骤 2：看源码

```php
$html="";
if(isset($_GET['url']) && $_GET['url'] != null){
    $url = $_GET['url'];
    if($url == 'i'){
        $html.="<p>好的,希望你能坚持做你自己!</p>";
    }else {
        header("location:{$url}");   // ← 直接跳，没有任何校验
    }
}
```

**`header("location:{$url}")` —— 用户传什么就跳什么。**

### 步骤 3：改成外部地址

```
http://pikachu/vul/urlredirect/urlredirect.php?url=http://example.com
```

我实测：**浏览器直接跳到了 `http://example.com/`。**

### 步骤 4：伪装成"站内链接"

真正的攻击场景不是直接给 `http://evil.com`，而是伪装：

```
http://pikachu/vul/urlredirect/urlredirect.php?url=http://evil.com
```

发到聊天里：

> 你看这个靶场，有个特别有意思的页面：http://pikachu/vul/urlredirect/urlredirect.php?url=http://evil.com

**前面那一截是可信域名，很多人不会注意到后面的参数。**

---

## 三、原理分析

### 3.1 完整链路

```
用户点击带 url 参数的链接
        ↓
服务端：header("location: " . $_GET['url'])
        ↓
浏览器收到 302，跳转到攻击者指定的地址
        ↓
用户落在钓鱼网站
```

### 3.2 为什么这是"漏洞"

跳转功能本身是必要的（登录后跳回原页、支付完成跳回商家）。

**问题在于：没有限制"能跳到哪"。**

### 3.3 真实的钓鱼场景

```
1. 攻击者构造：
   https://bank.com/redirect?url=https://bank.com.evil.com/login

2. 用户看到链接开头是 bank.com，放心点击

3. 跳到 bank.com.evil.com（其实是 evil.com 的子域名）

4. 页面做得和银行一模一样，用户输入账号密码
```

**"可信域名前缀" + "仿冒页面" = 成功率极高的钓鱼。**

---

## 四、知识点与笔记

### 4.1 开放重定向的判断

| 参数名 | 常见于 |
| --- | --- |
| `url` | 通用跳转 |
| `redirect` / `redirect_uri` | 登录后跳转 |
| `next` / `return` | 登录/支付回调 |
| `callback` | OAuth |
| `continue` / `dest` | 通用 |

**看到这些参数，先改成 `http://example.com` 试试。**

### 4.2 常见绕过

| 过滤 | 绕过 payload |
| --- | --- |
| 必须以 `/` 开头 | `//evil.com`（协议相对） |
| 必须含白名单域名 | `http://evil.com/?a=trusted.com` |
| 必须含白名单域名 | `http://trusted.com.evil.com` |
| 禁止 `//` | `http:/\evil.com` |
| 禁止 `http` | `HtTp://evil.com` |
| 只校验一次 | `http://trusted.com@evil.com`（userinfo 技巧） |

### 4.3 `@` 的坑

```
http://trusted.com@evil.com
```

浏览器会：

- 把 `trusted.com` 当**用户名**
- 真正的域名是 `evil.com`

**很多"检查是否包含 trusted.com"的逻辑，都会被这个骗过。**

### 4.4 和 SSRF 的区别

| | 开放重定向 | SSRF |
| --- | --- | --- |
| 谁去访问 | **用户的浏览器** | **服务器** |
| 跳转到哪 | 外部地址 | 内网地址 |
| 危害 | 钓鱼 | 打内网 |

**一个骗用户，一个骗服务器。**

---

## 五、和其他几关的区别

| 关卡 | 参数 | 效果 |
| --- | --- | --- |
| **URL 重定向** | `url` | 浏览器跳转 |
| SSRF | `url` | **服务器**去请求 |
| 文件包含 | `filename` | 包含并执行 |

**同样是"用户传 URL"，但"谁去访问"决定了漏洞类型。**

---

## 六、防御怎么写

### 6.1 白名单（最简单）

```php
$allow = ['/home', '/user', '/help', 'https://a.example.com'];
if (!in_array($_GET['url'], $allow, true)) {
    $_GET['url'] = '/';
}
header("Location: " . $_GET['url']);
```

### 6.2 ID 映射（最推荐）

```php
$map = [
    '1' => '/home',
    '2' => '/user',
    '3' => '/help',
];
$target = $map[$_GET['to'] ?? ''] ?? '/home';
header("Location: " . $target);
```

**用户只能传 `1` `2` `3`，连 URL 都碰不到。**

### 6.3 只允许站内相对路径

```php
$url = $_GET['url'] ?? '/';

// 必须是以单个 / 开头的站内路径
if (!preg_match('#^/(?!/)#', $url)) {
    $url = '/';
}
header("Location: " . $url);
```

**`//evil.com` 会被 `(?!/)` 挡掉。**

### 6.4 中转提示页

```html
<h3>即将离开本站</h3>
<p>目标地址：http://evil.com</p>
<p><a href="...">继续</a> | <a href="/">取消</a></p>
```

**把真实地址亮出来，让用户自己判断。**

---

## 七、总结

| 要点 | 内容 |
| --- | --- |
| 这关教什么 | 跳转目标可控 = 开放重定向 |
| 关键代码 | `header("location:{$url}")` |
| payload | `?url=http://example.com` |
| 实测 | 成功跳到外部地址 |
| 修复 | 白名单 / ID 映射 |

**一句话**：

> **开放重定向的杀伤力不在"跳"，在"用你的域名做担保"。**

---

## 参考

- [OWASP: Unvalidated Redirects and Forwards](https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
