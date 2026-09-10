---
layout:post
title:"BugKu CTF Writeup-计算器"
date:2026-09-10
---
#BugKu CTF Writeup-计算器
##题目类型
WEB
##解题过程
进入题目之后页面出现一道加法题目，输入答案发现只能打一位数字。
尝试Fn+F12，开发者工具上会出现源代码。
在Elements面板找到input type="text" maxlength="1"这一行代码，并将maxlength=1改为maxlength=3
再将正确的算式答案输入至页面输入框，并且点验证，就能拿到正确的flag了。
##笔记
这道题主要学习了：
1. 输入框属性 maxlength="1"，是浏览器前端的限制，只限制普通用户在页面上输入。
2. 通过开发者工具（Ctrl+Shift+I）修改/删除 maxlength 属性，就可以解除输入长度限制，提交正确答案拿到flag。
3. 笔记本F12受Fn键影响，需要使用Fn和F12共同使用。
