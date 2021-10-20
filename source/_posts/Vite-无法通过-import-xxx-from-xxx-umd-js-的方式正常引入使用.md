---
title: Vite 无法通过 import xxx from 'path/xxx.umd.js' 的方式正常引入 UMD 模块使用
date: 2021-10-20 14:22:48
tags:
    - Vite
---

# 背景

首先打包出一个 umd 模块的 js 库，这里用 add.umd.js 来表示这个库吧。

然后测试发现：

* 在 create-react-app 的项目中通过 ``import add from 'path/add.umd.js'`` 方式可以正常引入并使用

* 在 Vite 的项目中(个人测试是通过 ``yarn create vite`` 选择 react-ts 模板生成) ``import add from 'path/add.umd.js'`` 无法正常导入并使用。然而通过 ``import 'path/add.umd.js'`` 却可以正常导入，使用姿势大致如下所示：

  ```tsx
  import 'path/add.umd.js';
  
  console.log(window.add);
  ```

# 原因

最初以为 create-react-app 对 webpack 进行了配置使得其能像 import ESM 模块一样去 import UMD 模块，后经测试 webpack 天生支持。

然后翻阅了 Vite 的官方文档：

**CommonJS 和 UMD 兼容性:** 开发阶段中，Vite 的开发服务器将所有代码视为原生 ES 模块。因此，Vite 必须先将作为 CommonJS 或 UMD 发布的依赖项转换为 ESM。

相关链接：

* https://cn.vitejs.dev/guide/features.html#npm-dependency-resolving-and-pre-bundling
* https://cn.vitejs.dev/guide/dep-pre-bundling.html#the-why

然后测试所得我们并没法正常导入，后通过提 issue 得到回答：

Vite 支持在 node_modules 内部转换依赖项以实现兼容性。

但是来自用户项目的文件应该始终是 ESM。

相关 issue：https://github.com/vitejs/vite/issues/4520

