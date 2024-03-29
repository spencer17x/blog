---
title: 记一次monorepo项目由yarn workspace+lerna迁移至pnpm
date: 2022-02-21 10:52:40
tags:
    - Front-end
---

首先来说一下 npm、yarn、pnpm 的区别

# npm vs yarn vs pnpm 

## npm

在 npm3 之前，安装依赖包时采用简单的递归安装方法。而在npm3及之后，采用的是模块扁平化安装。

## yarn(摘自https://juejin.cn/post/6964006257715838984)

yarn 出生之后，解决了历史上 npm 的某些不足，比如 npm 缺乏对于依赖的完整性和一致性保障，以及 npm 安装速度过慢的问题等，尽管 npm 发展至今，已经在很多方面向 yarn 看齐，但 yarn 的安装理念仍然需要我们关注。 yarn 提出的安装理念很好的解决了当时 npm 的依赖管理问题：

* 确定性。通过 yarn.lock 等机制，保证了确定性，这里的确定性包括但不限于明确的依赖版本、明确的依赖安装结构等。即在任何机器和环境下，都可以以相同的方式被安装。
* 模块扁平化安装。将依赖包的不同版本，按照一定策略，归结为单个版本，以避免创建多个副本造成冗余。（npm 也有相同的优化)
* 更好的网络性能。Yarn 采用了请求排队的理念，类似并发连接池，能够更好地利用网络资源；同时引入了更好的安装失败时的重试机制。（npm 较早的版本是顺序下载，当第一个包完全下载完成后，才会将下载控制权交给下一个包)
* 引入缓存机制，实现离线策略。（npm 也有类似的优化)

### yarn.lock 文件结构

以 react 等依赖为例，先大致了解一下 yarn.lock 文件的结构以及确定依赖版本的方式：

```ts
# THIS IS AN AUTOGENERATED FILE. DO NOT EDIT THIS FILE DIRECTLY. 
# yarn lockfile v1 
expect-jsx@^5.0.0: 
 version "5.0.0" 
 resolved "[http://registry.npmjs.org/expect-jsx/-/expect-jsx-5.0.0.tgz#61761b43365f285a80eb280c785e0783bbe362c7](http://registry.npmjs.org/expect-jsx/-/expect-jsx-5.0.0.tgz#61761b43365f285a80eb280c785e0783bbe362c7)" 
 integrity sha1-YXYbQzZfKFqA6ygMeF4Hg7vjYsc= 
 dependencies: 
 collapse-white-space "^1.0.0" 
react "^16.0.0"
 react-element-to-jsx-string "^13.0.0" 
react-rater@^6.0.0: 
 version "6.0.0" 
 resolved "[http://registry.npmjs.org/react-rater/-/react-rater-6.0.0.tgz#2e666b6e5e5c33b622541df6a7124f6c99606927](http://registry.npmjs.org/react-rater/-/react-rater-6.0.0.tgz#2e666b6e5e5c33b622541df6a7124f6c99606927)" 
 integrity sha512-NP1+rEeL3LyJqA5xF7U2fSHpISMcVeMgbQ0u/P1WmayiHccI7Ixx5GohygmJY82g7SxdJnIun2OOB6z8WTExmg== 
 dependencies: 
 prop-types "^15.7.2" 
react "^16.8.0"
 react-dom "^16.8.0" 
//一或多个具有相同版本范围的依赖声明，确定一个可用的版本。这就是 lockfile 的确定性。
react@^16.0.0, react@^16.8.0:
version "16.14.0"
 resolved "[http://registry.npmjs.org/react/-/react-16.14.0.tgz#94d776ddd0aaa37da3eda8fc5b6b18a4c9a3114d](http://registry.npmjs.org/react/-/react-16.14.0.tgz#94d776ddd0aaa37da3eda8fc5b6b18a4c9a3114d)" 
 integrity sha512-0X2CImDkJGApiAlcf0ODKIneSwBPhqJawOa5wCtKbu7ZECrmS26NvtSILynQ66cgkT/RJ4LidJOc3bUESwmU8g== 
 dependencies: 
 loose-envify "^1.1.0" 
 object-assign "^4.1.1" 
 prop-types "^15.6.2" 
//如果同一个依赖存在多个版本，那么最高版本安装在顶层目录，即 node_modules 目录。
react@^17.0.1:
version "17.0.2"
 resolved "[http://registry.npmjs.org/react/-/react-17.0.2.tgz#d0b5cc516d29eb3eee383f75b62864cfb6800037](http://registry.npmjs.org/react/-/react-17.0.2.tgz#d0b5cc516d29eb3eee383f75b62864cfb6800037)" 
 integrity sha512-gnhPt75i/dq/z3/6q/0asP78D0u592D5L1pd7M8P+dck6Fu/jJeL6iVVK23fptSUZj8Vjf++7wXA8UNclGQcbA== 
 dependencies: 
 loose-envify "^1.1.0" 
 object-assign "^4.1.1"
```

