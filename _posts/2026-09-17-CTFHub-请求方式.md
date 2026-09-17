
---
layout: post
title: "CTFHub Writeup-请求方式"
date: 2026-09-17
categories: CTF
tags: [CTFHub, Web, 技能树]
---

# CTFHub Writeup - 请求方式

## 题目类型

WEB

## 题目描述

打开题目后，点开链接，页面显示：

HTTP Method is GET
Use CTF**B Method, I will give you flag

## 解题过程

键盘按Win+R，输入cmd，回车，弹出黑窗口

输入curl -X CTFHUB http://challenge-02a1c28d5e4b71cf.sandbox.ctfhub.com:10800/index.php，回车，改变请求方式就得到了正确的flag

## 笔记

这道题主要学习了：

1.curl是个自定义请求方式的工具。

2.HTTP协议必须要有一个动作单词。
