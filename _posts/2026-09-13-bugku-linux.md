
---
layout: post
title: "BugKu CTF Writeup-linux"
date: 2026-09-13
categories: CTF
tags: [BugKu, Web, 签到题]
---

# BugKu CTF Writeup - linux

## 题目类型

WEB

## 题目描述

打开题目后，需要下载一个压缩包

## 解题过程

将压缩包先进行解压，将解压后的文件放置桌面

再将刚刚解压下来的文件放入extract.me中进行二次解压，并将解压出来的压缩包下载下来

打开压缩包直接进行解压，并将解压后的文件再次保存至桌面

打开文件将里面的文档flag进行重命名，改为flag.html

用浏览器的方式将文档打开，拉到最下面就能找到正确的flag了

## 笔记

这道题我学会了：

1.嵌套压缩包需要用extract.me进行二次解压

2.二进制文件不能用记事本打开

