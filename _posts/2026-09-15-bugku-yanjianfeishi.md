
---
layout: post
title: "BugKu CTF Writeup-眼见非实"
date: 2026-09-15
categories: CTF
tags: [BugKu, Web, 签到题]
---

# BugKu CTF Writeup - 眼见非实

## 题目类型

WEB

## 题目描述

打开题目后，点击下载文件

## 解题过程

先将下载好的文件进行解压至桌面

在文件资源管理器顶部的查看中勾选文件扩展名

右键此文件将眼见非实.docx改为眼见非实.zip，双击打开此文件进行二次解压至桌面

打开解压后的文件，找到里面那个word文件夹，再进去找到document.xml

用记事本的方式打开，Ctrl+F找到搜索栏，搜索flag，就能找到正确的flag了

## 笔记

这道题主要学习了：

1..docx的本质是zip压缩包，只有把文件后缀改为zip解压，就能拆开文档底层所以文件。

2.不要被代码里的Flag在这里给骗了，只在搜索栏里搜flag。



