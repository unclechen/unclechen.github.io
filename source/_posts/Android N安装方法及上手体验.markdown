---
layout: post
title: Android N安装方法及上手体验
date: '2016-03-10'
photo: '/content/images/cover/android-n.png'
tags:
  - Android7.x
  - Android
  - 适配
categories: 
  - 技术
---

今天早上一睁眼，手机上就收到几条Android N的新闻，瞅了一眼发现手里的Nexus6可以安装beta版，于是迫不及待的开始看怎么安装了。直接看[官网](http://developer.android.com/intl/zh-cn/preview/download.html#device-preview)，一共三种安装方式：

- 在真机上直接安装，通过OTA的方式升级
- 在真机上用Android N的系统镜像安装
- 使用模拟器

# 1.真机安装
简单方便，[Android Beta Program](https://www.google.com/android/beta?pli=1)官网注册你的设备，然后。我是把手机插在电脑上的，登录我的Google账号以后，就检测出我的设备了，然后再点击下图中的`ENROLL DEVICE`（由于我已经点击过了，所以现在显示的是`UNENROLL DEVICE`），手机秒秒地就收到官方的推送！爽就一个字，点击进入下载，850.6M。

![enroll-device](/content/images/enroll-device.png)

下载完成后直接重启安装，大概需要5分钟。然后我手机上哪些应用喜欢自己后台一顿乱跑的就全部暴露出来了，可能是因为适配的原因很多应用都开始各种崩溃。首先是猎豹清理大师，接着是虾米音乐啊，淘宝啊，这些的，甚至有的SDK都开始不停地弹出对话框和Toast，“阿里百川XXXX”，真的是要哭了，我连关闭弹窗时间都来不及啊。。。。

<!-- more -->

# 2.下载系统镜像，线刷
其实我觉得有设备的同学，通过OTA是最好的。但是如果你没有翻墙的梯子，那可能就只能线刷了。下载地址在[这里](http://developer.android.com/intl/zh-cn/preview/download.html#device-preview)。如果有好心搬运到墙内，大家才可以下载。。

支持的机型为：

![android-n-devices](/content/images/android-n-devices.png)

下面开始刷机，官方的步骤在这里[Factory Images for Nexus Devices](https://developers.google.com/android/nexus/images#instructions)。

## 2.1首先下载最新的`Fastboot`工具

**获取途径有两种：**

- 编译好的[Android Open Source Project](https://source.android.com/)
- Android SDK下面的`platform-tools/`目录，记得打开你的`SDK Manager`下载最新的Android SDK Platform-tools，具体版本可以看看**模拟器**部分。

下载好了`Fastboot`工具以后，把它添加到系统的环境变量，这样下面我们要用的`flash-all`脚本才能在命令行中找到它。例如我的`fastboot`在`/Users/noughtchen/Library/Android/sdk/platform-tools/fastboot`下。

> 注意：刷机会删除掉手机上已有的所有用户数据，请备份好自己个人数据如手机上的照片。

**由于需要unlock bootloader来刷机，请先确保`开发者选项`中的`OEM unlocking`是选中状态。**

## 2.2下面正式进入刷机

- 1.下载合适的系统镜像，下载列表在[这里](http://developer.android.com/intl/zh-cn/preview/download.html#device-preview)。然后解压到你的硬盘上。
- 2.使用USB把手机连接到电脑上。
- 3.以`fastboot`模式启动你的手机的bootloader，有两种方式：
	- adb工具: 保持手机开机，命令行中执行：`adb reboot bootloader`
	- 使用手机的组合键：关机，然后使用同时按下`音量-`、`音量+`和`电源键`（**这是nexus5的组合键，其他机器请自行确认是哪几个组合**），来启动你的手机。
	
- 4.使用命令行unlock手机的`bootloader`，`fastboot flashing unlock`（如果是旧设备，则命令行为`fastboot oem unlock`，**具体是哪个命令，请根据自己的手机搜索一下吧**），此时，你的设备应该会弹出一个确认提示框，提示说会删除所有的数据。
- 5.打开一个终端，使用命令行进入到你最开始解压的镜像目录。
- 6.执行**`flash-all`**脚本，开始安装镜像。

当这个脚本执行完成后，你的设备会重启。为了安全起见，你需要lock住bootloader，这分两步：
- 1.重新以`fastboot`启动你的设备，参考上面的第3步。
- 2.执行`fastboot flashing lock`（如果是旧设备，则命令行为`fastboot oem lock`，**这里和上面的unlock是刚好相反的，请自己根据手机输入对应的命令**）。

在一些设备上，Lock bootloader也会删除数据。如果你想要再次刷机，那么就得按照上面的步骤重新再fastboot启动设备，然后unlock bootloader。 

**刷机有风险，操作需谨慎!!!**建议动手能力比较强的同学可以试试，我也只是把官网的步骤翻译解释了一下，手机变砖了不要来找我。。。

# 3.模拟器
这个简单，低成本，无风险。。其实我觉得没有设备的同学，通过这个也是可以体验一下。

操作之前，你需要下载最新的`Android N Preview SDK`，并新建一个虚拟机。

- 1.安装最新的SDK和build-tools，这里我就不说了。安装完成后，需要有`Android SDK Built-Tools 24.0 0 rc1`和`Platform-Tools 24.0.0 rc1`和`SDK Tools 25.0.9`（如果你不安装SDK Tools25.0.9，就无法运行Android N的x86_64的系统镜像）。

- 2.新建一个虚拟机。在Android Studio中打开`Tool-Android-AVD Manager`，点击`Create Virtual Device`，选择一个设备（建议选择Nexus 5X, Nexus 6P, Nexus 9其中一个）。然后选择下一步，需要注意一点的是，虚拟机只支持x86的Android N系统镜像。好了，你可以一直点击下一步知道这个新的虚拟机创建完成，然后启动它吧。为了获得最佳的体验，建议安装`Android Studio 2.1 Preview`，因为Android Studio2.0开始，虚拟机的速度得到了几十倍的提升。

> 注意: 如果你现在使用的是Android Studio 2.0 Beta, 它有个已知的bug，是无法用Android N Preview的镜像来创建虚拟机的, 所以你需要安装Android Studio 2.1 preview 来新建虚拟机。

> For more information about creating virtual devices, see [Managing Virtual Devices](http://developer.android.com/tools/devices/index.html).

# 4.上手体验

不说别的，今天发现一个一个特别好的Android网站[DROIDLIFE](http://www.droid-life.com)，推荐给大家，这里面有Android相关的新闻视频等等，非常的棒。今天他们已经出了一个视频，介绍了Android 7.0的新特性了，例如设置菜单、通知栏、分屏等等，iOS去年就出了画中画特性，但是只能在iPad上用，Android 7.0在手机上也支持应用分屏的哟，赶紧体验一下吧。[heres-everything-new-in-android-n-developer-preview](http://www.droid-life.com/2016/03/09/heres-everything-new-in-android-n-developer-preview/)，最后奉上一张我自己体验的效果图。

![first-experience](/content/images/first-experience.png)










