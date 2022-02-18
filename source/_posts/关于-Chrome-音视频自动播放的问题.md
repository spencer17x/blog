---
title: 关于 Chrome 音视频自动播放的问题
date: 2022-02-18 11:16:17
tags:
    - Front-end
---
    
在 Chrome 音视频播放的时候，play() 返回的是一个 promise，当播放失败的时候一般会显示一个按钮让用户手动进行播放。

但是在浏览其他大型视频网站的时候，会发现他们的网站貌似没出现过该问题。于是去查阅了相关资料。

其实最主要查看 https://developer.chrome.com/blog/autoplay 即可得到答案。

总结就是 Chrome 有个自动播放的策略。

# Chrome 中的自动播放策略

改善用户体验，减少安装广告拦截器的动机，并减少数据消耗

> Chrome 66 中针对音频和视频元素推出的自动播放策略有效地阻止了 Chrome 中大约一半不需要的媒体自动播放。对于 Web Audio API，在 Chrome 71 中启动了自动播放策略。这会影响网络游戏、某些 WebRTC 应用程序和其他使用音频功能的网页。更多详细信息可以在下面的 Web Audio API 部分中找到。

Chrome 的自动播放政策在 2018 年 4 月发生了变化

## 新行为

您可能已经注意到，Web 浏览器正朝着更严格的自动播放策略发展，以改善用户体验，最大限度地减少安装广告拦截器的动机，并减少昂贵和/或受限网络上的数据消耗。这些更改旨在为用户提供更大的播放控制权，并通过合法用例使发布者受益。

Chrome 的自动播放策略很简单：

* 始终允许静音自动播放。
* 在以下情况下允许自动播放声音：
  * 用户与域进行了交互（单击、点击等）。
  * 在桌面上，用户的媒体参与指数阈值已被超过，这意味着用户之前播放过有声视频。
  * 用户已将网站添加到移动设备的主屏幕或在桌面设备上安装 PWA。
* 顶级 frames 可以将自动播放权限委托给其 iframe 以允许自动播放声音。

### 媒体参与指数

媒体参与指数 (MEI) 衡量个人在网站上消费媒体的倾向。 Chrome 的方法是每个来源的访问与重要媒体播放事件的比率：

* 媒体（音频/视频）的消耗必须大于七秒。
* 音频必须存在且未静音。
* 带有视频的选项卡处于活动状态。
* 视频大小（以像素为单位）必须大于 200x140。

Chrome 从中计算出媒体参与度得分，该得分在定期播放媒体的网站上最高。当它足够高时，媒体只允许在桌面上自动播放。

用户的 MEI 位于 about://media-engagement 内部页面。

