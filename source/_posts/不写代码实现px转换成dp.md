---
layout: post
title: 不写代码，快速实现px转换成dp
date: '2016-08-21'
tags:
  - Android
  - 适配
  - 微技巧
categories: 
  - 技术
---

有很多朋友在实际的工作中，会遇到设计同事给了一张设计图，上面只有px标注的距离和尺寸。产品看到设计稿后，就拿给iOS和Android的开发，说就按这个做。iOS开发可能还好，虽然也有9种屏幕，但那毕竟是两只手数的来的。但是Android开发可能是心中无数只草泥马奔腾 + 一脸懵逼。。

其实我们只要把px转换成dp就可以了，两步走：

- 第1步：确认设计稿中的屏幕dpi是多少
- 第2步：根据dpi，将px值转换成dp值

<!-- more -->

# 方法1：不写代码，快速实现

不管哪种方法，请先记住下面这个dpi范围对照表：

- 0dpi ~ 120dpi：ldpi

- 120dpi ~ 160dpi：mdpi

- 160dpi ~ 240dpi：hdpi

- 240dpi ~ 320dpi：xhdpi

- 320dpi ~ 480dpi：xxhdpi

- 480dpi ~ 640dpi：xxxhdpi



## 第1步 - 确认手机屏幕的DPI

写代码不是万能的，假如设计师给的稿子里面用的屏幕，是我们手上没有的呢？或者我们根本不确定它这个设计稿上的屏幕是什么手机呢？？上哪儿去写代码run？

（1）网上找一下这个手机的屏幕分辨率和尺寸（如果这个手机真实存在的话。。可能有些设计师给的就是iphone的设计稿。。）。或者如果设计师可以给你的话，那就是最好的。

（2）得到了宽、高像素，用勾股定理计算出对角线的像素值，再除以屏幕尺寸就得到了屏幕的dpi，然后根据上面的表格，即可得到你手机是哪个dpi的了。

> tips：其实很多网上列出手机参数的时候，除了**主屏分辨率**，一般也会列出**屏幕像素密度**，但一般都是拿ppi作为单位的。 所以如果设计稿里面是真实存在的机器，一般可以不用自己计算了。


## 第2步 - 把PX值转换成DP值

直接上利器：[android_dp_px_calculator](http://labs.rampinteractive.co.uk/android_dp_px_calculator/)

![android_dp_px_calculator](http://ww2.sinaimg.cn/large/006y8lVagw1f70nuqvhiqj30go0dqgnb.jpg)

这个利器是一个开发者已经做好的工具，显然选择一个dpi，然后输入px值，就自动生成dp值了。反之亦可。如果能在开发之前就能搞定px转dp，心里还是比较爽的。

> 其实可以看到这个工具原理很简单：如果可以确定屏幕的dpi是**xxh-dpi**的，那么也就确定了density = 3，此时`dp = px/3`，其实是很简单的计算。我们只需要知道`density = dpi/160`。

***

# 方法2：通过写代码自己实现

还是两步走，不过都要靠写代码run。

## 第1步 - 确认手机屏幕的DPI

找到设计师给的设计稿手机型号，写几行代码，run一下。得到横纵方向的xdpi，ydpi，通常，这两个值应该接近，至少也是在上面列出的同一个dpi范围的。

```
// 这两行是从郭神的博客里看到的
float xdpi = getResources().getDisplayMetrics().xdpi;
float ydpi = getResources().getDisplayMetrics().ydpi;

// 后来发现其实有更直接的方法，具体返回值待确认
float density = getResources().getDisplayMetrics().density;
int densityDpi = getResources().getDisplayMetrics().densityDpi;
```


## 第2步 - 把PX值转换成DP值

相信很多人都写过这段代码，直接用当前运行中的手机的屏幕密度来转换px和dp。

```
public class PxUtils {

    public static int px2dp(Context context, float pxValue) {
        float scale = context.getResources().getDisplayMetrics().density;
        return (int) (pxValue / scale + 0.5f);
    }

    public static int dp2px(Context context, float dpValue) {
        float scale = context.getResources().getDisplayMetrics().density;
        return (int) (dpValue * scale + 0.5f);
    }

    public static int px2sp(Context context, float pxValue) {
        float fontScale = context.getResources().getDisplayMetrics().scaledDensity;
        return (int) (pxValue / fontScale + 0.5f);
    }

    public static int sp2px(Context context, float spValue) {
        float fontScale = context.getResources().getDisplayMetrics().scaledDensity;
        return (int) (spValue * fontScale + 0.5f);
    }

}

```

***

# 知识总结

基础知识不能丢，把一些基本概念记录一下。

- dip：Density independent pixels，设备无关像素。
- dp：等于dip。
- px：物理像素。
- dpi：dots per inch，直译就是每英寸多少个像素。常见取值为`120`，`160`，`240`，`320`，`480`，`640`，，也可以叫像素密度。
- density：屏幕密度，`density = dpi / 160px`。所以它的常见取值为`0.75`，`1`，`1.5`，`2`，`3`，`4`，对应着Android工程里面的`ldpi`，`mdpi`，`hdpi`，`xhdpi`，`xxhdpi`，`xxxhdpi`。
- 分辨率：横纵2个方向的像素点的数量，常见取值`1080x1920`这样的，表示宽度有1080个像素点，高度有1920个像素点。
- 屏幕尺寸：屏幕对角线的长度。

关于屏幕适配的问题，更多细节参考官网：[https://developer.android.com/guide/practices/screens_support.html](https://developer.android.com/guide/practices/screens_support.html)










