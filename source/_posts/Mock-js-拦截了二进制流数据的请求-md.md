---
title: Mock.js 拦截了二进制流数据的请求.md
date: 2021-10-17 15:38:58
tags: 
    - Front-end
---

# 背景

在一次开发中，客户提出我们提供的 SDK 无法正常使用，但是我们提供的 demo 可以正常使用。后经对比排查发现，客户的项目中引入了 Mock.js 而导致的 SDK 中对二进制资源的请求被拦截并进行了处理，导致请求的资源发生错误。

# 产生的原因

后通过在 [Mock.js](https://github.com/nuysoft/Mock) 的 repo issues 里也有搜索到 [相关 issue](https://github.com/nuysoft/Mock/issues/299)。

原因在于引入 Mock.js 发出请求后，服务器端返回的文件为二进制流，原生请求会正常的将其作为 blob 对象返回，而 Mock.js 则会将其转为字符串，Mock.js 完全重写了原生的 XMLHttpRequest。

# 如何解决

可以参考该 commit: https://github.com/Hugo-X/Mock/commit/2a6cc9ce7cfa407ce0036c187a6de1803f567884 修改 Mock.js 源码来解决这个问题。

也可以使用 [better-mock](https://github.com/lavyun/better-mock) 替代 Mock.js。

