---
layout: post
title: Android6.0权限适配之WRITE_EXTERNAL_STORAGE（SD卡写入）
date: '2016-03-06'
tags:
  - Android
  - 适配
  - 权限
categories: 
  - 技术
---

前一篇博客中介绍了[Android6.0运行时权限简介](http://unclechen.github.io/2016/03/05/Android6.0运行时权限简介/)，最近遇到这么一个情况，就是一个App以前都是在SD卡根目录直接新建了一个`XXX/image/`目录，来保存图片缓存的，但是如果适配到Android6.0，我们就需要弹出对话框给用户，来申请`WRITE_EXTERNAL_STORAGE`权限，如果仅仅是缓存图片为了提高加载速度，对于一个小白用户来讲，好像并不是什么值得让他授权的理由。。。

下面记录一下我是怎么处理的，其实这次处理也不能叫做Android6.0权限适配了，不过对于`WRITE_EXTERNAL_STORAGE`这个权限而言，的确有一些需要注意的地方（坑）使我们以前没有关心的。

首先，App在手机上保存文件或者缓存数据时，我认为应该遵守以下几点：

- 1.不要随意占用用户的内置存储。
- 2.不要随意在SD卡上新建目录，应该放置自己应用包名对应的扩展存储目录下，卸载App时可以被自动清除。
- 3.对占用的磁盘空间有上限，并按照一定的策略进行清除。

<!-- more -->

# Android下有哪些文件目录

在Android系统中，根据调用的系统API接口，有3种目录可以给我们写入文件：

- 1.**应用**私有存储（内置存储）
	- 获取方式：
		- `Context.getFileDir()`：获取内置存储下的文件目录，可以用来保存不能公开给其他应用的一些敏感数据如用户个人信息
		- `Context.getCacheDir()`：获取内置存储下的缓存目录，可以用来保存一些缓存文件如图片，当内置存储的空间不足时将系统自动被清除（然而具体多大，清除时的策略我也没查到。。）
	- 绝对路径：
		- `Context.getFileDir()`：`/data/data/应用包名/files/`
		- `Context.getCacheDir()`：`/data/data/应用包名/cache/`
	- 写权限：不需要申请
	
	这是手机的内置存储，没有root的过的手机是无法用文件管理器之类的工具查看的。而且这些数据也会随着用户卸载App而被一起删除。这两个目录其实就对应着`设置->应用->你的App->存储空间`下面的`清除数据`和`清楚缓存`，如下图所示。
	
	![app-存储空间](/content/images/app-storage.png)
	
- 2.**应用**扩展存储（SD卡）
	- 获取方式：
		- `Context.getExternalFilesDir()`：`获取SD卡上的文件目录`
		- `Context.getExternalCacheDir()`：`获取SD卡上的缓存目录`
	- 绝对路径：
		 - `Context.getExternalFilesDir()`：`SDCard/Android/data/应用包名/files/`
		- `Context.getExternalCacheDir()`：`SDCard/Android/data/应用包名/cache/`
	- 写权限：
		- API < 19：需要申请
		- API >= 19：不需要申请
		
	既然是SD卡上的目录，那么是可以被其他的应用读取到的，所以这个目录下，不应该存放用户的敏感信息。同上面一样的，这里的文件会随着App卸载而被删除，也可以由用户手动在设置界面里面清除。	
	
- 3.公共存储（SD卡）
	- 获取方式：`Environment.getExternalStorageDirectory()`
	- 绝对路径：`SDCard/你设置的文件夹名字/`
	- 写权限：需要申请
	
	如果我们的App需要存储一些公共的文件，甚至希望下载下来的文件即使在我们的App被删除之后，还可以被其他App使用，那么就可以使用这个目录。这个目录是始终需要申请SD写入权限的。
	
# Android6.0下应该把文件放到哪里？

有了前一节的介绍，其实很清楚了，根据最开始提到的规则，其实如果仅仅是做了简单的图片缓存工作，那么我们应该把图片缓存放到`/data/data/应用包名/cache/`或者`SDCard/Android/data/应用包名/cache/`，因为在6.0系统（`API > 23`）时，不需要申请权限就可以向这两个目录写入文件。而且`/data/data/应用包名/cache/`目录，是内置存储的应用私有缓存目录，在系统空间不够时还会被自动清除，对于图片缓存来讲也是一个不错的管理策略，不过谷歌建议我们最好还是自己实现缓存清除管理，例如用`DiskLruCache`。

实际上我们可以在`API >= 19（不一定非要大于23）`时，就可以在不需要申请权限的情况下把文件放到这两个目录了。如果开发的时候足够规范，即使在`API < 19`时，我们申请到写入权限后，我们也应该手动创建和前面相同的目录，使得应用存储数据目录统一化。


# Last，最后还有个坑！

好了，是不是现在不用SD卡上创建的目录`XXX/image/`，直接改为改为`SDCard/Android/data/应用包名/cache/image/`就OK了？还真不完全是这样的。。。

**？？？纳尼？？？？**

通常我们开发App时会设置`targetSDKVersion=23`时，并同时**向前兼容**，还会设置`minSdkVersion=14`表示支持的最低系统版本是Android4.0(`API = 14`)。也就是说我们的`build.gradle`一般长这样：

```
android {
    compileSdkVersion 23
    buildToolsVersion "23.0.2"
    ...
    defaultConfig {
        applicationId "xxx.xxx"
        minSdkVersion 14
        targetSdkVersion 23
        versionCode 1
        versionName "1.0"
    }
    ...
}
```

但是前面我们说过，通过`Context.getExternalCacheDir()`接口获取应用扩展存储目录时，只有在`API >= 19`时才不需要申请权限。也就是说如果是上面这种兼容到`API 14`的应用，还是需要在`AndroidManifest.xml`中注册`WRITE_EXTERNAL_STORAGE`权限的。

前一篇博客[Android6.0运行时权限简介](http://unclechen.github.io/2016/03/05/Android6.0运行时权限简介/)知道，如果在`AndroidManifest.xml`文件里注册过`WRITE_EXTERNAL_STORAGE`，当App运行在一台6.0的设备时，即使你的App全程都没有调用`requestPermissons`来申请权限，用户还是可以在**Android6.0系统上** 进入`设置->应用->你的App->权限`里面，取消`存储空间`这一个权限。记住是运行在**6.0系统的机器**上，这是关键，因为低于6.0的系统根本没有这个设置。

如下图所示，只要在manifest里面注册了，就可以动态取消之！

![app-权限](/content/images/app-perm-before.png)

此时会发生什么？？？此时你的图片在6.0机器上也就没法缓存喽。。/(ㄒoㄒ)/~~  

为啥啊？6.0机器上，我不是不需要申请权限就可以获得写入`SDCard/Android/data/应用包名/cache/`目录吗？实际测试时发现，当用户取消了权限之后，SDK接口中与`File`相关的API全部都返回空了，于是我们就没法写文件了。

其实我们还需要做的是：

将AndroidManifest.xml文件中的

```
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>

```
改为

```
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"
android:maxSdkVersion="18"/>

```

表示只在`API <= 18`时，才申请`WRITE_EXTERNAL_STORAGE`权限。这样用户就无法在Android6.0系统的设置下面看到`存储空间`权限的开关，当然也就无法关闭它了，如下图所示。

![app-权限](/content/images/app-perm-after.png)