从上面依赖版本描述的信息中，可以确定以下几点：

1. 所有依赖，不管是项目声明的依赖，还是依赖的依赖，都是扁平化管理。
2. 依赖的版本是由所有依赖的版本声明范围确定的，具备相同版本声明范围的依赖归结为一类，确定一个该范围下的依赖版本。如果同一个依赖多个版本共存，那么会并列归类。
3. 每个依赖确定的版本中，是由以下几项构成：
   1. 多个依赖的声明版本，且符合 semver 规范；
   2. 确定的版本号 version 字段；
   3. 版本的完整性验证字段
   4. 依赖列表
4. 相比 npm，Yarn 一个显著区别是 yarn.lock 中子依赖的版本号不是固定版本。 也就是说单独一个 yarn.lock 确定不了 node_modules 目录结构，还需要和 package.json 文件进行配合。

## pnpm

采用软链+硬链的方式进行包管理，使用 npm 或 Yarn Classic 安装依赖项时，所有包都被提升到模块目录的根目录。 因此，项目可以访问到未被添加进当前项目的依赖。

默认情况下，pnpm 使用软链的方式将项目的直接依赖添加进模块文件夹的根目录，而通过硬链为你节省了磁盘空间并提升安装速度。

# 为什么使用 pnpm 

* 节省了磁盘空间
* 解决了扁平化依赖树带来的问题：
  * 模块可以访问它们不依赖的包
  * 扁平化依赖树的算法非常复杂
  * 一些包必须复制到一个项目的 node_modules 文件夹中
* 安全。与 Yarn 一样，pnpm 有一个特殊文件，其中包含所有已安装包的校验和，以在执行每个已安装包的代码之前验证其完整性。
* 离线模式。 pnpm 将所有下载的包压缩包保存在本地注册表镜像中。当包在本地可用时，它从不发出请求。使用 --offline 参数，可以完全禁止 HTTP 请求。
* 速度。 pnpm 不仅比 npm 快，而且比 Yarn 快。无论是冷缓存还是热缓存，它都比 Yarn 快。 Yarn 从缓存中复制文件，而 pnpm 只是从全局存储中链接它们。

# 迁移

yarn 迁移至 pnpm 很简单，官网也有提供可能遇到的问题以及解决方案。

首先可以通过 [pnpm import](https://pnpm.io/zh/cli/import) 从另一个软件包管理器的 lock 文件生成 pnpm-lock.yaml。 支持的源文件：

* package-lock.json
* npm-shrinkwrap.json(类似package-lock.json)
* yarn.lock (v6.14.0 起)

请注意，如果您有要为其导入依赖项的工作区，那么在导入之前，您需要在 [pnpm-workspace.yaml](https://pnpm.io/zh/pnpm-workspace_yaml) 文件中声明它们。

## 遇到的问题

在迁移的过程中，发现有子 package 打包失败。查看了下依赖，发现导致打包的原因是：

该 package 有个 npm package，我们暂时就叫它 npm-packageA。然后 npm-packageA 依赖 npm-packageB，而 npm-packageB 被提升到了根目录下的 node_modules/.pnpm 里，所以导致了打包失败。

解决方案：根目录下添加 .npmrc 文件，并输入 ```shamefully-hoist=true``` 即可解决。

## shamefully-hoist vs node-linker vs public-hoist-pattern

摘自群里大佬的讨论：

目前使用 pnpm 来作为 electorn 项目的库管理工具时，在打包后运行，会报错找不到模块，目前的解决方案有 三种：
* 移除 node_module，把依赖打成 vendor.js
* 使用 pnpm 的 node-linker = hoisted
* 使用 pnpm 的 public-hoist-pattern = * 或者 shamefully-hoist = true

其中 2 和 3 的区别为:
node-linker = hoisted 是真正意义上的 和 yarn 的管理方式一样
public-hoist-pattern = * 或者 shamefully-hoist = true 的方式还是基于 pnpm 的管理方式，只是在 node_modules 层里，做了一次软链接

### 对比 shamefully-hoist 和 node-linker

**shamefully-hoist=true:**

![img.png](/articleImgs/记一次monorepo项目由yarn-workspace-lerna迁移至pnpm/img.png)

![img_1.png](/articleImgs/记一次monorepo项目由yarn-workspace-lerna迁移至pnpm/img_1.png)

**node-linker=hoisted**

![img_2.png](/articleImgs/记一次monorepo项目由yarn-workspace-lerna迁移至pnpm/img_2.png)

参考资料:
* https://juejin.cn/post/7003593666581233694#heading-7
* https://juejin.cn/post/6964006257715838984
* https://github.com/worldzhao/blog/issues/9
* https://pnpm.io/zh/blog/2020/05/27/flat-node-modules-is-not-the-only-way
* https://pnpm.io/zh/symlinked-node-modules-structure
* https://www.kochan.io/nodejs/why-should-we-use-pnpm.html