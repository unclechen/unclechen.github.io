---
layout: post
title: Android WebView调用JS
---

个人认为Android的WebView一直是一个比较难搞的东西，因为它需要和很多的Web开发打交道，如果以前没接触过Web相关的开发就会觉得有些不爽，但是现在越来越多的应用都是Hybrid的模式，HTML5定稿一年多，感觉也挺火，这也是以做内容为主的App非常需要的技术，所以还得多学学。

从Android4.4开始，WebView底层的实现从原来的Webkit变成了chromium，从而实现了对HTML5更好的支持，并且也和Chrome浏览器的一些特征越来越像。接触过WebView开发应该对`WebView.setWebContentsDebuggingEnabled(true)`不会陌生，正是从4.4开始的改变才使得WebView的调试变得更加方便。

只是用来展示一个网页内容还好，如果要通过WebView执行JS脚本来和Native代码做一些通信，就要小心可能会踩到各种坑了。例如onclick事件没用，用onTap又会触发两次，4.4以上只能用loadUrl的方法执行一行js代码，API17以上需要注解保证安全性等等。在这里记录一下我自己的学习心得和踩过的坑。


# Java与JS互相调用

在Android开发里面，我们说的WebView与JS互相调用，通常就是指用Java写的Native代码与JS的互相调用。所以下面我都会说Java调用JS，JS调用Java。而不是说WebView调用JS，JS调用WebView了。

## Java调用JS

* 首先在JS中定义好即将提供给Native的方法`jsFunction()`
* 然后在Java代码里，通过`WebView.loadUrl("javascript:jsFunction()"); `就可以在调用JS的方法了。

## JS调用Java

### 方法1——addJavascriptInterface：
* 首先在Java里定义一个类`JavaObject`，然后在Java中通过`WebView.addJavascriptInterface(new JavaScriptObject(), "javaObj");  `就可以在JS中创建这个类的一个实例`javaObj`对象

* 然后在JS中可以直接使用`javaObj`对象和它的方法，这样就实现了JS调用Java。

### 方法2——iframe + CustomWebViewClient：
* 通过JS动态添加一个iframe，将其src属性设置为JS需要传给Java的参数。
* 给Java的WebView set一个自定义的WebViewClient，在这个WebViewClient的shouldOverrideUrl方法里面去处理JS那个iframe传递过来的url。拿到参数，执行相应的工作。
* Java拿到JS传过来的参数，完成工作后，通过loadUrl(javascript:)的形式，把结果返回给JS。

这是一种比较Trick的方式，js在执行的过程中去给整个dom添加一个iframe，并将这个iframe设置为`display:none`，通过这个iframe去load一个url。会去触发WebViewClient的onShouldOverrideUrl()，然后在这的方法里面，我们可以决定如何处理JS传递过来的参数。由于这个url我们是自己来解析和处理的，不打算交给WebView去直接load，所以我们其实可以自己定义一个协议，例如`nought://js.msg?arg1=x&arg2=y`。然后在WebView的WebViewClient里面拿到这个`nought://`开头的url后，我们自己写Java代码处理它。

说到这里，我们首先要了解一下WebViewClient，如果你自定义一个CustomWebViewClient继承自WebViewClient，并重写里面的shouldOverrideUrlLoading()方法，然后把CustomWebViewClient的一个实例set给了你的WebView。那么就可以在shouldOverrideUrlLoading方法中将WebView里面iframe本来将要自动load的url拦截下来，并决定是否由开发者自己的Java代码处理它。那么怎么才能自行处理这个url，而不是让WebView去自动load呢？我们看看官方文档[http://developer.android.com/intl/zh-cn/guide/webapps/webview.html](http://developer.android.com/intl/zh-cn/guide/webapps/webview.html)，总得来说是下面这样的：

* CustomWebViewClient的shouldOverrideUrlLoading返回true，表示由Java处理url，WebView不用管。
* CustomWebViewClient的shouldOverrideUrlLoading返回false，表示由WebView自己处理url。

所以JS调用Java的第二种方法，就是通过CustomWebViewClient的shouldOverrideUrlLoading返回true，告诉WebView将由Java来处理这个url。Java拿到这个url以后，可以自己解析一下参数，根据参数做一下处理，然后把结果通过Java调用JS的形式返回给JS就行了。

## Java和JS互相调用的一些实践

写一个demo，实践一下上面的思路。代码大概写了一下，占个坑，晚点来补。。


# 从4.4开始WebView执行JS的一个坑

loadUrl也是可行的，但是调用的JS代码必须是单行的。所以稍微复杂一点的js脚本就要用evaluateJavaScript来执行。





