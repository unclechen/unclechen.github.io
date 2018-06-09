---
layout: post
title: 如何实现网页和Chrome插件之间的通信
date: '2018-06-09'
tags:
  - Chrome插件
  - 前端
  - JS
categories:
  - 技术
---

## 一、需求场景

前面我写过一篇博客[使用React.js开发Chrome插件](http://unclechen.github.io/2017/06/16/%E4%BD%BF%E7%94%A8ReactJS%E5%BC%80%E5%8F%91Chrome%E6%8F%92%E4%BB%B6/)，里面介绍了作为一个新手怎么去开发Chrome插件。这次我总结一下在开发Chrome插件中很容易遇到的一些需求，比如在网页中判断是否安装了某个Chrome插件，安装的版本是多少？或者在网页上点击右键菜单里面的某个按钮，然后执行Chrome插件的某个功能。这些需求本质上都实现了网页和Chrome之间的通信。

<!-- more -->

**这里指的通信是指用户浏览的网页和Chrome插件的通信，不是指Chrome插件中的popup.html这种页面和js的通信。**

Chrome插件中有两类js代码，一种是`“background.js”`，只要Chrome插件一启用的时候，就会被运行起来。但是这个js运行在独立的隔离环境中，完全无法干预到网页的Dom和js运行；第二种是`“content.js”`，这类js可以在指定的条件下（例如某一类域名的网页中）运行，这个js运行的时候是直接运行在用户浏览的网页环境中的，可以操作用户的Dom，但是无法操作网页中运行的其他js。具体使用这两类js需要在`manifest.json`文件中声明一些配置，请参考官方文档，此处不赘述。


## 二、实现思路

从Chrome插件里面两类js的能力出发，我们实现网页与Chrome插件通信时，有两个思路，其中第一个思路有两种方法。

### 2.1 通过content.js操作DOM实现通信

由于content.js可以操作用户的Dom，我们可以动态一个隐藏的Dom节点来作为通信的媒介。

```js
// content.js
var installNode = document.creatElement('div');
installNode.id = 'my-chrome-extension-installed';
installNode.style.display = 'none';
installNode.setAttribute('version', chrome.extension.getManifest().version); // 把版本号放到属性里
installNode.innerText=JSON.stringify({key: 'value'}); // 把通信的data放到标签的html text里面
document.body.appendChild(installNode);
```

这样，在安装了Chrome插件后，content.js就会生成这样一个Dom Node。然后只要在网页的js中去查找这个Node就可以判断是否安装了Chrome插件。

```js
// page.js，指的是用户网页里面的js
var installNode = document.getElementById(''my-chrome-extension-installed'');
if (installNode) {
  console.log('Chrome extension is installed! Here is the infomation: ' + installNode.innerText);
} else {
  console.log('Chrome extention is not installed yet...');
}
```

为了让通信真正有效率，我们还可以创建Dom Event。然后在js中通过监听事件的方式来保证消息的发送和接收。

```js
// content.js
// ...接上面的代码
// 创建一个事件，表示从Chrome发送消息给网页
var eventFromChrome = document.createEvent('Event');
eventFromChrome.initEvent('EventFromChrome', true, true);
// 修改installNode的innerText把需要发送的消息内容放在里面
installNode.innerText = JSON.stringify({type: 'HELLO', msg: 'FMVP is nothing for me'});
// 发出事件
installNode.dispatchEvent(eventFromChrome);
```

```js
// page.js
// ...接上面的代码
// 监听installNode的EventFromChrome事件
installNode.addEventListener('EventFromChrome', function() {
  var data = JSON.parse(installNode.innerText);
  console.log(data.msg);
});
```

这样就实现了`content.js`向`用户网页`发送消息的过程，反过来，可以在`page.js`中创建一个`EventFromPage`事件，然后在`content.js`中监听installNode的这个事件，实现双向通信。

实现了双向通信，就可以在用户的网页中使用各种Chrome插件的能力了。写到这里我想起Android里面也经常用到这个非常类似的技术方案，**相信Android开发者一定记得WebView里面的JS和JAVA通信，也可以采用在Dom中添加一个iframe，然后通过改变iframe的src来实现JS与JAVA通信。** 小感慨一下，技术很多时候真的是相通的有木有~

### 2.2 通过window.postMessage实现通信

到这里，我们再想一下，在`content.js`中，除了通过操作Dom这种方式，我们还有没有其他的方式实现用户网页的通信。我们知道`content.js`是运行用户的网页里面的，它们相互隔离，无法使用对方定义的变量和方法。这和我们用过的`iframe`之间的通信是不是有点像？？没错，我们可以通过`postMessage`实现用户网页和`content.js`的通信。这里我们可以把`content.js`看成一个`iframe`里面运行的js。

下面看下示例代码：

```js
// page.js
// 向content.js发送消息，注意这里并非是真正的iframe，所以我们直接拿当前的window发消息
window.postMessage({type:'MsgFromPage', msg: 'Hello, I am page.'}, '*');
```

然后在`content.js`中接收这个message。

```js
// content.js
window.addEventListener("message", funtion (event) {
  if (event.source != window) {
    return;
  }
  console.log(event.data);
}, false);
```

这样同样可以实现用户网页和Chrome插件的通信，比操作Dom的方式还要更加简洁高效一点。

### 2.3 通过background.js收发Message实现通信

我们知道Chrome插件内部通信，可以用`chrome.runtime.sendMessage`实现。那么用户网页和Chrome插件之间的通信可以吗？官方也给我们提供了这种方式，但是这里有一点必须注意，这种方法要求我们的Chrome插件的`manifest.json`文件中必须加下面的配置：

```json
"externally_connectable": {
  "matches": ["*://*.example.com/*"]
}
```

**注意：出于安全考虑，这里的`matches`配置，必须是具体的域名，不可以是通配符。**

配置好了以后，我们可以在用户网页的js代码中直接调用`chrome.runtime.sendMessage`来发送消息给Chrome插件。

```js
// page.js
var targetExtensionId = "asdljiadjasjasdasdada"; // 插件的ID
chrome.runtime.sendMessage(targetExtensionId, {type: 'MsgFromPage', msg: 'Hello, I am page~'}, function(response) {
  console.log(response);
});
```

`background.js`中这么写来接收消息：

```js
// background.js
chrome.runtime.onMessageExternal.addListener(function(request, sender, sendResponse) {
  // 可以针对sender做一些白名单检查
  // sendResponse返回响应
  if (request.type == 'MsgFromPage') {
    sendResponse({tyep: 'MsgFromChrome', msg: 'Hello, I am chrome extension~'});
  }
});
```

可以看到在`background.js`中可以使用官方的`chrome.*`API实现消息发送和接收。但是需要注意的是，并不是所有的网页中都可以调用`chrome.rumtime.*`的API，必须是在`manifest.json`中配置过才可以。因为在写代码的时候，做一些异常检查判断比较好。

**需要注意**，通过`background.js`与用户网页通信，还有一个好处就是不会受到**Chrome插件首次安装时，内容脚本content.js不会在已经加载完成了的网页中运行的问题**。内容脚本`content.js`的运行可以通过`manifest.json`清单声明，也可以通过代码动态执行，但是如果是声明式的运行，那么只能在`document_start|document_end|document_idel`等事件节点发生，然而已经加载完成了网页不会再发生这些事件，那么内容脚本也就不会在这些网页中运行了。

## 三、如何在网页中实现一键安装Chrome插件（inline install）

前面讲了几种网页和Chrome插件的通信方式，再提一个比较实用的功能，那就是可以在网页中直接弹出安装Chrome插件的对话框，然后监听安装进度，提示用户去实用安装好的Chrome插件。

这个功能主要也是依赖于官方的接入方法——[inline install](https://developer.chrome.com/webstore/inline_installation)，需要做3件事：

- 1.去谷歌的官网，认证你的网站。

出于安全和用户体验考虑，不是所有的网页都可以具有这个inline install能力，必须在谷歌的管理系统认证你的网页才可以。认证方式有很多种，我觉得比较简单的一种就是在网页的头部添加这样一行代码，这个代码是你的谷歌的后台生成的，每个网站的content值是不同的：

```html
<head>
<!-- 省略其他代码 -->
<meta name="google-site-verification" content="djasdjoasjkdasdj0821038233dsda" />
</head>
```

- 2.在你的网页头部添加Chrome插件的资源地址：

```html
<head>
<!-- 省略其他代码，这里的插件地址是随手乱打的 -->
<meta name="google-site-verification" content="djasdjoasjkdasdj0821038233dsda" />
<link rel="chrome-webstore-item" href="https://chrome.google.com/webstore/detail/dasdasdadasdsadsadsadsadasdsa">
</head>
```


- 3.调用安装插件的API：`chrome.webstore.install()`

```js
// 假设你有一个install按钮
document.getElementById('intall chrome extension').addEventListener('click', function() {
  chrome.webstore.install(undefined, function() {
    console.log('onInstalled');
  }, function(error, errorCode) {
    console.log('install failed...');
  });
  // 如果你的头部只添加了一个插件地址，那么可以直接在调用chrome.webstore.install的时候，插件地址传入 undefined。如果有多个地址的话，那么需要传入你想要用户安装的那个插件的地址。
});
```

注意，建议写代码的时候，调用`chrome.webstore.*`时，也要做一下异常检查判断。

 
## 四、小结

其实Chrome插件开发和普通的JS开发没有太大的区别，无法就是对Chrome提供的各种API进行熟悉和调用。所以只要仔细阅读官方文档，并评估清楚自己的需求，然后确定自己要使用的API是哪个就行了。下面是我在网上看到的一张关于Chrome插件通信相关的图，给大家一起分享一下。

![chrome message](https://ws3.sinaimg.cn/large/006tKfTcly1fs51vu3sw2j317s10mwpv.jpg)

还有两个官方文档的地址，也贴出来，以后如果Chrome升级发生什么变化了，记得要先看官方文档：

- [https://developer.chrome.com/extensions/messaging](https://developer.chrome.com/extensions/messaging)
- [https://developer.chrome.com/extensions/content_scripts#capabilities](https://developer.chrome.com/extensions/content_scripts)
