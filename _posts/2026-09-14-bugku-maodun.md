
---
layout: post
title: "BugKu CTF Writeup-矛盾"
date: 2026-09-10
categories: CTF
tags: [BugKu, Web, 签到题]
---

# BugKu CTF Writeup - 矛盾

## 题目类型

WEB

## 题目描述

点开题目后打开链接后，屏幕上出现一串代码：

$num=$_GET['num'];
if(!is_numeric($num))
{
echo $num;
if($num==1)
echo 'flag{**********}';
}

## 解题过程

打开链接后对代码进行理解

然后在链接的最后加上?num=1abc，回车，然后就得出正确的flag了

## 笔记

这道题主要学习了：

1.只要题目源码写了 $_GET['num']，需要让你在网址后面加 ?num=xxx。

2.更改链接可以得到正确的flag。

