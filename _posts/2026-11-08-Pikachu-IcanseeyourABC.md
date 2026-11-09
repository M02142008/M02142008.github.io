# 敏感信息泄露 IcanseeyourABC 通关笔记

> 菜单里叫 **IcanseeyourABC**，页面面包屑写的是 **find abc**。
>
> 页面就一个登录框，提示写着「找找看，很多地方都漏点了...」
>
> **密码不在数据库里，就在页面的 HTML 注释里。**

---

## 一、环境与目标

| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | 敏感信息泄露 → IcanseeyourABC（页内叫 find abc） |
| 登录页 | `http://pikachu/vul/infoleak/findabc.php` |
| 登录后 | `http://pikachu/vul/infoleak/abc.php` |
| 目标 | 找到可登录的账号 |

---

## 二、我的解题步骤

### 步骤 1：先随便试一次

输入 `admin` / `123456`。

页面提示：

```
您输入的账号错误
```

### 步骤 2：注意到提示会"区分"

换个存在的账号（比如 `vince`）+ 错误密码：

```
您输入的密码错误
```

**两种提示不一样！**

这意味着：**可以枚举出哪些用户名存在。**

### 步骤 3：看源码，发现注释里的账号

按 `F12` 看 Elements，或者 `Ctrl+U` 看原始 HTML。

在表单下面几行，有一个 HTML 注释：

```html
</div><!-- 测试账号:lili/123456-->
```

**账号和密码，直接写在注释里。**

我实测抓到的内容：

```
[infoleak 源码注释] lili/123456
```

### 步骤 4：用这个账号登录

输入 `lili` / `123456`。

登录成功，跳到 `abc.php`，页面显示一段王小波的《黄金时代》。

### 步骤 5：顺手验证用户名枚举

| 输入 | 提示 |
| --- | --- |
| `admin` / 随便 | 您输入的**账号**错误 |
| `lili` / 随便 | 您输入的**密码**错误 |

**能区分 = 可以批量枚举用户名。**

---

## 三、原理分析

### 3.1 漏洞一：HTML 注释泄露

源码：

```html
</div><!-- 测试账号:lili/123456-->
```

这一行来自 `findabc.php` 第 109 行：

```html
</div><!-- 测试账号:lili/123456-->
```

**开发时留下的调试信息，上线时忘了删。**

### 3.2 漏洞二：用户名枚举

```php
if(mysqli_num_rows($result)==1){
    // 登录成功
}else{
    $query_username = "select * from member where username='$username'";
    $res_user = execute($link,$query_username);
    if(mysqli_num_rows($res_user) == 1){
        $html.="<p class='notice'>您输入的密码错误</p>";   // ← 账号存在
    }else{
        $html.="<p class='notice'>您输入的账号错误</p>";   // ← 账号不存在
    }
}
```

**代码逻辑本身没错，错在"把内部判断结果告诉用户"。**

### 3.3 这两件事组合起来

```
1. 枚举出所有存在的用户名（lili、vince、allen...）
2. 对每个用户名爆破密码
3. 密码是 123456
```

**如果只有用户名枚举、没有注释泄露，攻击者还得爆破；但注释直接把答案给了。**

---

## 四、知识点与笔记

### 4.1 源码里还能藏什么

| 位置 | 常见内容 |
| --- | --- |
| HTML 注释 | 账号密码、TODO、内网地址、调试开关 |
| JS 变量 | API Key、内网 IP、接口路径 |
| 被注释掉的代码 | 旧接口、测试账号 |
| hidden 字段 | 用户 ID、角色、价格 |
| `data-*` 属性 | 后端下发的原始数据 |

### 4.2 看源码的三种方式

| 方式 | 能看到什么 |
| --- | --- |
| `Ctrl+U` | 服务器返回的原始 HTML |
| F12 → Elements | 浏览器解析后的 DOM（可能被 JS 改过） |
| F12 → Network | 完整响应（含响应头） |

**要抓"服务器原本返回了什么"，用 `Ctrl+U` 或 Network。**

### 4.3 用户名枚举的检测

| 观察点 | 说明 |
| --- | --- |
| 提示文字 | "账号不存在" vs "密码错误" |
| 状态码 | 200 vs 401 |
| 响应长度 | 明显不同 |
| 响应时间 | 存在账号时更慢（要查两次） |

**四种信号，任何一种都够用。**

### 4.4 这关提示的"很多地方都漏点了"指什么

除了注释，还能找到：

- `abc.php` 页面本身没有登录校验（直接访问也能看）
- Cookie 里 `abc[uname]` / `abc[pw]` 是明文用户名 + md5 密码

**这一关其实演示了"信息泄露是一串，不是一处"。**

---

## 五、和其他几关的区别

| 关卡 | 泄露什么 |
| --- | --- |
| **敏感信息泄露** | 页面/源码里的"现成信息" |
| 文件下载 | 服务器上的任意文件 |
| 文件包含 | 服务器上的文件内容 |
| SQL 注入 | 数据库里的数据 |

**这一关最简单——连"攻击"都不算，只是"看"。**

---

## 六、防御怎么写

### 6.1 删掉注释

**上线前统一检查：**

```bash
grep -rn "<!--" . --include="*.php" | grep -i "账号\|密码\|password\|todo"
```

### 6.2 统一登录提示

```php
// 错误：区分两种
if ($user_exists) echo "密码错误";
else echo "账号错误";

// 正确：统一
echo "账号或密码错误";
```

### 6.3 统一响应时间

即使提示一样，**查询次数不同也会导致响应时间不同**。

```php
// 无论账号是否存在，都执行一次 md5 计算和一次查询
$hash = md5($password);
$query = "select * from member where username=? and pw=?";
```

**让两种情况的开销尽量一致。**

### 6.4 上线前 Checklist

| 检查项 | 说明 |
| --- | --- |
| 注释里有没有敏感信息 | 账号、IP、TODO |
| JS 里有没有硬编码密钥 | `key`、`secret`、`token` |
| 报错信息是否详细 | 关掉 `display_errors` |
| 有没有备份文件 | `.bak`、`.zip`、`.git` |
| 响应头是否暴露版本 | 去掉 `X-Powered-By` |

---

## 七、总结

| 要点 | 内容 |
| --- | --- |
| 这关教什么 | 敏感信息可以藏在注释里 |
| 泄露点 1 | HTML 注释 `<!-- 测试账号:lili/123456-->` |
| 泄露点 2 | 用户名枚举（区分两种错误） |
| 实测 | 源码里直接抓到 `lili/123456` |
| 修复 | 删注释 + 统一提示 |

**一句话**：

> **最危险的密码，往往就写在最显眼的地方——只是没人去看源码。**

---

## 参考

- [OWASP: Information Exposure](https://owasp.org/www-community/vulnerabilities/Information_exposure)
- [OWASP: Testing for Account Enumeration](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/03-Identity_Management_Testing/04-Testing_for_Account_Enumeration_and_Guessable_User_Account)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
