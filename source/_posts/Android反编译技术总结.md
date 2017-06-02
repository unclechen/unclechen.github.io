---
layout: post
title: Android反编译技术总结
date: '2016-09-07'
tags:
  - Android
  - 反编译
categories: 
  - 技术
---

# 一、Apk反编译工具及其使用方法

## 1.原理

学习反编译之前，建议先学习一下Apk打包的过程，明白打包完成后的Apk里面都有什么文件，各种文件都是怎么生成的。

这里有两篇AndroidWeekly中推荐过的好文章：

- [浅析 Android 打包流程](http://mp.weixin.qq.com/s?__biz=MzI0NjIzNDkwOA==&mid=2247483789&idx=1&sn=6aed8c7907d5bd9c8a5e7f2c2dcdac2e&scene=1&srcid=0831CCuRJsbJNuz1WxU6uUsI#wechat_redirect)

- [Android构建过程分析](http://mp.weixin.qq.com/s?__biz=MzI1NjEwMTM4OA==&mid=2651232113&idx=1&sn=02f413999ab0865e23d272e69b9e6196&scene=1&srcid=0831gT4p6M0NFG5HTTeRHTUC#wechat_redirect)


Apk技术也有非常多的技术可以学习，主要都是围绕着如何减小体积，如何提高打包速度展开，这里先不多说了。下面是一张基本的Apk文件结构图。

![APK文件结构](http://ww2.sinaimg.cn/large/65e4f1e6gw1f7l2d6e4z5j20a80a0jrw.jpg)

<!-- more -->

Apk文件本质上其实是一个zip包。直接拿解压工具解压就可以看到其中包含了什么。下面简单介绍一下Apk文件的结构。

- AndroidManifest.xml：应用的全局配置文件
- assets文件夹：原始资源文件夹，对应着Android工程的assets文件夹，一般用于存放原始的网页、音频等等，与res文件夹的区别这里不再赘述，可以参考上面介绍的两篇文章。
- classes.dex：源代码编译成class后，转成jar，再压缩成dex文件，dex是可以直接在Android虚拟机上运行的文件。
- lib文件夹：引用的第三方sdk的so文件。
- META-INF文件夹：Apk签名文件。
- res文件夹：资源文件，包括了布局、图片等等。
- resources.arsc：记录资源文件和资源id的映射关系。

上面的截图中每个文件都是一个最基本的Apk
文件应该包含在内的。但是直接把Apk当做zip解压后的这些文件是没法直接阅读的，毕竟他们都是经过了build-tools打包工具处理过的。我们直接用文本编辑器打开这里面的Manifest文件看看。

![反编译前的Manifest文件](http://ww3.sinaimg.cn/large/801b780agw1f7kzdom045j20go06sjul.jpg)

**反编译Apk**的目的就是Apk拆成我们可以阅读的文件。通过反编译，我们一般想要得到里面的**AndroidManifest.xml文件**、**res文件**和**java代码**。


## 2.Apk反编译步骤

### (1) ApkTool拆包，得到AndroidManifest和res等资源文件

**工具下载地址：**[https://bitbucket.org/iBotPeaches/apktool/downloads](https://bitbucket.org/iBotPeaches/apktool/downloads)

**功能：**拆解Apk文件，反编译其中的资源文件，将它们反编译为可阅读的**AndroidManifest.xml文件**和**res文件**。前面讲过，直接把Apk文件当做zip解压，得到的xml资源文件，都是无法直接用文本编辑器打开阅读的，因为它们在打包时经过了build-tools的处理。

**用法：**官网[https://ibotpeaches.github.io/Apktool/documentation/](https://ibotpeaches.github.io/Apktool/documentation/)有介绍，最新版本是**2.2.0**，运行环境需要**jre1.7**。

这里，我演示一下用apktool来拆解Apk文件的基本方法，只需要在终端里面执行下面的命令。

```
java -jar apktool.jar d yourApkFile.apk
// 注意`apktool.jar`是刚才下载后的jar的名称，`d`参数表示decode
// 在这个命令后面还可以添加像`-o -s`之类的参数，例如
// java -jar apktool.jar d yourApkFile.apk -o destiantionDir -s
// 几个主要的参数设置方法及其含义：
-f 如果目标文件夹已存在，强制删除现有文件夹
-o 指定反编译的目标文件夹的名称（默认会将文件输出到以Apk文件名命名的文件夹中）
-s 保留classes.dex文件（默认会将dex文件解码成smali文件）
-r 保留resources.arsc文件（默认会将resources.arsc解码成具体的资源文件）
```


下面我们看一下`java -jar apktool.jar d yourApkFile.apk`拆解后的结果：

![Apk拆包结果](http://ww1.sinaimg.cn/large/801b780agw1f7kxk0y0i9j20h00c6dgm.jpg)

我们已经得到一个可以用文本编辑器打开的阅读的**AndroidManifest.xml文件、assets文件夹、res文件夹、smali文件夹**等等。original文件夹是原始的AndroidManifest.xml文件，res文件夹是反编译出来的所有资源，smali文件夹是反编译出来的代码。注意，smali文件夹下面，结构和我们的源代码的package一模一样，只不过换成了smali语言。它有点类似于汇编的语法，是Android虚拟机所使用的寄存器语言。

这时，我们已经可以文本编辑器打开**AndroidManifest.xml**文件和**res下面的layout文件**了。这样，我们就可以查看到这个Apk文件的**package包名、Activity组件、程序所需要的权限、xml布局、图标**等等信息。其实我们把Apk上传到应用市场时，应用市场也会通过类似的方式解析我们的apk。


> **note1：**其实还有一种方法，可以省去每次解包时，都要输入`java -jar apktool.jar xxx`这行命令，官网也有说明，就是将这个命令包装成shell脚本，方法见：[https://ibotpeaches.github.io/Apktool/install/](https://ibotpeaches.github.io/Apktool/install/)


> **note2：**如果你在编译的时候，发现终端里面提示发生了**brut.android.UndefinedResObject**错误，说明你的apktool.jar版本太低了，需要去下载新版工具了。


> **note3：**如果想要自己实现一个解析Apk文件，提取版本、权限信息的***java***服务时，可以引用`apktool.jar`中的`ApkDecoder`，调用`decode`方法来实现。可以看下图中，apktool.jar里面有解析Apk文件的实现。

![apktool.jar](http://ww4.sinaimg.cn/large/801b780agw1f7kxsfcc9gj20m80f8ju0.jpg)



### (2) dex2jar反编译dex文件，得到java源代码

上一步中，我们得到了反编译后的资源文件，这一步我们还想看java源代码。这里要用的工具就是**dex2jar**。

**工具下载地址：**[https://sourceforge.net/projects/dex2jar/](https://sourceforge.net/projects/dex2jar/)

**功能：**将dex格式的文件，转换成jar文件。dex文件时Android虚拟机上面可以执行的文件，jar文件大家都是知道，其实就是java的class文件。在[官网](https://github.com/pxb1988/dex2jar)有详细介绍。

**用法：**打开下载的dex2jar-2.0文件夹，里面有shell和bat脚本，进入终端，就可以在命令行使用了。

```
d2j-dex2jar classes.dex
// 获取classes.dex文件在最前面说过，只要把Apk当做zip解压出来，里面就有dex文件了
// 或者用apktool反编译时带上 `-s` 参数
```

运行后，可以看到**classes.dex**已经变成了**classes-dex2jar.jar**。

![进入dex2jar文件夹](http://ww4.sinaimg.cn/large/801b780agw1f7kyy6qkwhj20go0e675q.jpg)


> **note1：**第一次下载下来后，在mac里运行的时候可能会提示需要管理员的权限，这里我给这些sh脚本`chmod 777`后，即可运行它。

![root执行dex2jar](http://ww1.sinaimg.cn/large/801b780agw1f7kz7zdeewj20go018t92.jpg)

> **note2：**写完这一节的时候，我发现**把dex转换成jar**已经有了更好的工具**enjarify**，[https://github.com/google/enjarify](https://github.com/google/enjarify)这个工具是谷歌官方开源的用于反编译dex文件的。使用方法和dex2jar差不多，也是简单的命令行操作。这个工具的主页中也提到dex2jar已经是一个比较老的工具，在遇到混淆等等复杂的情况时，可能无法正常工作。所以这里推荐大家使用**enjarify**这个工具。


### (3) jd-gui查看java源代码

**工具下载地址：**官网[http://jd.benow.ca/](http://jd.benow.ca/)上选择自己所需要的版本。

**功能：**这个工具不用多说，写java的人都知道。有时候我们自己开发一个jar包给别人用，也会用它来查看class是不是都被正确的打入到了jar内，我以前介绍的gradle自定义打包jar的博客中也提到过它。

**用法：**下载后双击既可以运行这个工具，直接把上一步得到的**classes-dex2jar.jar**拖到jd-gui程序的界面上即可打开了，效果如下图所示。

![classes-dex2jar.jar](http://ww3.sinaimg.cn/large/801b780agw1f7kzaffqirj20go04lmxb.jpg)

### 反编译Apk步骤小结

反编译一个Apk，查看它的资源文件和java代码，我们需要用到3个工具。

- apktool：[https://ibotpeaches.github.io/Apktool/](https://ibotpeaches.github.io/Apktool/)
- dex2jar：[https://github.com/pxb1988/dex2jar](https://github.com/pxb1988/dex2jar)
- jd-gui：[http://jd.benow.ca/](http://jd.benow.ca/)

反编译就是用这3个工具得到AndroidManifest.xml、res、java代码等。但是我们可以看到，如果你要对一个Apk做尽可能彻底的反编译，把它扒得干干净净，这一步一步的基本操作还是稍显麻烦。另外加固过Apk的情况可能更复杂，需要我们勤动手尝试。为了能提高效率，下面我把自己见过的一些集成工具介绍给大家，尽可能实现可以一键反编译Apk。


# 二、自动化工具汇总（一键反编译Apk）

## 1.谷歌提供的工具：[android-classyshark](http://classyshark.com/)

**下载地址：**[https://github.com/google/android-classyshark/releases](https://github.com/google/android-classyshark/releases)，下载下来之后是一个可执行的jar文件，win下或者mac下都只要双击即可运行。

**功能：**带有界面，一键反编译Apk工具，直接打开Apk文件，就可以看到Apk中所有的文件结构，甚至还集成了dex文件查看，java代码查看，方法数分析、导入混淆mapping文件等一系列工具。谷歌推出这个工具的目的是为了让我们开发者更清楚的了解自己的Apk中都有什么文件、混淆前后有什么变化，并方便我们进一步优化自己的Apk打包实现。下面带上几张截图，真是帅气的一笔的好工具啊！

![dex文件查看](http://ww1.sinaimg.cn/large/801b780agw1f7l03o4znvj20m80e8n0f.jpg)

![方法数分析](http://ww2.sinaimg.cn/large/65e4f1e6gw1f7l21d8lm6j20p009iac1.jpg)

即将到来的**Android Studio 2.2**中集成了一个叫做**APK Analyzer**的功能，这个功能不知道是不是和这个工具有关系呢，本人还没有尝试过2.2版本，有兴趣的朋友可以体验一下[preview版本](http://android-developers.blogspot.com/2016/05/android-studio-22-preview-new-ui.html)。


## 2.Python实现的工具：[AndroidGuard](https://github.com/androguard/androguard)

**下载地址：**[https://github.com/androguard/androguard/releases](https://github.com/androguard/androguard/releases)

**功能：**集成了反编译资源、代码等各种文件的工具包。需要安装Python环境来运行这个工具，这个工具按照不同的反编译需求，分别写成了不同的py功能模块，还有静态分析的功能。所以如果想要用Python开发一个解析Apk文件并进行静态扫描分析的服务，可以引用这个工具来实现。

**用法：**具体用法比较多，这里也不再展开了。可以通过工具内置的`-h`帮助指令查看各个模块的功能。

```
unclechendeiMac:androguard-2.0 unclechen$ python androaxml.py -h
Usage: androaxml.py [options]

Options:
  -h, --help            show this help message and exit
  -i INPUT, --input=INPUT
                        filename input (APK or android's binary xml)
  -o OUTPUT, --output=OUTPUT
                        filename output of the xml
  -v, --version         version of the API

// androaxml.py这个模块是用来解析AndroidManifest文件的，`-i` 表示输入的apk文件，`-o` 表示输出xml文件。

```


## 3.Mac专属工具：[Android-Crack-Tool](https://github.com/Jermic/Android-Crack-Tool)

**功能：**这是网上一位名为[Jermic](https://github.com/Jermic)的大神开发的、在Mac环境下使用的App，集成了Android开发中常见的一些编译/反编译工具，方便用户对Apk进行逆向分析，提供Apk信息查看功能。工具的截图如下所示，非常强大。

![Android-Crack-Tool.app](http://ww3.sinaimg.cn/large/801b780agw1f7l1q0hwugj20rs0gtwjl.jpg)


## 4.手机上的反编译工具：[ApkParser](https://github.com/jaredrummler/APKParser)

**功能：**在电脑上已经有了这么多的工具，在手机上的也有很方便的工具。**APKParser**是一款在查看手机上已经安装的Apk的信息的工具，他可以查看软件的**AndroidManifest.xml文件、方法数、res资源文件**，并在手机上直接展示出来。个人觉得这是一个非常实用的工具，作为开发者，手机里面必须要有它。

![ApkParser](http://ww1.sinaimg.cn/large/65e4f1e6gw1f7l1wnbkv1j20m80b4jtf.jpg)


## 5.工具汇总

以上几款工具都是我体验过、感觉不错的集成工具，推荐给大家。临近本文结束前，又发现了这么一个福利网站-[http://www.androiddevtools.cn/](http://www.androiddevtools.cn/)，其中有一章专门总结了各种Apk反编译的工具。相信有了这么多的利器，大家应该有100种方法将一个App扒得干干净净了。

![Apk反编译工具汇总](http://ww3.sinaimg.cn/large/801b780agw1f7l112kl4yj20m80pfwiq.jpg)