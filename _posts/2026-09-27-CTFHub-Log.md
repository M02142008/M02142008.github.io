
---
layout: post
title: "CTFHub Writeup-Baby PHP"
date: 2026-09-27
categories: CTF
tags: [CTFHub, Web, 技能树]
---

# CTFHub Writeup - Log

## 题目类型

WEB

## 题目描述

打开题目，点开链接后，网页跳出一个大标题：

Where is flag?

# 解题过程

## 踩坑过程
我先选择使用的工具是GitHack ，下载解压后，在 CMD 里执行 python GitHack.py http://靶场/.git/

结果要么报 Python 语法错误，要么报错 repository not found。

后来尝试在浏览器搜索栏搜索http://challenge-d84dfecf231d342c.sandbox.ctfhub.com:10800/.git/logs/HEAD

下载文件，并且用记事本的方式打开，里面出现这一内容：

000000000000... cd421bd... commit (initial): init
cd421bd... e43e394... commit: add flag
e43e394... 21e34bf... commit: remove flag

但我依旧没有找到正确的flag

我又再次将文件内容输入进cmd,依旧显示错误进

去敲 git log 也提示 fatal: not a git repository

所以我决定尝试新路径

## 新路径 

我先重新在官网下载了 Git for Windows 安装包。安装时一路 Next

重点是在环境变量配置界面必须勾选 Git from the command line and also from 3rd-party software，否则 CMD 里用不了 git 命令

装完 Git 后，在 CMD 里敲：

git --version

成功显示 git version 2.55.0.windows.5，说明 Git 配置完毕

回车在cmd中输入：pip install GitHacker

结果报错：'pip' 不是内部或外部命令，也不是可运行的程序或批处理文件

回车，我将指令改为：python -m pip install GitHacker

结果屏幕一片空白，直接换行

直接去 Python 官网，找到 Windows installer (64-bit) 的下载链接

下载完成后双击安装包，最关键的一步来了：安装界面最下方有一个复选框 Add python.exe to PATH，必须勾选！ 如果不勾选，装完之后 CMD 依然找不到 python，等于白装

勾选之后点 Install Now，等待进度条跑完，看到 Setup was successful 后点 Close

然后关掉旧终端，重新开一个CMD

然后输入：python -m pip install GitHacker，回车

这一次屏幕疯狂滚动，大量 Downloading... 和 Collecting... 一闪而过，下载了十几个依赖包（beautifulsoup4、gitpython、requests、urllib3、smmap、gitdb 等等）
 
等待大约 1-2 分钟，屏幕停止滚动，最后出现：
Successfully installed GitHacker-1.1.10 ...
 
至此，GitHacker 终于安装成功，可以开始正式提取靶场的 .git 历史了

首先进行提取目标源码，输入命令：

githacker --url http://challenge-d84dfecf231d342c.sandbox.ctfhub.com:10800/.git/ --output ./source_code

进入生成的目录：cd source_code dir

执行后，出现了一个极长的哈希命名文件夹（780c67bb9bd04b0388fc2d4671382d0），这就是 Git 仓库的根目录

进入具体的Git仓库，输入命令：cd 780c*

查看提交历史，寻找线索：git log --oneline

屏幕上显示：

4f9f7066 (HEAD -> master...) remove flag
66d7942 add flag

在黑框中输入：git diff 66d7942

回车后，就出现了正确的Flag了

# 笔记

## 失败原因：

第一坑：工具选型错误（GitHack vs GitHacker）

原因：最开始跟着网上的老旧教程，下载了 BugScanTeam/GitHack。

为什么踩坑：这个工具用 Python 2 编写，在我装好 Python 3 的电脑上直接水土不服（语法报错），而且它只能抓取最新版本的文件快照，会把 .git 目录弄丢。

后果：我把源码抓下来后，在本地敲 git log 报错 fatal: not a git repository。没有 .git 历史记录，就永远找不到被删除的 Flag。这是导致最初失败的致命原因。 

第二坑：环境配置与 Windows 系统特性

原因 1（没有 Git）：电脑里没装 Git，直接敲 git clone 报错 'git' 不是内部或外部命令。

原因 2（没有 Python / PATH 没配好）：敲 pip install 报错 'pip' 不是内部或外部命令。

原因 3（Windows 应用商店假 Python）：输入 python 毫无反应，系统自动跳转微软应用商店；输入 python -m pip 直接换行无响应。

原因 4（忘记勾选 PATH）：重装 Python 时，第一遍忘了勾选 Add python.exe to PATH。

原因 5（没重启终端）：装完软件后，没有关掉旧的 CMD 窗口重新开一个，导致系统认不出新装的 Python 和 Git。

原因 6（网络环境）：官方源下载 Python 依赖包（GitHacker 的十几个组件）太慢，一度卡死。 

第三坑：对 Git 协议和 CTF 靶场机制理解不深

原因：以为可以直接用 git clone http://靶场/.git/ 把仓库拉下来。

为什么踩坑：CTFHub 的靶场服务器通常禁用了 Git 的智能 HTTP 协议，它只把 .git 目录当作普通的静态文件暴露在 Web 上。直接 clone 它会报错 repository '...' not found。 

第四坑：手动解压和长路径的繁琐操作

原因：在工具没配好之前，尝试用纯浏览器手工提取。访问 .git/logs/HEAD 找到了哈希，又去访问 .git/objects/xx/xxxx 下载文件。

为什么踩坑：下载下来的文件是 zlib 压缩的二进制乱码，要去在线网站解压，解压出来只有一个 tree，还要继续套娃访问新哈希，再下载、再解压。加上生成的文件夹名字是一长串 40 位哈希值（780c67..易出错

补充坑（终端分页器）：敲 git log 时，屏幕突然卡在带冒号 : 的地方，乱按键还产生乱码，一度以为电脑死机。 

## 这道题主要学习了：

1、Git 泄露漏洞的利用原理（从快照到历史）

Git 泄露不仅仅能下载到网站的最新源码（快照），更致命的是可以恢复出完整的历史提交记录。出题人经常会先提交 Flag（add flag），再将其删除（remove flag）。只要还原了完整的 .git 目录，就能通过“时光倒流”找回被删除的敏感信息。
 
2、全历史恢复工具 GitHacker 的核心用法

学会了区分 GitHack 和 GitHacker 的本质差异。前者只能抓取最新快照，会丢失 .git 目录；而 GitHacker 能够突破目录浏览限制，多线程地把整个 Git 仓库（包括 objects、logs、refs）完整克隆到本地，这是解题的关键。
 
3、CTF 中 Git 实战命令的三板斧

掌握了三个极其关键的 Git 命令配合思路：

• git log --oneline：用来快速浏览提交历史，查找 add flag 的 Commit Hash。

• git diff <commit_hash>：用来直接对比版本差异，通过回显找出被删除的 Flag。

• git reset --hard HEAD^：实现代码库的“时光倒流”，直接回退到上一个提交，让被删掉的文件重新出现。
 








