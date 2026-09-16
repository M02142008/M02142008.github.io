
---
layout: post
title: "BugKu CTF Writeup-你以为是md5吗"
date: 2026-09-16
categories: CTF
tags: [BugKu, Web, 签到题]
---

# BugKu CTF Writeup - 你以为是md5吗

## 题目类型

WEB

## 题目描述

打开题目后，点击下载

## 解题过程

用记事本的方式打开题目，发现文件内容是：

bci177a7a9c7udf69c248647b4dfc6fd84o

发现内容是一条md5哈希，但文本中出现了字母d、o，对文本进行还原，最终还原为：

bc177a7a9c7df69c248647b4dfc6fd84

将其复制粘贴至解码器中进行解码

我直接交给AI帮我完成

## 笔记

这道题主要学习了：

1.md5哈希语言里没有d、o这种字母。

2.ND5输出永远只有0-9和a-f。
