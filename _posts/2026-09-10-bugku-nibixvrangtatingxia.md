
---
layout: post
title: "BugKu CTF Writeup-你必须让他停下"
date: 2026-09-10
categories: CTF
tags: [BugKu, Web, 签到题]
---

# BugKu CTF Writeup - 你必须让他停下

## 题目类型

WEB

## 题目描述

打开页面后，页面不断跳转或者刷新，文字提示你必须让它停下。因为一直在刷新所以来不及看到源代码。

## 解题过程

进入页面后直接点击Fn+F12,找到源代码。但此每次刷新出现的HTML源码都不同。

所以我们打开开发者工具找到Setting，向下滑找到并且勾选Disable JavaScript

勾选完成后点击Fn+F5刷新页面，刷到< img src="10.jpg">这一版，往下翻源码就能看到隐藏的flag。

## 笔记

这道题主要学习了：

1.没有勾选Disable JavaScript的网页源码是不固定的，需要自己设置并且勾选上，让源代码固定。

2.F5可以刷新页面
