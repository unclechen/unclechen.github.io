---
layout: post
title:Android混合开发之——WebView中使用原生组件替换标签元素
date: '2017-10-15'
tags:
  - Android
  - WebView
categories: 
  - 技术

---

# 一、背景

在Android混合开发中，常常会把界面渲染全部交给html，而后台数据相关的处理交给Native。然而在有些时候html无法完全满足我们在界面处理上的要求，比如像要有一个自己定制的软键盘或者在html里面播放视频，或者想要把html里面的图片替换成Native中统一封装的ImageView等等。这不，跟WebView打交道这么多年，我最近还真遇到这样的需要了，希望把html中展示的一个大图换成Native实现的播放器，这个播放器是自己封装过的，播放控制的界面和交互也全部都由Native实现。拍脑袋一想，这有点困难啊？html里面的标签怎么替换成Native组件呢？这不可能啊？难道要实现一套把html全部转成Native的框架？这岂不是得自己做一套ReactNative？

<!-- more -->

你别说我还真在万能的Github上找到一个叫[HtmlNative](https://github.com/hsllany/HtmlNative)的库，这货就真的实现了把一部分css+html转成Native，看了下它的demo，效果其实不错。但是对于我来讲还有点偏重了，因为如果一旦我们开始转换css，那么到底对css支持到一个什么样的程度呢？这种无法走到尽头的大难路，我不想走。于是我又开始看微信小程序，发现小程序大部分的组件还是WebView渲染的dom，只有几个组件入输入框，视频播放器是原生的，并且我很惊讶地发现它就是把原生组件“嵌入”到了WebView中！！！看到这里我觉得如果是把html里面的某些指定的元素替换成Native组件，是可行的，这时我开始想办法了。从界面绘制的角度，界面由一个个的View组成，每个View都应该由坐标和尺寸来描述，从而可以被摆放到正确的位置上。举个最简单的例子，我们知道ViewGroup里面的onLayout方法，当我们实现一个ViewGroup的时候，需要在onLayout方法中调用每一个子View的layout方法，并给这个方法传入left、top、right、bottom参数，这几个参数表示这个View距离父控件的左、上、右、下距离。**如果我可以把html中需要替换的元素，相对WebView控件的left、top、right、bottom参数获取，并通过js传给Native，Native再把一个原生组件盖在WebView的位置上，是不是就可以实现“原声组件嵌入WebView里？”**

# 二、思路

这里我们就以一个简单场景来做示例，比如有一个组件是包装WebView实现的，转门用于加载html格式的广告。现在需要把这个WebView里面**img标签变成一个ImageView**，思路如下：

- 1.把WebView放到一个FrameLayout里面，使用WebView加载这个html，让其中的元素都被加载、渲染完成，这时img标签的位置和尺寸才可以确定。
- 2.自定义WebViewClient，监听onPageFinished回调，当回调发生时，执行一段js，去获取指定的img标签的left、top、width、height属性，然后传给Native
- 3.Native接收到之后，把ImageView添加到第一步中的FrameLayout里面。

# 三、具体实现方案

## 1.准备html

html中一定要能清楚的获取到需要替换的img标签，例如我们可以给这个img标签加上特定的id，如下所示：

```
<!DOCTYPE html>
<html>
<head>
<title>test</title>
<meta http-equiv="content-Type" content="text/html;charset=utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, minimum-scale=1.0, maximum-scale=1.0, user-scalable=no"/>
<style type="text/css">
.container{ margin:0 auto; width:300px; overflow:hidden} 
.container img{ float:left; width:100px; height:100px} 
.container .right{ float:right; width:180px; text-align:left} 
.container .right h3{ height:20px; line-height:20px; font-family:"Microsoft YaHei"; font-size:16px; overflow:hidden;} 
.container .right div{ padding-top:0px; height:50px; overflow:hidden}
</style>
</head>
<body>
<div class="container"> 
    <img id="imageHolder" src="./img.jpeg"/> 
    <div class="right">
        <h3>我是一个标题好吗</h3> 
        <div>无论事态变迁，你总有一颗人仰马翻的少年心</div> 
    </div> 
</div>

</body>
</html>

```

这段html里面的有两个需要注意的地方：

- 需要替换的html标签img，我们给它加上了一个id叫“imageHolder”，后面我们需要通过js获取这个标签。
- viewport里面把device-width设为设备的宽度，这样我们获取到的图片位置和宽高都是dp为单位。

# 2.准备好获取img标签left、top、width、height属性的js方法，提供给Native调用

```
<script type="text/javascript">
var jsFun = {
  // 测量图片的大小和位置
  measureImagePlaceHolder: function () {
    var img = document.getElementById("imageHolder");
    var left = img.getBoundingClientRect().left + img.scrollLeft;
    var top = img.getBoundingClientRect().top + img.scrollTop;
    var width = img.getBoundingClientRect().right - left;
    var height = img.getBoundingClientRect().bottom - top;
    JavaFun.replaceImgWithImageView(left, top, width, height);
  }
}
</script>
```

这段代码的功能就是获取img标签在网页中的绝对位置和大小，我是从阮一峰老师的[博客](http://www.ruanyifeng.com/blog/2009/09/find_element_s_position_using_javascript.html)学到的，把这段js加入到html的body最后即可。

这时其实已经可以用chrome打开这个页面，进入inspect界面，手动调用一下`measureImagePlaceHolder`方法已经可以看到效果了，如下图所示。

![chrome查看js](https://ws4.sinaimg.cn/large/006tKfTcly1fkjcqk16dgj30rs082774.jpg)

# 3.在Native中把WebView放到一个FrameLayout里面，自定义WebViewClient监听onPageFinished事件。

```

    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        rootView = (FrameLayout) findViewById(R.id.root_view);
        initView();
        initWebView();
    }

    private void initWebView() {
        webView.getSettings().setJavaScriptEnabled(true);
        webView.setWebViewClient(new WebViewClient() {
            @Override
            public void onPageFinished(WebView view, String url) {
                super.onPageFinished(view, url);
                view.loadUrl("javascriprt:jsFun.measureImagePlaceHolder();");
            }
        });
    }

    private void initView() {
        webView = new WebView(this);
        imageView = new ImageView(this);
        FrameLayout.LayoutParams layoutParams = new FrameLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT);
        rootView.addView(webView, layoutParams);
    }

```

# 4.Native中提供一个给js调用的方法，用于传递需要替换的img标签的方法

```
    class JavaFun {
        @JavascriptInterface
        public void replaceImgWithImageView(int left, int top, int width, int height) {
            final Context context = MainActivity.this.getApplicationContext();
            if (imageView == null) {
                imageView = new ImageView(MainActivity.this);
            }
            final FrameLayout.LayoutParams params = new FrameLayout.LayoutParams(dp2px(context, width), dp2px(context, height));
            params.leftMargin = dp2px(context, left);
            params.topMargin = dp2px(context, top);
            new Handler(Looper.getMainLooper()).post(new Runnable() {
                @Override
                public void run() {
                    rootView.addView(imageView, params);
                    imageView.setBackgroundColor(Color.WHITE);
                    imageView.setImageDrawable(context.getResources().getDrawable(R.drawable.shepherd));
                }
            });
        }
    }

    // 此后还需要在initWebView方法添加一行。把一个名为JavaFun的对象注入js：webView.addJavascriptInterface(new JavaFun(), "JavaFun");
    // 然后在js的measureImagePlaceHolder方法后面添加一行调用Java的代码：JavaFun.replaceImgWithImageView(left, top, width, height);
```

关于Java和JS通信的方法，这里不做介绍，感兴趣的同学可以看看前面写过的博客。

我们看下两种模式下的效果，左边是html的img标签渲染图片的效果，右边是ImageView渲染图片的效果，为了明显对比，我用了两张不同的图片，打开了开发者模式的布局边界：

![two mode](https://ws2.sinaimg.cn/large/006tKfTcly1fkjcpehayoj30xc08kk08.jpg)

怎么样，是不是还可以用来其他的Native组件来替换html标签啊？哈哈，我要用我们的视频组件去替换喽。上面这个小例子的代码在[这里](https://github.com/unclechen/ReplaceElementInHtml)，仅供大家参考，更复杂的例子还需要具体情况具体分析了。

# 四、总结

在界面开发的时候，不论是Android、iOS还是html，其实我们都是在处理布局，也就是说撇开各个平台上它们自己定义的一套标准，大部分时候，我们编写界面就是在处理界面上每一个元素在这个界面的位置和这个元素自身的大小。ReactNative类的框架干得事情就是帮开发者把html里面那套布局转换到Android和iOS各自的平台，站在现在看，可能会有人会争论html什么时候统一天下。但也许将来会出现一个新标准，在各个平台上都可以执行，而不是现阶段的哪个平台去取代哪个平台这么简单。前段时间看到过一个叫[Flutter](https://flutter.io/)的东西，好像就有点这个方向的意思，感兴趣的同学可以看看去。

