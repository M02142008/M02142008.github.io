
---
layout: post
title: "BugKu CTF Writeup-隐写"
date: 2026-09-15
categories: CTF
tags: [BugKu, Web, 签到题]
---

# BugKu CTF Writeup - 隐写

## 题目类型

WEB

## 题目描述

打开题目后，下载一个文件

## 解题过程

将下载好的文件进行解压至桌面，以图片的方式打开文件发现是一张照片

打开https://tools.qsnctf.com/#/misc/png_dim_fix网站，在左侧的工具栏里面找到图片操作里的PNG图片尺寸修复

将刚刚解压出来的图片放入网站中，点击旁边的修复宽高，照片就会自动生成正确的flag了

## 笔记

这道题主要学习了：

1.PNG图片IHDR头部宽高篡改，需要用特定的工具找到图片下半部分隐藏的部分。

2.我先使用的是这个网站
https://hexed.it/，感觉这个不够智能。