![](https://wd.imgix.net/image/sQ51XsLqKMgSQMCZjIN0B7hlBO02/x0XKHYcLKjmDzjfxQxFg.png?auto=format&w=1600)

### 开发者开关

作为开发者，您可能希望在本地更改 Chrome 自动播放政策行为，以测试您的网站的不同用户参与度。

* 您可以使用命令行标志完全禁用自动播放策略：chrome.exe --autoplay-policy=no-user-gesture-required。这使您可以测试您的网站，就好像用户与您的网站密切互动一样，并且始终允许播放自动播放。
* 您还可以决定通过禁用 MEI 来确保永远不允许自动播放，以及是否默认为新用户自动播放具有最高总体 MEI 的站点。使用标志执行此操作：chrome.exe --disable-features=PreloadMediaEngagementData、MediaEngagementBypassAutoplayPolicies。

### iframe 委托

权限策略允许开发人员有选择地启用和禁用浏览器功能和 API。一旦源获得了自动播放权限，它就可以将该权限委托给具有自动播放权限策略的跨源 iframe。请注意，默认情况下，同源 iframe 上允许自动播放。

```html
<!-- Autoplay is allowed. -->
<iframe src="https://cross-origin.com/myvideo.html" allow="autoplay">

<!-- Autoplay and Fullscreen are allowed. -->
<iframe src="https://cross-origin.com/myvideo.html" allow="autoplay; fullscreen">
```

当自动播放的权限策略被禁用时，在没有用户手势的情况下调用 play() 将拒绝带有 NotAllowedError DOMException 的承诺。并且自动播放属性也将被忽略。

>
> 警告：较早的文章错误地建议使用不受支持的属性gesture=media。
>

### 例子

示例 1：每次用户在笔记本电脑上访问 VideoSubscriptionSite.com 时，他们都会观看电视节目或电影。由于他们的媒体参与度得分很高，因此允许自动播放。

示例 2：GlobalNewsSite.com 包含文本和视频内容。大多数用户只是偶尔访问该站点以获取文本内容并观看视频。用户的媒体参与度得分较低，因此如果用户直接从社交媒体页面或搜索导航，则不允许自动播放。

示例 3：LocalNewsSite.com 同时包含文本和视频内容。大多数人通过主页进入网站，然后点击新闻文章。由于用户与域的交互，将允许在新闻文章页面上自动播放。但是，应注意确保用户不会对自动播放内容感到惊讶。

示例 4：MyMovieReviewBlog.com 嵌入了一个带有电影预告片的 iframe，以配合评论。用户与域交互以访问博客，因此允许自动播放。但是，博客需要将该权限明确委派给 iframe，以便内容自动播放。

### Chrome 企业政策

对于自助服务终端或无人值守系统等用例，可以使用 Chrome 企业策略更改自动播放行为。查看策略列表帮助页面，了解如何设置自动播放相关的企业策略：

* AutoplayAllowed 策略控制是否允许自动播放。
* AutoplayAllowlist 策略允许您指定始终启用自动播放的 URL 模式的允许列表。

## Web 开发人员的最佳实践

### Audio/Video 元素

这是要记住的一件事：永远不要假设视频会播放，也不要在视频未实际播放时显示暂停按钮。这非常重要，我将在下面再写一次，以供那些只是浏览该帖子的人使用。

> 陷阱：不要假设视频会播放，也不要在视频未实际播放时显示暂停按钮。

您应该始终查看 play 函数返回的 Promise 以查看它是否被拒绝：

```ts
var promise = document.querySelector('video').play();

if (promise !== undefined) {
  promise.then(_ => {
    // Autoplay started!
  }).catch(error => {
    // Autoplay was prevented.
    // Show a "Play" button so that user can start playback.
  });
}
```

> 不要在没有显示任何媒体控件的情况下播放插页式广告，因为它们可能不会自动播放，用户将无法开始播放。

吸引用户的一种很酷的方法是使用静音自动播放并让他们选择取消静音。 （参见下面的示例。）一些网站已经有效地做到了这一点，包括 Facebook、Instagram、Twitter 和 YouTube。

```html
<video id="video" muted autoplay>
<button id="unmuteButton"></button>

<script>
  unmuteButton.addEventListener('click', function() {
    video.muted = false;
  });
</script>
```

触发用户激活的事件仍然需要跨浏览器一致地定义。我建议你暂时坚持“点击”。请参阅 [GitHub 问题 whatwg/html#3849](https://github.com/whatwg/html/issues/3849)。

### Web Audio

自 Chrome 71 起，自动播放就涵盖了 Web Audio API。关于它有一些事情需要了解。首先，在开始音频播放之前等待用户交互是一种很好的做法，以便用户知道正在发生的事情。例如，想想“播放”按钮或“开/关”开关。您还可以根据应用程序的流程添加“取消静音”按钮。

> 如果在文档接收到用户手势之前创建了 AudioContext，它将在“暂停”状态下创建，您需要在用户手势之后调用 resume()。

如果您在页面加载时创建 AudioContext，则必须在用户与页面交互后的某个时间（例如，在用户单击按钮后）调用 resume()。或者，如果在任何附加节点上调用 start()，AudioContext 将在用户手势之后恢复。

```ts
// Existing code unchanged.
window.onload = function() {
  var context = new AudioContext();
  // Setup all nodes
  // ...
}

// One-liner to resume playback when user interacted with the page.
document.querySelector('button').addEventListener('click', function() {
  context.resume().then(() => {
    console.log('Playback resumed successfully');
  });
});
```

您也可以仅在用户与页面交互时创建 AudioContext。

```ts
document.querySelector('button').addEventListener('click', function() {
  var context = new AudioContext();
  // Setup all nodes
  // ...
});
```

要检测浏览器是否需要用户交互来播放音频，请在创建 AudioContext.state 后检查它。如果允许播放，则应立即切换到运行。否则将被暂停。如果您监听 statechange 事件，您可以异步检测更改。

要查看示例，请查看为 https://airhorner.com 的这些自动播放策略规则修复 Web 音频播放的小型 [Pull Request](https://github.com/GoogleChromeLabs/airhorn/pull/37)。

> 您可以在 Chromium 网站上找到 Chrome 自动播放功能的摘要。

相关链接：

* https://developer.chrome.com/blog/autoplay/
* https://sites.google.com/a/chromium.org/dev/audio-video/autoplay
* https://github.com/gnipbao/iblog/issues/25
* https://www.douyu.com/cms/gong/201807/06/8146.shtml
* https://chromeenterprise.google/policies/#AutoplayAllowed
* https://chromeenterprise.google/policies/#AutoplayAllowlist
* https://developers.google.com/web/updates/2017/06/play-request-was-interrupted