
---
layout: post
title: "CTFHub Writeup-SVN泄露"
date: 2026-09-27
categories: CTF
tags: [CTFHub, Web, 技能树]
---

# CTFHub Writeup - SVN泄露 

## 题目类型

WEB

## 题目描述

打开题目，点开链接后，页面出现一个大标题：

信息泄露 - Subversion

标题下方有一行字：

Flag 在服务端旧版本的源代码中

## 解题过程

开启靶场环境后，直接在浏览器访问目标地址拼接 /.svn/entries，确认存在 SVN 泄露漏洞。
 
直接访问 http://challenge-8a95da6b8afe669f.sandbox.ctfhub.com:10800/.svn/wc.db，成功下载 wc.db 文件
 
将下载的 wc.db 拖入在线 SQLite 查看器（如 sqliteviewer.app）

首先查看 NODES 表：发现可疑文件 flag_2000515978.txt（记录中的 presence 字段为 not-present，说明该文件在工作区已被删除，直接访问会 404）

再次查看 PRISTINE 表：找到 size 为 33 字节（恰好是一个 Flag 的长度）的那一行记录，提取其 checksum 哈希值：

$sha1$ff6fc4f097738f63ebe395ac791097a1bfc0ea5f

手工拼接下载链接

去掉 $sha1$ 前缀后，前两位是 ff，剩余部分为 6fc4f097738f63ebe395ac791097a1bfc0ea5f。
 
构造出的完整 URL 为：

http://challenge-8a95da6b8afe669f.sandbox.ctfhub.com:10800/.svn/pristine/ff/6fc4f097738f63ebe395ac791097a1bfc0ea5f.svn-base
 
在浏览器访问上述 URL，并同步使用 CMD 的 curl -v 命令进行多次验证，服务器均返回 HTTP/1.1 404 Not Found（Server: openresty）

更换浏览器，回车，出现了正确的flag

## 笔记

这道题主要学习了：

1.SVN 原始文件的访问规则是：/.svn/pristine/ + 哈希前两位 + / + 剩余字符 + .svn-base。

2.SVN 泄露的核心原理与利用流程
理解了 SVN 和 Git 一样，都是版本控制工具，如果配置不当，.svn 目录会直接暴露在 Web 服务器上。

学会了完整的利用链路：访问 /.svn/wc.db 下载核心数据库 → 用 SQLite 工具打开 → 在 NODES 表中找到目标文件名 → 在 PRISTINE 表中提取哈希值 → 拼接 .svn-base 链接下载原始文件。

3.3、用在线 SQLite 工具替代本地软件的高效方案
放弃了下载和安装 DB Browser for SQLite 的繁琐流程，改用 sqliteviewer.app 等纯网页工具，直接拖拽 wc.db 就能可视化查看数据表。这个技巧以后遇到任何数据库泄露题都能直接复用。

