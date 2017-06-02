---
layout: post
title: Android6.0运行时权限简介
date: '2016-03-05'
tags:
  - Android
  - 适配
  - 权限
categories: 
  - 技术
---

Android6.0发布距离现在快1年了，虽然它市场占有率仍在龟速上升中，但还是有一些App开发者已经在打包应用时将`targetSDKVersion`设置到了`23`，也就是说把App适配到了Android6.0。以前调用Android系统中需要声明权限的API时，只需要在`AndroidManifest.xml`文件中一次性列出来。但是如果在`build.gradle`文件里将`targetSDKVersion`设为`23`以后，除了在`AndroidManifest.xml`声明，我们还需要根据App运行时所在的手机的系统版本，在调用权限之前向用户申请授权，并在用户允许以后，才可以安全的调用对应的API。

<!-- more -->

简单举个例子，我的App有一个地理位置读取的功能，需要`ACCESS_COARSE_LOCATION`（粗略的地理位置信息）权限，如果Apk是以`targetSDKVersion=22`的方式进行打包的，那么在安装时用户将不得授权我的App这个权限，否则他将无法安装这个应用；如果是以`targetSDKVersion=23`的形式打包的，而且用户使用的手机是Android6.0的系统，那么在安装时我的App是没有获得读取地理位置权限的，我们需要调用系统提供的`requestPermissions`接口来申请权限。这时候你可能会说那我还设置`targetSDKVersion=23`干嘛？还不如直接用`22`打包得了。。

不要高兴得太早，下面我们会说理由。因为只要用户使用的是6.0系统的手机，他是可以在安装完成以后，在手机的设置界面取消一些`dangerous`权限的。

下图就是6.0系统上权限管理的界面。

![app-cancel-perm](/content/images/app-cancel-perm.png)

# targetSDKVersion和compileSDKVersion的作用

在`build.gradle`中有这么一段：

```
android {

    compileSdkVersion 23
    buildToolsVersion "23.0.2"
    ...
    
    defaultConfig {
        ...
        targetSdkVersion 22
        ...
    }

```

- targetSDKVersion：简单来说就代表着你的App能够适配的系统版本，意味着你的App在这个版本的手机上做了充分的**前向**兼容性处理和实际测试。其实我们写代码时都是经常干这么一件事，就是`if(Build.VERSION.SDK_INT >= 23) { ... }`，这就是兼容性处理最典型的一个例子。如果你的target设置得越高，其实调用系统提供的API时，所得到的处理也是不一样的，甚至有些新的API是只有新的系统才有的，例如前一篇博客里用到WebView的`setWebContentsDebuggingEnabled(boolean))`方法，这是Android4.4以后才可以用的一个API。

- compileSdkVersion：是你SDK的版本号，也就是你在编程时引用的`android.jar`的版本。一般都会和targetSDKVersion相等，或者比targetSDKVersion高。

因此，如果我们要把自己的App适配到Android6.0系统，首先要把`targetSDKVersion`和`compileSDKVersion`全部设置为`23`。


# Android6.0运行时权限系统到底是干嘛的？

介绍了targetSDKVersion和compileSDKVersion以后，我们就知道什么叫做适配到Android X.X了。那么Android6.0这个运行时权限系统和以前最大的不同就在于：将我们以前一股脑儿以`<permission>`标签方式写在`AndroidManifest.xml`文件中的权限划分成了**normal permission** 和 **dangerous permission**。

- Normal Permission：你写在xml文件里，那么App安装时就会默认获得这些权限，即使是在Android6.0系统的手机上，用户也无法在安装后动态取消这些normal权限，这和以前的权限系统是一样的，不变。

- Dangerous Permission：你还是得写在xml文件里，但是App安装时具体如果执行授权分以下几种情况：
	- `targetSDKVersion < 23` **&** `API(手机系统) < 6.0`：安装时默认获得权限，且用户无法在安装App之后取消权限。
	- `targetSDKVersion >= 23` **&** `API(手机系统) < 6.0`：安装时默认获得权限，且用户无法在安装App之后取消权限。
	- `targetSDKVersion < 23` **&** `API(手机系统) >= 6.0`：安装时默认获得权限，但是用户可以在安装App完成后动态取消授权（**取消时手机会弹出提醒，告诉用户这个是为旧版手机打造的应用，让用户谨慎操作**）。
	- `targetSDKVersion >= 23` **&** `API(手机系统) >= 6.0`：安装时不会获得权限，可以在运行时向用户申请权限。用户授权以后仍然可以在设置界面中取消授权。
	
6.0手机上取消授权的界面如下，如果是`targetSDKVersion < 23`打包的应用，取消权限时，会：

![app-权限](/content/images/app-permission.png)
	
我们可以看到其实只有在6.0的机器上，打包了适配到6.0的App，才需要做一些明显的权限适配工作。但是实话说，不论是不是适配6.0系统，我们都应该在调用需要权限的接口前，检查一下是否具有具有该权限。因为我们会遇到厂商的修改过权限系统的ROM，也有可能自己忘记在xml里加上权限清单。当然这里面也会有坑，因为有时在不同厂商手机上直接调用`checkCallingOrSelfPermission`方法得到的结果并不准确。

# 如果我不target到23会怎么样？

很显然，如果不target到23，短期内还可以不写权限适配的代码。但是！！！前面提到了，用户仍然可以在6.0的手机上取消安装时默认赋予的dangerous权限。**那么这时候会发生什么？？？？**如果这个API原本应该返回的是对象，那么这时将返回`null`；如果这个API原本应该返回的是数字，这时就是`0`。可想而知，如果你只是对返回的`null`或者`0`做了一些不恰当的处理，App还是可能会crash的。所以这不是长久之计，尽快地处理好权限申请才是王道，这也是一种对App、对用户更负责的做法。

# 如何在运行时申请权限 & 官方的权限分级列表

- 关于如何适配到6.0的权限系统，网上有非常多的例子，我觉得看了这篇文章基本上就OK了，因为网上很多貌似都翻译自这篇文章，强力推荐一下：[things-you-need-to-know-about-android-m-permission](http://inthecheesefactory.com/blog/things-you-need-to-know-about-android-m-permission-developer-edition)

- Google官方的RuntimePermissions介绍：[runtime-permissions](https://developer.android.com/preview/features/runtime-permissions.html)

- Google官方的dangerous权限及分组：[permissions: normal-dangerous](https://developer.android.com/intl/zh-cn/guide/topics/security/permissions.html#normal-dangerous)

- 开源社区-权限适配组件：伟大的开源社区已经有轮子了，可以学习一下！
	- [PermissionHelper](https://github.com/k0shk0sh/PermissionHelper)
	- [PermissionsDispatcher](https://github.com/hotchemi/PermissionsDispatcher)
	- [PermissionGen](https://github.com/lovedise/PermissionGen)
	- [TedPermission](https://github.com/ParkSangGwon/TedPermission):不用自己调用checkSelfPermission(), requestPermissions(),然后再处理回调onRequestPermissionsResult(), onActivityResult()。只要一行代码，设置一个listener就可以监听授权的结果，使用起来非常简单。内部实现采用的是类似event bus的otto实现的。


# 举个栗子（坑）：WRITE_EXTERNAL_STORGE的Android6.0适配问题

见下篇博客[Android6.0权限适配之WRITE_EXTERNAL_STORAGE（SD卡写入）](http://unclechen.github.io/2016/03/06/Android6.0权限适配之SD卡写入/)





