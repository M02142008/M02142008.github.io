
---
layout: post
title: "CTFHub Writeup-PHPINFO"
date: 2026-09-16
categories: CTF
tags: [CTFHub, Web, 技能树]
---

# CTFHub Writeup - PHPINFO

## 题目类型

WEB

## 题目描述

打开题目后，点开链接，页面出现大规模的表格

## 解题过程

踩坑过程：

首先点击Ctrl+F，在搜索框里搜索ctfhub和FLAG

我发现并没有找到正确的flag,我决定尝试新路径

新路径：点击Win+R,搜索cmd

将此curl命令输入黑框：

curl http://challenge-b0df483a675dcd92.sandbox.ctfhub.com:10800/phpinfo.php | findstr ctfhub

然后点击回车，页面出现一堆代码，在此代码里就能找到正确的flag了

## 笔记

错误原因：有时页面太长，浏览器有时候不会一次性加载、索引全部内容。
所以你在可视化页面按Ctrl+F搜索，会漏掉藏在下方的那一行FLAG，搜不到。

这道题主要学习了：

1. curl 干的事：直接原封不动下载全部原始HTML源码，不做任何美化、不渲染表格。
