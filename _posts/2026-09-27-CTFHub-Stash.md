
---
layout: post
title: "CTFHub Writeup-Stash"
date: 2026-09-16
categories: CTF
tags: [CTFHub, Web, 技能树]
---

# CTFHub Writeup - Stash 

## 题目类型

WEB

## 题目描述

打开题目，点开链接后，页面出现一个大标题：

CTFHub Git泄露-Stash

标题下方有一句简短的话：

Where is flag?

## 解题过程

首先点击Win+R搜索cmd，然后输入：

git --version python -m pip show GitHacker

确认工具存在后，回车

输入：githacker --url http://challenge-da4195344a95e3e2.sandbox.ctfhub.com:10800/.git/ --output ./source_code_stash来提取完整的.git仓库

回车后，终端快速滚动大量 INFO 级日志和少量红色的 ERROR（那是抓不到的钩子文件，正常现象）。最后一行出现 1/1 were exploited successfully，表示抓取成功，并生成了对应的文件夹。

然后为进入生成目录，输入命令：cd source_code_stash

回车后光标前的路径变成了 C:\Users\35395\source_code_stash>

为进入具体的 Git 仓库，输入的命令：cd 1988*

回车后光标前的路径变成了 C:\Users\35395\source_code_stash\19888edc30b2e28f0162c84ee9248687>，成功进入真正的 Git 仓库

直接在光标后输入git stash list为查看隐藏起来的记录

 屏幕上立刻回显 stash@{0}: WIP on master: 56646b9 add flag。这印证了 Flag 确实被藏在了暂存区里

 输入git stash pop，让栈顶的暂存记录弹出并应用到当前的工作目录上

 回车后，终端刷出一大段红色的英文，提示 Unmerged paths: deleted by us: 222552269960.txt 和 modified: 50x.html

 输入dir查看当前目录下的文件

 页面显示凭空多出了此222552269960.txt 文件

 再次输入type 222552269960.txt

 回车就得到了正确的Flag

 ## 笔记

 这道题主要学习了：

 1.Log&Stash的区别

 上一题（Log）：Flag 是正式提交（commit）过的，后来被删除了。所以要通过 git log 找历史，用 git reset --hard HEAD^ 时光倒流。
 
 这一题（Stash）：Flag 根本没被正式提交，而是被“临时藏起”了（stash）。所以 git log 找不到它，必须用 git stash list 和 git stash pop 来翻抽屉。

 2、针对不同场景选择正确工具（GitHacker vs GitHack）
 
上一题踩了 GitHack 丢失历史记录的坑，这次果断换用了 GitHacker。掌握了它不仅能抓取当前源码，还能完整恢复包括 objects、refs、logs、stash 在内的全套 Git 仓库信息，是应对复杂 .git 泄露题的绝对利器。
 
3、CMD 命令行实战技巧与避坑

学会了处理极长哈希文件夹名时，用通配符 cd 1988* 自动匹配，避免手敲错导致“系统找不到指定路径”；掌握了遇到 git 命令卡在带冒号 : 的界面时，按键盘 q 键退出的自救操作。
 
4、Git 报错的深度理解

在 git stash pop 后，终端瞬间刷出一大堆红色的 Unmerged paths、deleted by us 和 modified。学会了不能被红色字体吓倒，明白这不是报错，而是 Git 在告诉你“文件已经成功取出并修改了工作区”，克服了“红字恐惧症”。



