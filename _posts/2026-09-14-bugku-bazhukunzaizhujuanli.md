
---
layout: post
title: "BugKu CTF Writeup-把猪困在猪圈里"
date: 2026-09-14
categories: CTF
tags: [BugKu, Web, 签到题]
---

# BugKu CTF Writeup - 把猪困在猪圈里

## 题目类型

WEB

## 题目描述

打开题目后，点击下载文件，打开文件后里面是一串代码

## 解题过程

将文件里的代码复制粘贴

在浏览器搜data:image/jpeg;base64,+刚才粘贴的代码，就会自动出现猪圈符号的照片

将照片交给AI或者解码器，就会出现正确的flag

## 笔记

这道题主要学习了：

1.Base64图片编码：txt里看着像乱码的长串字符，不是文字，是图片被转成Base64存储。可以用data:image/jpeg;base64,拼接编码，在浏览器直接还原图片，不用下载工具。

2.AI可以解码很好用。
