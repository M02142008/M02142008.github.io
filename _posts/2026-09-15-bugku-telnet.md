
---
layout: post
title: "BugKu CTF Writeup-telnet"
date: 2026-09-10
categories: CTF
tags: [BugKu, Web, 签到题]
---

# BugKu CTF Writeup - telnet

## 题目类型

WEB

## 题目描述

打开文件后，点击下载

## 解题过程

打开下载好的压缩包，进行解压至桌面

将解压好的文件放入https://www.toolbox365.cn/tools/pcap-viewer/此网址并上传

加载完成后，页面会出现一堆数据包列表

在此列表中带数据的PSH,ACK的目标包（我的在#24中）

点开#24，将此弹窗拉到最底下，找到Bytes-55B

点开后将里面的原始数据复制粘贴给十六进制解码器进行解码

我依旧交给AI给我解出来了正确的flag

## 笔记

这道题主要学习了：

1.Telnet是一个远程登录工具，主要走的是23号端口。

2.PSH+ACK包：才是真正带着传输文字的包，这是我们需要找的。

3.在线简易抓包查看器：https://www.toolbox365.cn/tools/pcap-viewer/，可以顺利打开页面并使用。


