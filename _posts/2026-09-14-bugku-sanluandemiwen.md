

---
layout: post
title: "BugKu CTF Writeup-散乱的密文"
date: 2026-09-14
categories: CTF
tags: [BugKu, Web, 签到题]
---

# BugKu CTF Writeup - 散乱的密文

## 题目类型

WEB

## 题目描述

打开题目后，描述栏会出现一串乱码：lf5{ag024c483549d7fd@@1} 一张纸条上凌乱的写着2 1 6 5 3 4

## 解题过程

通过评论区的提示，找到密码的排列规律：

分组拆开：

1. l f 5 { a g → 重组得到 f l a g { 5

2. 0 2 4 c 4 8 → 2 0 4 8 c 4

3. 3 5 4 9 d 7 → 5 3 d 7 4 9

4. f d @ @ 1 } → d f 1 } @ @

然后我直接交给AI帮我解码的

## 笔记

这道题主要学习了：

1.AI是个解码好工具。

2.密文有着它对应的顺序。



