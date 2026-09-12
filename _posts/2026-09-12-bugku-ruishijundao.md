
---
layout: post
title: "BugKu CTF Writeup-瑞士军刀"
date: 2026-09-10
categories: CTF
tags: [BugKu, Web, 签到题]
---

# BugKu CTF Writeup - 瑞士军刀

## 题目类型

WEB

## 题目描述

启动场景后会出现nc 160.202.254.160 15363，并发现这个nc ip端口无法进入网络页面

## 解题过程

进入题目后发现这是一条Netcat，直接按下win+R，并输入cmd打开黑框命令提示符

然后输入 powershell 回车，进入 PowerShell后，输入一串代码：


$c=New-Object System.Net.Sockets.TcpClient('160.202.254.160',15363);$s=$c.GetStream();$w=New-Object System.IO.StreamWriter($s);$r=New-Object System.IO.StreamReader($s);$w.AutoFlush=$true;while($true){while($s.DataAvailable){Write-Host $r.ReadLine()};$cmd=Read-Host;$w.Write($cmd+[char]10)}

连上之后，输入ls回车，再输入cat flag后，返回了一个bin

再直接输入cd bin，回车后再次输入ls，回车后就得到了正确的flag

## 笔记

这道题主要学习了：

1.nc 160.202.254.160 15363这个不是一个网址，它是一个专门用来直接连接服务器端口的工具。

2. 连上之后，有一套 Linux 基础命令：

 ls 看当前目录有什么文件

 cd 目录名 进文件夹

 cat 文件名 看文件内容

 不知道 flag 在哪就一层一层翻，先 ls，进去了再 ls

