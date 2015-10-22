---
layout: post
title: Android Studio Gradle打包实践之多渠道+版本号管理
---

上次介绍了[Android Studio的安装、配置和基本使用](http://unclechen.github.io/2015/06/01/Android%20Studio%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AE%E7%AC%94%E8%AE%B0/)。这次讲一下Android Studio用到的打包工具Gradle。[Gradle](http://gradle.org/)是一种构建项目的框架，兼容Maven、Ant，为Java项目提供了很多插件去实现打包功能。废话不多说，下面直接进入实战。当我写这篇博客的时候，Android Studio的版本已经更新到了**1.4**，比上一篇博客的版本又更新了。

# Android Studio工程build.gradle脚本介绍

在进行多渠道打包之前，先介绍一下Android Studio工程中的gradle脚本长什么样。打开Android Studio，新建一个Project，这里我给它命名为Hello Gradle，一路点击下一步，最后Android Studio自动为我们建立的如下图的这个工程。

![hello AS](/content/images/create-a-project.png)

按照上篇博客中介绍的，我们推荐大家采用Android结构的视图来查看项目结构。展开Gradle Scripts我们可以看到里面有两个`build.gradle`文件和一个`settings.gradle`文件。其中的`build.gradle(Project: HelloGradle)`文件是我们整个工程的build文件，而`build.gradle(Module: app)`文件是我们工程下的一个Module的build文件。前面我们就说过Android Studio采用单工程多Module结构，一个工程可以理解为Eclipse下的一个Workspace，一个Module可以理解为Eclipse下的Project。当我们用Android Studio建立一个默认的工程时，它自动为我们建立了一个名字为`app`默认的Module。

所以我们可以知道，一个Android Studio工程会有一个`工程级别的build.gradle`文件，同时有N个Module，就还会有N个`Module级别的build.gradle`文件。

## 工程目录下的build.gradle(Project: HelloGradle)文件

接着我们先看下这个工程级别的build.gradle文件。

{% highlight xml %}
// Top-level build file where you can add configuration options common to all sub-projects/modules.

buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:1.3.0'

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

allprojects {
    repositories {
        jcenter()
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}

{% endhighlight %}

这个文件里的buildscript闭包中为我们定义了工程用到的repository地址，默认为我们加上了jcenter，并添加了版本号为1.3.0的Android Gradle插件。关于闭包，由于gradle是基于Groovy语言编写的，而闭包是里面的一个概念，可以理解为最小的代码执行块。关于jcenter，可以理解为一个兼容Maven中央仓库的东西，是Google为Android建立的。

最下面还有一个task clean，**task**是gradle脚本中用到最多的东西了。Gradle实际上是一个容器，实现真正的功能的都是Gradle的插件Plugin，而Plugin中又定义了各式各样的Task，这一个个的Task是执行任务的基本单元。

这里一看就知道是一个delete类型的task，意思是在我们执行打包脚本前做一个清理工作，把项目输出文件夹中的文件先全部清理干净。

## Module目录下的build.gradle(Module: app)文件

接着看app Module下的build.gradle文件。

{% highlight xml %}
apply plugin: 'com.android.application'
{% endhighlight %}

第一行`apply plugin: 'com.android.application'`：指的是在这个脚本中应用**Android Application**插件。前面我们说到了Gradle中真正起作用的是插件，每个插件中可以定义各种各样的Task，当然还可以有一些Property属性，如果你以前是用Ant打包的，那么对属性一定不会陌生吧。

{% highlight xml %}
android {
    compileSdkVersion 22
    buildToolsVersion "23.0.1"

    defaultConfig {
        applicationId "com.nought.hellogradle"
        minSdkVersion 14
        targetSdkVersion 22
        versionCode 1
        versionName "1.0"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}
{% endhighlight %}

接着看android闭包，里面首先定义了我们这个Module使用的**compileSdkVersion**和**buildToolsVersion**，这两个属性大家肯定知道，一个是用来编译代码的sdk版本，一个是用来打包apk的build-tools版本。

再看里面的defaultConfig，又定义了几个属性。依次有**applicationId**，代表着你的包名，以前我们都是在AndroidManifest.xml文件中通过`package="com.nought.hellogradle"`指定应用程序的包名，现在我们可以在gradle打包脚本中指定它，后面你会发现我们结合buildTypes和flavor，还可以动态的改变它，有点神奇了吧！ **minSdkVersion** 指的是你的应用程序兼容的最低Android系统版本； **targetSdkVersion** 指的是你的应用程序希望运行的Android系统版本；**versionCode** 是你的代码构建编号，一般我们每打一次包就将它增加1；**versionName** 则是你对外发布时，用户看到的应用程序版本号，一般我们都用“点分三个数字”来命名，例如`1.0.0`。

接着看下buildTypes，这里面默认只定义了 **release** type，其实还可以定 **debug** type。在release和debug两个type中，可以分别配置不同的选项，例如目前的release中配置了混淆文件，`minifyEnabled false`指的是不混淆代码，下面这行 `proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'` 指定的是你的混淆配置文件。这里就不详细介绍了，马上我们就会看一下多渠道打包，实践一下大家就清楚了。

{% highlight xml %}
dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    testCompile 'junit:junit:4.12'
    compile 'com.android.support:appcompat-v7:22.2.1'
    compile 'com.android.support:design:22.2.1'
}
{% endhighlight %}

最后，我们看dependencies闭包，这里指的是我们的工程依赖的库，以往在Eclipse中开发，我们常常通过jar包，以及添加library的形式来添加依赖，现在方便了，在gradle脚本里，一行代码通通搞定！真是简单啊！dependencies闭包下，有几种基本的语法。1：`compile fileTree(dir: 'libs', include: ['*.jar'])`,指的是依赖libs下面所有的jar包，你还可以指定具体的每一个jar包，而不是采用这个通配符匹配的方式，例如`compile files('libs/文件名.jar')`；2：`compile 'com.android.support:appcompat-v7:22.2.1'`，这种语法是通过包名：工程名：版本号的形式来依赖的，3：`testCompile 'junit:junit:4.12'`，指的是测试时才会用到的依赖，这里一看就知道是指做单元测试时依赖junit。

好了，上面介绍了Android Studio默认生成的基本的Gradle打包脚本的结构。下面我们在实践中学习一下怎么根据自己各种需求来实现自动化的打包。

# 多渠道打包实践

占坑。。回家接着写。。

# 版本号管理实践

占坑。。




