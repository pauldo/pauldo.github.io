---
layout:     post
title:   "使用 Jekyll+github pages 搭建日志"
date:       2015-09-10 22:46:00
author:     "Paul Du"
header-img: "img/post-bg-01.jpg"
---

## 前言
转眼间从事PHP开发已经近6个年头了, 回想几年下来并没有给自己留下足迹, 甚是惭愧. 之前有打算坚持写自己的博客的念头终因种种原因没坚持下去. 今天开始, 应该要养成良好的习惯, 勤总结, 多分享! 废话不多说, 进入正题 :)

## Git
Git已经变得越来越普及, Github也成为一名有理想有抱负的有志青年必须攻占的土地, 除了托管代码, 我们还可以将日志记录在github提供的pages服务中, 享受git版本管理, 不用担心文章丢失.

## Jekyll
是由ruby语言写的静态站生成器, 根据源码生成静态文件, 它提供了许多模板和插件.

<h3>安装依赖的包</h3>
<code>
apt-get install git ruby-dev
</code>

## 安装步骤
1. 打开github.com, 创建一个仓储, 规则是 姓名.github.io 将"姓名"替换成你的用户名
1. 克隆代码到本地
1. 初始化jekyll目录
1. 运行, 看效果

