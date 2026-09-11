---
layout: post
title: "BugKu CTF Writeup-计算器"
date: 2026-09-10
categories: CTF
tags: [BugKu, Web, 签到题]
---

# BugKu CTF Writeup - 计算器

## 题目类型

WEB

## 题目描述

计算，正确即可获得到正确的flag。

## 解题过程

进入题目之后页面出现一道加法题目，输入答案发现只能打一位数字。

尝试 `Fn+F12`，开发者工具上会出现源代码。

在 Elements 面板找到 `input type="text" maxlength="1"` 这一行代码，并将 `maxlength=1` 改为 `maxlength=3`。

再将正确的算式答案输入至页面输入框，并且点验证，就能拿到正确的 flag 了。

## 笔记

这道题主要学习了：

1. 输入框属性 `maxlength="1"`，是浏览器前端的限制，只限制普通用户在页面上输入。
2. 通过开发者工具（`Ctrl+Shift+I`）修改/删除 `maxlength` 属性，就可以解除输入长度限制，提交正确答案拿到 flag。
3. 笔记本 F12 受 Fn 键影响，需要使用 Fn 和 F12 共同使用。
