
---
layout: post
title: "BugKu CTF Writeup-POST"
date: 2026-09-10
categories: CTF
tags: [BugKu, Web, 签到题]
---

# BugKu CTF Writeup - POST

## 题目类型

WEB

## 题目描述

进入网页以后是一串代码：

$what=$_POST['what'];
echo $what;
if($what=='flag'){
echo 'flag{****}';
}

## 解题步骤

进入网页后，先进行理解代码，然后点击F12,找到开发者工具。

在开发者工具中找到控制台，并在里面打入代码：

document.body.innerHTML += `<form method="post" action="http://160.202.254.160:18395"><input name="what" value="flag"><button type="submit">提交</button></form>`

点击回车键，再点击flag旁边的提交，则会出现正确的代码

## 笔记

1.POST的参数在body内部，无法直接想GET那样在网址后面输入内容。则需把数据塞进请求的信封里发送给服务器

2.在输入代码后，需注意到flag旁边的提交键，并点击
