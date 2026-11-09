# 文件上传 MIME type 通关笔记

> 这次服务端终于校验了——**但它只看了请求头里的 `Content-Type`。**
>
> 而这个头，是客户端随便填的。

---

## 一、环境与目标

| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | Unsafe Fileupload → MIME type |
| 页面地址 | `http://pikachu/vul/unsafeupload/servercheck.php` |
| 提交方式 | POST（multipart/form-data） |
| 目标 | 绕过 MIME 校验，上传 PHP 文件 |

---

## 二、我的解题步骤

### 步骤 1：先正常传个图片

选一张 png，上传成功：

```
文件上传成功
文件保存的路径为：uploads/xxx.png
```

### 步骤 2：试直接传 .php

选 `shell.php` 上传。

**这次前端没有拦截**（这关没有 JS 校验），但服务端返回：

```
上传的图片只能是jpg,jpeg,png格式的！
```

**服务端确实校验了。**

### 步骤 3：看源码，发现只看了 Content-Type

```php
$mime=array('image/jpg','image/jpeg','image/png');
$upload=upload_sick('uploadfile',$mime,$save_path);
```

`upload_sick()` 里：

```php
//验证一下MIME类型
if(!in_array($_FILES[$key]['type'], $mime)){
    $return_data['error']='上传的图片只能是jpg,jpeg,png格式的！';
    return $return_data;
}
```

**`$_FILES[$key]['type']` 是什么？**

**是 HTTP 请求里 `Content-Type` 头——客户端自己填的。**

### 步骤 4：改包

用 Burp 拦下上传请求，改成：

```
Content-Disposition: form-data; name="uploadfile"; filename="shell.php"
Content-Type: image/png
```

**只改 `Content-Type`，文件名保持 `.php`。**

放行。

### 步骤 5：结果

我实测：

| 文件名 | Content-Type | 结果 |
| --- | --- | --- |
| `t9f3b.php` | `image/png` | **上传成功：`uploads/t9f3b.php`** |
| `t9f3c.php` | `application/octet-stream` | 被拒绝 |

### 步骤 6：访问确认

```
http://pikachu/vul/unsafeupload/uploads/t9f3b.php
```

输出 `UPLOAD_OK_9f3a`。

**RCE 成立。**

---

## 三、原理分析

### 3.1 源码

`servercheck.php`：

```php
if(isset($_POST['submit'])){
    $mime=array('image/jpg','image/jpeg','image/png');//指定MIME类型,这里只是对MIME类型做了判断。
    $save_path='uploads';
    $upload=upload_sick('uploadfile',$mime,$save_path);
}
```

`upload_sick()`：

```php
//验证一下MIME类型
if(!in_array($_FILES[$key]['type'], $mime)){
    $return_data['error']='上传的图片只能是jpg,jpeg,png格式的！';
    $return_data['return']=false;
    return $return_data;
}
// ... 然后直接 move_uploaded_file，用原始文件名 ...
```

### 3.2 `$_FILES['type']` 到底是什么

PHP 文档写得很清楚：

> `$_FILES['file']['type']` —— 文件的 MIME 类型，**如果浏览器提供了这个信息的话**。

**它来自请求头，完全由客户端控制。**

```
POST /upload.php HTTP/1.1
Content-Type: multipart/form-data; boundary=...

--boundary
Content-Disposition: form-data; name="uploadfile"; filename="shell.php"
Content-Type: image/png          ← 这里，随便写
```

**这不是"检测文件内容"，是"读一个用户填的字符串"。**

### 3.3 正确的检测方式

| 方式 | 检测什么 | 可靠性 |
| --- | --- | --- |
| `$_FILES['type']` | 客户端声明的类型 | ❌ 完全不可信 |
| `finfo_file()` | **文件真实内容** | ✅ 较可靠 |
| `getimagesize()` | 文件头 + 图片结构 | ✅ 较可靠 |
| `exif_imagetype()` | 文件头魔数 | ✅ 较可靠 |

```php
$finfo = finfo_open(FILEINFO_MIME_TYPE);
$realMime = finfo_file($finfo, $_FILES['uploadfile']['tmp_name']);
// 结果可能是 image/png / text/x-php
```

**这才是"检测文件真实类型"。**

### 3.4 为什么文件名也没改

```php
if(!move_uploaded_file($_FILES[$key]['tmp_name'],$save_path.$_FILES[$key]['name'])){
```

**还是用原始文件名。** 所以 `.php` 还是 `.php`。

---

## 四、知识点与笔记

### 4.1 MIME 校验的绕过方式

| 方式 | 说明 |
| --- | --- |
| **改 `Content-Type`** | 最直接 |
| 改文件扩展名 | `.php` → `.jpg`（但这样不能执行） |
| 图片马 | 前缀加 `GIF89a` |

**只有第一种能同时满足"扩展名是 .php"和"通过 MIME 校验"。**

### 4.2 常见 MIME 值

| 扩展名 | MIME |
| --- | --- |
| `.jpg` | `image/jpeg` |
| `.png` | `image/png` |
| `.gif` | `image/gif` |
| `.php` | `application/x-httpd-php`（浏览器不会填这个） |

**浏览器的 `Content-Type` 是按扩展名猜的，所以传 `.php` 时它会给一个"看不懂"的类型。**

### 4.3 为什么这关比上一关"高级"一点

| | client check | MIME type |
| --- | --- | --- |
| 校验位置 | 浏览器 | **服务端** |
| 校验内容 | 文件名 | **Content-Type** |
| 绕过难度 | 低（直接发包） | 低（改个头） |

**"服务端校验"听起来更安全，但校验的对象选错了，一样白搭。**

---

## 五、和其他两关的区别

| 关卡 | 校验 | 绕过 |
| --- | --- | --- |
| client check | 前端 JS | 直接发包 |
| **MIME type** | **`$_FILES['type']`** | **改 `Content-Type`** |
| getimagesize | 扩展名 + MIME + 真实图片 | 图片马 + 包含 |

---

## 六、防御怎么写

### 6.1 用真实内容检测

```php
$finfo = finfo_open(FILEINFO_MIME_TYPE);
$realMime = finfo_file($finfo, $_FILES['uploadfile']['tmp_name']);
finfo_close($finfo);

$allow = ['image/jpeg', 'image/png', 'image/gif'];
if (!in_array($realMime, $allow, true)) {
    exit('只允许上传图片');
}
```

### 6.2 扩展名白名单 + 重命名

```php
$ext = strtolower(pathinfo($_FILES['uploadfile']['name'], PATHINFO_EXTENSION));
if (!in_array($ext, ['jpg', 'jpeg', 'png', 'gif'], true)) {
    exit('非法后缀');
}
$newName = bin2hex(random_bytes(16)) . '.' . $ext;
```

### 6.3 上传目录禁止执行

```nginx
location ^~ /uploads/ {
    location ~ \.php$ { deny all; }
}
```

**即使前面全被绕过，这一条也能救命。**

---

## 七、总结

| 要点 | 内容 |
| --- | --- |
| 这关教什么 | MIME 校验看的是请求头，不可信 |
| 绕过 | 改 `Content-Type: image/png` |
| 实测 | `t9f3b.php` + `image/png` → 上传成功且可执行 |
| 正确做法 | `finfo_file()` 检测真实内容 |
| 兜底 | 目录禁执行 |

**一句话**：

> **校验 `Content-Type` 等于让攻击者自己声明"我是好人"。**

---

## 参考

- [OWASP: Unrestricted File Upload](https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload)
- [PHP: finfo_file](https://www.php.net/manual/zh/function.finfo-file.php)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
