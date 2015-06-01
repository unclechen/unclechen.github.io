---
layout: post
title: Android Studio+Gradle安装配置笔记
cover_image: '/content/images/cover/helloAndroidStudioAndGradle.png'
---

2015年5月29号凌晨，第一次翻墙看了谷歌开发者大会的直播，还特意跑到车库咖啡和一堆GDG开发者一起看的直播。现场来了一些Android、Google开发大牛，介绍了他们对于最新的Android、Google技术的体验。加上谷歌官方宣布了1.3beta版本的[Android Studio](https://developer.android.com/sdk/index.html)已经支持NDK开发，这样一来貌似Eclipse在Android开发方面的最后一个独有优势也已经失去。我想如果不是公司项目需要，我已经可以把AS作为我的Android主打开发环境了。另外据说1.3开始，gradle脚本的速度也会得到优化，因此希望在AS这个官方的平台上能够有更多优化的体验。

下面我还是以自己的AS 1.2.1.1 stable版的AS介绍一些我的学习经验。


# 1. Android Studio安装配置

## 1.1 下载、安装和配置

首先最重要的一步！！**科学上网**！！官网[下载](https://developer.android.com/sdk/index.html)最新版的AS安装文件，Windows下对应的是exe文件，MAC下应该是一个dmg文件，安装过程很简单方便。

**（1）解决Fetching android sdk component information卡住的问题**

安装完成后，打开Android Studio的时候可能常常会卡在启动界面 **“Fetching android sdk component information”** 上，这是因为AS启动时会去获取SDK组件信息，然而我们现在还没有去给Android Studio的SDK组件配置代理，因为可能会非常慢，甚至没有速度。

**解决办法是->** 进入刚安装的Android Studio目录，打开`bin`文件夹，找到`idea.properties`文件，用文本编辑器打开。在文件末尾添加一行：
> disable.android.first.run=true

，保存。然后在任务管理器中关闭Android Studio，重新启动即可。

**（2）设置SDK、JDK目录**

安装完成以后，打开AS。进入的是`Android Studio Setup Wizard`界面，如下所示。

![hello AS](/content/images/helloWizard.png)

这里会引导我们进行一些IDE的基本设置。包括你的Android SDK目录设置等等，这里我推荐点击`Cancel`跳过，因为我们原来在Eclipse环境下开发的时候，已经下载过Android SDK了。只需要在进入AS以后，再把SDK的目录设置为原来的SDK就好，免去下载新的sdk的麻烦，国内这个环境，就是有代理也是很慢的。跳过设置向导以后，进入AS的欢迎首页。

![hello AS](/content/images/helloAS.png)

接着，选择界面上`Configure->Project Defaults`，进入之后再选择`Project Structure`。到这里就可以把Android SDK和JDK的目录设置为我们之前已经有的目录了。

![hello SDK](/content/images/helloSDK.png)

**注意:** AS需要的JDK版本必须在1.7以上，因为AS默认使用gradle打包。


**（3）设置主题、字体设置，网络代理**

设置了SDK、JDK目录以后，我们可以再给AS做一个基本的设置，为了让这个IDE更加顺手好用。回到AS的首页，选择`Configure->Settings`，进入下面的界面。

![hello Settings](/content/images/helloSettings.png)

首先，在`Appearance & Behavior->Appeara`选择`Darcula`主题，字体可以调大一些。这样整个AS就变成黑色背景的主题了，看着舒服很多。

然后是代理的配置，在Settings界面左侧的菜单栏中，最上面有一个搜索框，可以直接搜索各种设置选项，我们输入`proxy`，即可跳转到代理设置的界面，如图所示我把代理设置为公司的`xxx-proxy.oa.com`。这样以后我们就可以使用AS自动更新了。所以大家一定要去搞一个自己的翻墙代理。。

![hello Proxy](/content/images/helloProxy.png)

（4）新建一个test项目

完成了上面的设置以后，我们使用AS的新建一个test项目。回到AS的欢迎主页，点击`Start a new Android Studio Project`，如下如所示设置应用的名称为`TestApplication`，选择项目目录。

![hello Proxy](/content/images/helloCreatTestApplicaiton.png)

然后选择Minimum SDK的版本。接着一路点`next`直到`finish`。

![hello Proxy](/content/images/helloCreatTestApplicaiton2.png)

进入代码编辑界面。突然发现`Code Editor`字体很小，原来是刚才设置的Appearance的字体只是IDE的字体，没有给编辑器设置字体大小，没关系，我们在菜单栏里依次选择`File->Settings`，然后搜索`font`，点击`Save AS...`，然后把字号Size由`12`改成`18`，字体Primary font改成`Consolas`，最后点击`OK`即可保存。

![hello Proxy](/content/images/helloFont.png)

这样，就使用AS建立了一个Test项目。

**PS** ：实际操作时，并没有一次就搞定这么简单的一个新建项目，可能有的朋友会碰到像我一样的问题，就是这个新建的项目中，`Module app`的`gradle`脚本默认使用了最新的`buildToolsVersion "23.0.0 rc"`（因为我把SDK和buildTools都更新到最新版了，默认应该是选择最新的版本来build），因此遇到了 **`Execution failed for task ':app:compileDebugAidl': aidl is missing`** 的问题，解决办法很简单，把`Module app`的`gradle`脚本修改一下，使用22.0.1的buildTools就好了。

关于AS项目结构以及gradle脚本的知识，下面有介绍。



## 1.2 Android Studio项目结构

Android Studio的项目结构和Eclipse下有很大不同，相当于一个工作空间下面只有一个项目的概念。。。。


## 1.3 国内Android SDK升级方法

做Android开发的朋友都知道，当Google被墙掉之后，真的是太不方便了，什么新的东西，好用的资源都没法及时的得到。所以我们一定需要一个代理。但是国内也有很多组织，为大家带来了很多福利，比如Android SDK，就有东软的镜像，下面介绍一下怎么设置。

启动`Android SDK Manager`，在主界面上方的菜单栏里依次选择`Tools->Options`，弹出`Android SDK Manager Settings`窗口中，如下图所示。

![hello SDKProxy](/content/images/helloSDKProxy.png)

分别在`HTTP Proxy Server`输入`mirrors.neusoft.edu.cn`，`HTTP Proxy Port`中输入`80`。然后勾选中`Force https://... sources to be fetched using http://...`，最后点击`close`按钮关闭设置。回到`Android SDK Manager Settings`窗口以后，一般会自定刷新重新加载，如果没有自动刷新的话，可以依次手动点击菜单栏中的`Pakages->Reload`，然后就可以飞速地更新最新的Android SDK了。

可以看到我已经下载了最新的Android M SDK预览版，不过我用的是公司的代理，但其实速度还没有东软这个快呢。

## 1.4 Eclipse项目迁移到Android Studio

最新版的AS已经支持直接导入Eclipse项目了。（不需要向像原来那样，先从Eclipse导出一个gradle script，再导入到AS中）

这里有[官方的迁移教程](https://developer.android.com/sdk/installing/migrate.html)，大家还是多看官方教程，这样其实是最有效率的。

导入Eclipse的项目后，AS会在项目的目录文件夹下生成一个`import-summary.txt`，这里面有导入过程的一些问题，导入进来的项目可能不一定立刻就能build，但是通过看导入报告，我们可以将有问题的地方一一修复，最后成功迁移到AS上来。相信我，我一开始的时候也是非常痛苦的，花了很多时间以后最终也就摸索出来了。StackOverflow真的是个好东西啊。


## 1.5 快捷键

以下摘自[Android Studio中文论坛](http://android-studio.org/)

Alt+回车 导入包,自动修正

Ctrl+N   查找类

Ctrl+Shift+N 查找文件

Ctrl+Alt+L  格式化代码

Ctrl+Alt+O 优化导入的类和包

Alt+Insert 生成代码(如get,set方法,构造函数等)

Ctrl+E或者Alt+Shift+C  最近更改的代码

Ctrl+R 替换文本

Ctrl+F 查找文本

Ctrl+Shift+Space 自动补全代码

Ctrl+空格 代码提示

Ctrl+Alt+Space 类名或接口名提示

Ctrl+P 方法参数提示

Ctrl+Shift+Alt+N 查找类中的方法或变量

Alt+Shift+C 对比最近修改的代码

Shift+F6  重构-重命名

Ctrl+Shift+先上键

Ctrl+X 删除行

Ctrl+D 复制行

Ctrl+/ 或 Ctrl+Shift+/  注释（// 或者 ）

Ctrl+J  自动代码

Ctrl+E 最近打开的文件

Ctrl+H 显示类结构图

Ctrl+Q 显示注释文档

Alt+F1 查找代码所在位置

Alt+1 快速打开或隐藏工程面板

Ctrl+Alt+ left/right 返回至上次浏览的位置

Alt+ left/right 切换代码视图

Alt+ Up/Down 在方法间快速移动定位

Ctrl+Shift+Up/Down 代码向上/下移动。

F2 或Shift+F2 高亮错误或警告快速定位

代码标签输入完成后，按Tab，生成代码。

选中文本，按Ctrl+Shift+F7 ，高亮显示所有该文本，按Esc高亮消失。

Ctrl+W 选中代码，连续按会有其他效果

选中文本，按Alt+F3 ，逐个往下查找相同文本，并高亮显示。

Ctrl+Up/Down 光标跳转到第一行或最后一行下

Ctrl+B 快速打开光标处的类或方法 

***

最常用快捷键

1.Ctrl＋E，可以显示最近编辑的文件列表

2.Shift＋Click可以关闭文件

3.Ctrl＋[或]可以跳到大括号的开头结尾

4.Ctrl＋Shift＋Backspace可以跳转到上次编辑的地方

5.Ctrl＋F12，可以显示当前文件的结构

6.Ctrl＋F7可以查询当前元素在当前文件中的引用，然后按F3可以选择

7.Ctrl＋N，可以快速打开类

8.Ctrl＋Shift＋N，可以快速打开文件

9.Alt＋Q可以看到当前方法的声明

10.Ctrl＋W可以选择单词继而语句继而行继而函数

11.Alt＋F1可以将正在编辑的元素在各个面板中定位

12.Ctrl＋P，可以显示参数信息

13.Ctrl＋Shift＋Insert可以选择剪贴板内容并插入

14.Alt＋Insert可以生成构造器/Getter/Setter等

15.Ctrl＋Alt＋V 可以引入变量。例如把括号内的SQL赋成一个变量

16.Ctrl＋Alt＋T可以把代码包在一块内，例如try/catch

17.Alt＋Up and Alt＋Down可在方法间快速移动

***

下面占个坑先。。

# 2. Gradle插件自动化多渠道打包实践

## 2.1 Gradle安装配置

## 2.2 使用Gradle自动化多渠道打包


今年的开发者大会上我比较关注的还是Android在系统上的一些改进，以及Material Design的一些最新实践。当我打开IO大会官网看到他们用polymer写的网页时，感觉还是非常不错的。