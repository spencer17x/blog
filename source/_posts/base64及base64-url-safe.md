---
title: base64 及 base64 url-safe
date: 2021-09-21 17:01:40
tags:
    - 未分类
---

# 背景

在一次开发中，服务端需要接受某个参数为 base64 编码的值，测试所得服务端所得的编码值无法与前端所传的值匹配不上，即所传的参数不是服务端所需的正确的编码值。后联调得知，服务端需要接收的是 base64 url-safe 的值，所以前端需要也对数据进行 base64 编码为 url-safe 的值。

遇到问题的相关 issue: https://github.com/brix/crypto-js/issues/252

# 问题复现

```js
// 有效的 base64 字符如下: 
// ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/=
// 因为 base64 字符串可以包含“+”、“=”和“/”字符，这些字符可能会改变数据的含义
// JavaScript 中可以通过 btoa 对字符串进行 base64 编码
const ret = btoa(data);
```

# 如何解决

```js
const ret = btoa(data).replace(/\//g, '_').replace(/\+/g, '-');
```

