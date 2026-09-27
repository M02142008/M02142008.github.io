
---
layout: post
title: "CTFHub Writeup-Index"
date: 2026-09-27
categories: CTF
tags: [CTFHub, Web, 技能树]
---

# CTFHub Writeup - Index

## 题目类型

WEB

## 题目描述

打开题目后，点开链接，页面出现一个大标题：

Where is flag?

## 解题过程

点击Win+R,搜索cmd

打开后在黑框中输入命令去提取完整.git仓库：githacker --url http://challenge-xxx.sandbox.ctfhub.com:10800/.git/ --output ./source_code_index

回车后，屏幕快速滚动大量 INFO 级日志，夹杂着一些红色的 ERROR（那是抓不到的钩子文件，正常现象）。最后一行显示 1/1 were exploited successfully，表示抓取成功。

在光标后cd source_code_index ，点击回车

光标前的路径变成了 C:\Users\35395\source_code_index\1156...>

然后在光标前输入cd 1156*后，回车

再次输入git status为了查看工作区和暂存区的状态，回车

屏幕列出 Changes not staged for commit:，里面出现了 modified: 231333071225574.txt 和 50x.html、index.html。那个随机数字命名的 .txt 文件就是我们的目标

再次输入命令git checkout-index -a，将暂存区的文件强制恢复至工作目录

回车后终端可能会提示 already exists, no checkout，或者没有任何反应

再次输入命令：dir 然后再 type *.txt

回车后就出现了正确的Flag

## 笔记

这道题主要学习了：

1.Log&Stash&Index的区别

Git 泄露 - Log（日志题）

• Flag 的藏身之处：被正式提交（git commit）到分支里，后来又删除了。

• 特征：git log 能看到 add flag 和 remove flag 的记录。

• 关键命令：git reset --hard HEAD^（时光倒流）或 git diff（对比差异）。

Git 泄露 - Stash（暂存栈题）

• Flag 的藏身之处：被 git stash 临时藏到了“抽屉”里，没有提交（commit）。

• 特征：git log 看不到，因为它不在提交历史里。

• 关键命令：git stash list（看抽屉里有什么）和 git stash pop（把东西弹出来）。 

Git 泄露 - Index（暂存区题）

• Flag 的藏身之处：用 git add 加到了暂存区（Index），但连 stash 都没用，也没 commit。

• 特征：git log 和 git stash 都找不到它，它卡在“待提交”的索引里。

• 关键命令：git status（看出端倪）和 git checkout-index -a（从暂存区强制拉出文件）。

2.Git Index（暂存区）泄露的漏洞原理与利用

深刻理解了 Git 的工作流程：开发者执行 git add 把文件加入暂存区，但没有执行 git commit 正式提交。

此时 Flag 就卡在 Index 里。学会了用 git checkout-index -a 命令，强制将暂存区中的文件恢复出来。

3、git status 在漏洞侦查中的关键作用

在面对不知道 Flag 藏在哪里的窘境时，学会了第一时间使用 git status。

它能清晰列出工作区和暂存区被修改过的所有文件，帮助我们精准锁定那个随机数字命名的 .txt 文件。

4.CMD 通配符（*）的巧妙避坑

再次巩固了对付长哈希文件夹名的实战技巧。

面对 11567f3700cf... 这样的目录，用 cd 1156* 让系统自动匹配，不仅输得快，还彻底杜绝了把 0 和 O 混淆导致“系统找不到路径”的问题。




