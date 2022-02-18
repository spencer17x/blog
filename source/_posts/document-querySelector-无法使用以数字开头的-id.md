---
title: document.querySelector() 无法使用以数字开头的 id
date: 2022-02-18 14:40:44
tags: 
    - Front-end
---

一次开发中使用 document.querySelector 发现会报错

使用姿势如下：

```ts
document.querySelector(`#{xxx}`)
```

xxx 为服务端返回的 uuid 变量，经排查发现当 xxx 是数字开头的时候就会报错。

**原因如下：**

document.querySelector(selectors) 中 selectors 规范是：

> 包含一个或多个要匹配的选择器的 DOM 字符串 DOMString。 该字符串必须是有效的CSS选择器字符串；如果不是，则引发SYNTAX_ERR异常。请参阅使用选择器定位DOM元素以获取有关选择器以及如何管理它们的更多信息。
>
> 提示:必须使用反斜杠字符转义不属于标准CSS语法的字符。 由于JavaScript也使用退格转义，因此在使用这些字符编写字符串文字时必须特别小心。 有关详细信息，请参阅Escaping special characters。

然而 document.getElementById() 就没这个问题。

相关链接:

* https://stackoverflow.com/questions/20306204/using-queryselector-with-ids-that-are-numbers