
---
layout: post
title: "BugKu CTF Writeup-备份是个好习惯"
date: 2026-09-10
categories: CTF
tags: [BugKu, Web, 签到题]
---

# BugKu CTF Writeup - 备份是个好习惯

## 题目类型

WEB

## 题目描述

打开题目后，点开链接后，屏幕上出现一串代码：

d41d8cd98f00b204e9800998ecf8427ed41d8cd98f00b204e9800998ecf8427e

## 解题过程

打开链接后，理解代码意思。在链接后面加上 /index.php.bak，回车

页面自动弹出下载页面，点击下载文件，并以记事本的方法打开

理解记事本里代码，在浏览器的搜索框中搜索http://160.202.254.160:13877/?kkeyey1[]=1&kkeyey2[]=2，就会自动出现正确的flag

## 笔记

这道题主要学习了：

1.网站有时候会把源代码存成index.php.bak备份文件。

2.程序会自动过滤掉key，所以要写成kkeyey1，这样变量就不会直接消失了。

3.md5是一个生成乱码的工具。





