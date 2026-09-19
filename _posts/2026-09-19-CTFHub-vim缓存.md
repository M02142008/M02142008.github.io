
---
layout: post
title: "CTFHub Writeup-vim缓存"
date: 2026-09-19
categories: CTF
tags: [CTFHub, Web, 技能树]
---

# CTFHub Writeup - vim缓存

## 题目类型

WEB

## 题目描述

打开题目后，点开链接，页面出现黑体加重字体：

备份文件下载 - vim

标题下面有一行话：

flag 在 index.php 源码中

## 解题过程

踩坑记录：

我首先判断出它是一道信息泄露的题目了，优先选择了我最熟悉的方法

点击Win+R搜索cmd,在黑框中输入

curl http://challenge-102968cfe1e356a7.sandbox.ctfhub.com:10800/.index.php.swp

回车后结果显示：

Warning: Binary output can mess up your terminal. Use "--output -" to tell curl to output it to your terminal anyway,
Warning: or consider "--output <FILE>" to save to a file.

理解结果意思，决定继续尝试

在黑框中输入：

curl [http://challenge-102968cfe1e356a7.sandbox.ctfhub.com:10800/.index.php.swp](http://challenge-102968cfe1e356a7.sandbox.ctfhub.com:10800/.index.php.swp) --output -

强制让全部内容打印至终端

结果再次显示curl: (3) bad range in position 2:
[http://challenge-102968cfe1e356a7.sandbox.ctfhub.com:10800/.index.php.swp](http://challenge-102968cfe1e356a7.sandbox.ctfhub.com:10800/.index.php.swp)

放弃此方法选择新的路径

新路径：

在浏览器搜索框输入：

http://challenge-102968cfe1e356a7.sandbox.ctfhub.com:10800/.index.php.swp

回车后，弹出下载页面，点击下载

下载好后，选择以记事本的方式打开文件

打开文件后在一堆乱码中就能找到正确的flag了

## 笔记

错误原因：

CMD 里 curl 不行，因为`.swp`是二进制文件，Windows 黑框 cmd 不适合展示二进制内容；浏览器直接访问会**下载文件**，把内容存成本地文件，记事本打开看就正常。

这道题主要学习了：

1.swp：Vim 编辑器的**交换缓存文件（swap file）**

它不光保存你的代码文字，还额外存一堆编辑器内部标记、光标位置、文件状态这类看不见的控制数据。
所以它天生就是**二进制格式**，不是单纯纯文本。

2.做题的时候不用提前猜是不是二进制，**看 curl 返回的提示就行**：

如果 curl 一跑，弹出警告：Binary output can mess up your terminal

这句话就是系统直接告诉你：**这是二进制文件，终端显示会出问题**。






