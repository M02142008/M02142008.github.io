
---
layout: post
title: "CTFHub Writeup-bak文件"
date: 2026-09-16
categories: CTF
tags: [CTFHub, Web, 技能树]
---

# CTFHub Writeup - bak文件 

## 题目类型

WEB

## 题目描述

打开题目，点开链接后，简单空白页面会出现一句话：

Flag in index.php source code.

## 解题过程

首先判断此题为信息泄露类型题目

点击Win+R搜索cmd，在黑色输入框中输入：

curl http://challenge-d0080a8c53488b06.sandbox.ctfhub.com:10800/index.php.bak

黑框里直接打印全部备份源码，往上翻就能找到正确的flag了

## 笔记

1.bak文件这道题是指代码备份的无意泄露。

2.curl：不去画图渲染，直接把服务器发过来那一份**原始剧本完整打印出来**，效果等价于浏览器里的「右键 — 查看网页源代码」，只是一个输出在 cmd 黑框，一个输出在浏览器标签页。
