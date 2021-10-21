---
title: Chrome 中 post 请求体过大导请求体和响应体无法正常展示
date: 2021-10-21 21:05:44
tags:
    - 浏览器
---

# 背景

Chrome 版本: 版本 95.0.4638.54（正式版本） (arm64)

机型如下: 
* macOS Big Sur
* MacBook Pro (13-inch, M1, 2020)
* 芯片 Apple M1
* 内存 16GB

一次开发中，一个 post 请求的请求体含有多个 base64 数据，在 Chrome 中请求后遇到如下问题:
* http 响应 500 (服务端错误，忽略)
* 请求体消失了
* 响应体无法显示

在服务端 500 问题修复之后，起初以为请求体和响应体不显示的问题会导致请求体丢失导致无法正常响应，可是发现只是 Chrome 中在控制台中无法正常预览，但是请求和响应都是正常的，也有数据正常返回。

于是猜想只是并不影响请求和响应，只是无法正常在控制台进行数据预览。

# 验证猜想

```ts
// 服务端
const http = require('http');

const server = http.createServer((req, res) => {
  if (req.url === '/test' && req.method.toLowerCase() === 'post') {
    res.writeHead(200, {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Headers': 'X-Requested-With,Content-Type',
      'Access-Control-Allow-Methods': 'PUT,POST,GET,DELETE,OPTIONS'
    });
    let data = '';
    req.on('data', chunk => {
      data += chunk;
    });
    req.on('end', () => {
      res.end(JSON.stringify({
        method: req.method,
        message: 'ok',
        data: JSON.parse(data)
      }));
    })
  }
})
server.listen(8000);
```

```ts
// 前端
function main() {
  const b64 = 'xxx' // 由于 base64 太大了，这里就用伪代码代替了。
  fetch('http://localhost:8000/test', {
    method: 'POST',
    body: JSON.stringify({
      base64: Array(20).fill(b64)
    })
  }).then(response => response.json()).then(response => {
    console.log('response', response)
  })
}

main();
```

效果如下所示:

![img_0.png](/articleImgs/Chrome-中-post-请求体过大导请求体和响应体无法正常展示/img_0.png)

![img_1.png](/articleImgs/Chrome-中-post-请求体过大导请求体和响应体无法正常展示/img_1.png)

![img_2.png](/articleImgs/Chrome-中-post-请求体过大导请求体和响应体无法正常展示/img_2.png)

可见无法正常预览，请求体也没了!

当你修改 base64 的值，改为 ``base64: Array(10).fill(b64)`` 发现就可以正常预览了:

![img_3.png](/articleImgs/Chrome-中-post-请求体过大导请求体和响应体无法正常展示/img_3.png)

![img_4.png](/articleImgs/Chrome-中-post-请求体过大导请求体和响应体无法正常展示/img_4.png)

![img_5.png](/articleImgs/Chrome-中-post-请求体过大导请求体和响应体无法正常展示/img_5.png)

