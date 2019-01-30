---
title: 常用 IDEA 插件
date: 2018-11-21 13:50:26
updated: 2018-11-21 13:50:26
categories:
    Java
tags:
    - IdeaPlugins
---

# RestfulToolkit

在之前定位方法、Controoler 时，使用 `Ctrl + Shift + F` 查找只能输入方法上面的 @RequestMapping 的 value 路径，而且如果有多个相同的方法时，或者代码里面有对应的单词时，会全部匹配出来。如：查找 UserController 的 add 方法，如果在搜索框输入 add 查找的话，可能会查找出很多信息，不便于定位。
<!-- more -->
使用这个插件，可以使用 `ctrl + \`，在弹出的搜索框输入 /user/add 直接定位到 Controoler 的 add 方法

# lombok

使用 lombok 注解，省略 get、set、toString、equals 等

# String Manipulation

转换字符串。快捷键：`Alt + M`

# Grep Console

定义控制台打印日志格式、颜色

# Alibaba java Coding Guidelines

alibaba 代码规约扫描器

# .ignore

快速生成、定义 ignore 文件

# Rainbow Brackets

修改括号颜色，便于定位同域内容

# GsonFormat

直接把 JSON 生成实体类。快捷键： `Alt + S`
