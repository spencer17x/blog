---
title: 在网站搭建初期给z-index 分层规范
date: 2021-12-06 22:26:01
tags: 
    - Front-end
---

一般来说前端架构师在网站搭建初期就要给 z index 分层：

* 0+ 用于普通内容
* 16+ 用于导航
* 32+ 用于导航菜单
* 64+ 用于 tooltip
* 128+ 用于 popover
* 256+ 用于对话框
* 1024+ 用于 loading  

类似这样