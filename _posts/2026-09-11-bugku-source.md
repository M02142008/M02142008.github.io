
---
layout: post
title: "BugKu CTF Writeup-source"
date: 2026-09-10
categories: CTF
tags: [BugKu, Web, 签到题]
---

# BugKu CTF Writeup - source

## 题目类型

WEB

## 题目描述

打开页面后显示两行英文：

Hello,world!

This is my friend :

## 解题过程

先点击F12打开开发者工具，找到源代码。发现里面有一个flag，复制粘贴后发现此flag是一个错误的迷惑项。

在链接的后面加上/.git/logs/HEAD进行查看HEAD日志。

大致理解记录过程，找到每条更改flag的哈希，将其复制下来。

找到合适的工具进行哈希猜解（我直接运用了豆包），找到正确的flag

## 笔记

这道题让我学会了：

1. 网站如果泄露.git文件夹，我们可以直接读取git历史，挖出被回退、隐藏掉的旧文件内容。

2. 概念的辨析：commit：创建一个新提交对象，存储暂存区当前全部内容，附带本次变更的描述日志。一次commit保存项目某一刻完整快照。

 reset：不是删除快照！ 仅仅切换「当前网页展示哪一张快照」。旧快照全部保留，只是前台页面切走了。

 .git/logs/HEAD：git的操作日记本，从上到下=事件从最早到最新，记录全部commit、reset历史。

 3.豆包可以进行哈希猜解。
