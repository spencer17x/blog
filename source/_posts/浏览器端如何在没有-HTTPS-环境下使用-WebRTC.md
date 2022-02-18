---
title: 浏览器端如何在没有 HTTPS 环境下使用 WebRTC
date: 2022-02-18 12:54:50
tags: - webRTC
---

需求：两台pc设备通过rtc来建立连接进行音视频通话。

相关issue：https://github.com/pili-engineering/QNRTC-Web/issues/54

电脑系统及版本：macOS Catalina 10.15.7 (19H15)
当前浏览器版本：版本 90.0.4430.85（正式版本） (x86_64)

# 方案1：
参考：https://blog.csdn.net/qq_40556950/article/details/105048931 方法一。

注：记得添加端口号，不然无效的。

# 方案2：
参考：https://juejin.cn/post/6844903848750874632。

首先本地ip使用自签的ssl证书成功了，但是因为接口走http的，会报错：......was loaded over HTTPS, but requested an insecure XMLHttpRequest endpoint。
即另外一个问题：https://stackoverflow.com/questions/37387711/page-loaded-over-https-but-requested-an-insecure-xmlhttprequest-endpoint。

其次，即使本地ip配置成功了ssl证书，如果要实现局域网内其他设备可访问，还得去修改路由器的hosts。

