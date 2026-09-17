
---
layout: post
title: "CTFHub Writeup-Cookie"
date: 2026-09-17
categories: CTF
tags: [CTFHub, Web,技能树 ]
---

# CTFHub Writeup - Cookie

## 题目类型

WEB

## 题目描述

打开题目后，点进链接后，页面会出现：

hello guest.only admin can get flag.

## 解题过程

方法一：

点击Fn+F12打开开发者工具，找到》，点开找到应用

在应用中找到cookie然后点开，找到admin,把0那一项改为1

回车，然后点击Fn+F5刷新，找到新的Flag

方法二：

点击Win+R,搜索cmd

在黑框中输入curl --cookie "admin=1" http://challenge-0dd11b0229d5e0ec.sandbox.ctfhub.com:10800

回车，然后自动输出正确Flag

## 笔记

这道题主要学习了：

1.Cookie是什么

Cookie就是网站存在你浏览器里的一张身份小纸条。每次访问网站，浏览器会自动把这张纸条发给服务器。服务器看纸条，来判断你是谁。
这题纸条内容：admin=0，意思是你是普通访客。







