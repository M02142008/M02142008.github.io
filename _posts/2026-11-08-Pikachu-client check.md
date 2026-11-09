# 文件上传 client check 通关笔记

> 页面只允许上传 jpg / png / gif。
>
> 但这个限制是**浏览器里的 JavaScript** 做的——**跟我们发包没有任何关系。**

---

## 一、环境与目标

| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | Unsafe Fileupload → client check |
| 页面地址 | `http://pikachu/vul/unsafeupload/clientcheck.php` |
| 提交方式 | POST（multipart/form-data） |
| 目标 | 上传一个能被服务器执行的 PHP 文件 |

---

## 二、我的解题步骤

### 步骤 1：先试正常上传

选一张 png 图片，点「开始上传」。

页面显示：

```
文件上传成功
文件保存的路径为：uploads/xxx.png
```

**注意：页面把保存路径直接输出了。**

### 步骤 2：试上传 .php

把 `shell.php` 直接在浏览器里选——**选中的瞬间就弹窗了**：

```
上传的文件不符合要求，请重新选择！
```

页面还刷新了。

### 步骤 3：看源码，发现是前端校验

```html
<input class="uploadfile" type="file" name="uploadfile" onchange="checkFileExt(this.value)"/>
```

```javascript
function checkFileExt(filename) {
    var flag = false;
    var arr = ["jpg","png","gif"];
    var index = filename.lastIndexOf(".");
    var ext = filename.substr(index+1);
    for(var i=0;i<arr.length;i++) {
        if(ext == arr[i]) { flag = true; break; }
    }
    if(!flag) {
        alert("上传的文件不符合要求，请重新选择！");
        location.reload(true);
    }
}
```

**校验完全在浏览器里做**，服务端一行代码都没验。

### 步骤 4：绕过——直接发包

用 Burp 拦下正常的 png 上传请求，把文件名改成 `shell.php`，放行。

或者直接用脚本发 multipart 请求。

我实测的结果：

| 文件名 | Content-Type | 结果 |
| --- | --- | --- |
| `t9f3a.php` | `application/octet-stream` | **上传成功：`uploads/t9f3a.php`** |

### 步骤 5：访问，确认可执行

```
http://pikachu/vul/unsafeupload/uploads/t9f3a.php
```

页面输出 `UPLOAD_OK_9f3a`。

**RCE 成立。**

---

## 三、原理分析

### 3.1 服务端源码

```php
$html='';
if(isset($_POST['submit'])){
    $save_path='uploads';//指定在当前目录建立一个目录
    $upload=upload_client('uploadfile',$save_path);//调用函数
    if($upload['return']){
        $html.="<p class='notice'>文件上传成功</p><p class='notice'>文件保存的路径为：{$upload['new_path']}</p>";
    }
}
```

`upload_client()` 的实现（`inc/uploadfunction.php`）：

```php
function upload_client($key,$save_path){
    // ... 只检查了 error 码 ...
    $save_path=rtrim($save_path,'/').'/';
    if(!move_uploaded_file($_FILES[$key]['tmp_name'],$save_path.$_FILES[$key]['name'])){
        ...
    }
    $return_data['new_path']=$save_path.$_FILES[$key]['name'];
    $return_data['return']=true;
    return $return_data;
}
```

**`move_uploaded_file` 直接用原始文件名——没有类型检查、没有改名。**

> 函数名叫 `upload_client`，作者的意思是"这个函数配合前端校验用"——**但前端校验等于没有。**

### 3.2 为什么前端校验没用

```
浏览器里的 JS  →  只能约束"通过浏览器表单提交"这一种方式
攻击者的工具   →  curl / Burp / Python，完全不执行 JS
```

**前端校验的本质是"用户体验"，不是"安全边界"。**

### 3.3 三个条件凑成了 RCE

| 条件 | 这关的情况 |
| --- | --- |
| 上传任意后缀 | ✅ 没有服务端校验 |
| 保存为原名 | ✅ 用的 `$_FILES['name']` |
| 目录可访问 + 可执行 | ✅ `uploads/` 在 Web 根下 |

**三个都满足 → 上传即 RCE。**

---

## 四、知识点与笔记

### 4.1 前端校验的常见形式

| 形式 | 位置 |
| --- | --- |
| `onchange` 校验扩展名 | `<input type="file">` |
| `accept=".jpg,.png"` | 文件选择框的过滤 |
| `onsubmit` 阻止提交 | `<form>` |

**全部可以绕过：改包、禁用 JS、改 DOM。**

### 4.2 绕过方式

| 方式 | 操作 |
| --- | --- |
| **抓包改文件名** | Burp 拦截，把 `.jpg` 改回 `.php` |
| **禁用 JS** | 浏览器设置里关掉 JavaScript |
| **改 DOM** | F12 删掉 `onchange` |
| **直接发包** | 用脚本构造 multipart |

### 4.3 为什么"页面输出了保存路径"也很危险

```php
$html.="<p class='notice'>文件保存的路径为：{$upload['new_path']}</p>";
```

**攻击者不用猜路径，直接就能访问。**

生产环境应该：

- 不暴露真实路径
- 或者用随机名 + 访问控制

### 4.4 上传成功之后

```php
// 一个最小的 webshell
<?php @eval($_POST['cmd']); ?>
```

上传后用蚁剑/冰蝎连接，就能长期控制。

**（练习时注意：Pikachu 是本地靶场，代码是自己写的，别在真实环境乱传。）**

---

## 五、和其他两关的区别

| 关卡 | 校验位置 | 绕过 |
| --- | --- | --- |
| **client check** | **浏览器 JS** | **直接发包** |
| MIME type | 服务端，看 `Content-Type` | 改请求头 |
| getimagesize | 服务端，看扩展名 + 真实图片 | 图片马 + 包含 |

---

## 六、防御怎么写

### 6.1 服务端必须校验

```php
$allow = ['jpg', 'jpeg', 'png', 'gif'];
$ext = strtolower(pathinfo($_FILES['uploadfile']['name'], PATHINFO_EXTENSION));

if (!in_array($ext, $allow, true)) {
    exit('只允许上传图片');
}
```

### 6.2 重命名文件

```php
$newName = bin2hex(random_bytes(16)) . '.' . $ext;
move_uploaded_file($_FILES['uploadfile']['tmp_name'], 'uploads/' . $newName);
```

**不要用用户给的文件名——它可能带路径（`../../shell.php`）。**

### 6.3 上传目录禁止执行

```nginx
location ^~ /uploads/ {
    location ~ \.php$ {
        deny all;
    }
}
```

**这是最关键的兜底。**

### 6.4 检测真实类型

```php
$info = getimagesize($_FILES['uploadfile']['tmp_name']);
if ($info === false) {
    exit('不是有效的图片');
}
```

---

## 七、总结

| 要点 | 内容 |
| --- | --- |
| 这关教什么 | 前端校验等于没有校验 |
| 绕过 | 抓包改文件名 / 直接发包 |
| 实测 | `t9f3a.php` 上传成功且可执行 |
| 关键代码 | `move_uploaded_file(..., $save_path.$_FILES[$key]['name'])` |
| 修复 | 服务端白名单 + 重命名 + 目录禁执行 |

**一句话**：

> **"一切在前端做的安全措施都是不靠谱的"——页面提示自己都这么写了。**

---

## 参考

- [OWASP: Unrestricted File Upload](https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload)
- [PHP: move_uploaded_file](https://www.php.net/manual/zh/function.move-uploaded-file.php)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
