---
title: Mac 微信4.0多开
date: 2026-02-28 14:57:15
tags:
  - Mac
---

命令如下： 

1、复制微信应用

```shell
$ sudo cp -R /Applications/WeChat.app /Applications/WeChat2.app 
```

2、修改 Bundle ID

```shell
$ sudo /usr/libexec/PlistBuddy -c "Set :CFBundleIdentifier com.tencent.xinWeChat2" /Applications/WeChat2.app/Contents/Info.plist 
```

3、重新签名应用

```shell
$ sudo codesign --force --deep --sign - /Applications/WeChat2.app
```
