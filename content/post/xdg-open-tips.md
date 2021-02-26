---
title: "禁用烦人的网页弹框 Xdg Open"
date: 2021-01-26T12:14:58+08:00
lastMod: 2021-01-26T12:14:58+08:00
tags: [tips, manjaro, chrome, xdg-open]
comments: true
description: 禁用 chrome 中烦人的网页弹框
images:
    - /media/xdg-open-tips/popup.png
---

打开一个网页，它还想打开关联的桌面应用，有时候是合理的，比如 telegram 的 channel 分享链接，通过网页弹框打开 telegram 还蛮方便，可以直接加入 channel，但是另外一些时候（大部分时候），它就跟弹着玩儿似的，也并打不开什么应用，就很烦，所以我们要无情地把它禁用掉：不能惯着！

一个烦人的例子：

{{< figure src="/media/xdg-open-tips/popup.png" alt="烦人的弹出框" caption="烦人的弹出框" >}}

因为日常使用浏览器是 chrome，解决方案也是针对 chrome 的。

首先，确定是想要打开什么鬼应用，查看网页源码，在 `<head>` 标签中找到可疑 script 引用：

```html
<script src="https://analytics.snssdk.com/meteor.js/v1/1680583551421512/sdk"></script>
<script type="text/javascript" src="https://res.wx.qq.com/open/js/jweixin-1.3.2.js"></script>
<script type="text/javascript" src="https://g.alicdn.com/dingding/dingtalk-jsapi/2.7.13/dingtalk.open.js"></script>
```

然后，在新窗口中打开这些链接，看看是不是它们打算打开本地应用：

{{< figure class="zoom-img" src="/media/xdg-open-tips/bytedance.png" alt="抓到一个" caption="抓到一个" >}}

抓到了，这种好像私有协议的东西就很可疑，先记下来，然后接着往后看：

{{< figure src="/media/xdg-open-tips/wx.png" alt="又抓到一个" caption="又抓到一个" >}}

`dingtalk.open.js` 里倒是没有发现打开本地应用的代码，可能是兼容性比较好？

好，接下来进行屏蔽就好了，新建一个文件夹（注意，从此处开始，是只针对 chrome 的解决方案）：

```sh
sudo mkdir -p /etc/opt/chrome/policies/managed/
```

新建文件 

```sh
sudo touch /etc/opt/chrome/policies/managed/whitelist.json
```

把刚刚发现的两个私有协议放进新建的文件中：

```json
{ "URLWhitelist": ["wxlocalresource://*", "bytedance://*"] }
```

看一下：

```sh
cat /etc/opt/chrome/policies/managed/whitelist.json
{ "URLWhitelist": ["wxlocalresource://*", "bytedance://*"] }
```

OK，关掉之前页面再打开下，弹框消失，完成。

参考：

[1]: [How to prevent popping-up xdg-open dialogue from Ubuntu chrome while opening specific link?](https://stackoverflow.com/a/62942891)