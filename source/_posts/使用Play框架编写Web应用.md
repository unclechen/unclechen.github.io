---
layout: post
title: 使用Play框架编写Web应用
date: '2018-05-13'
tags:
  - Java Web
  - 后端
categories:
  - 技术
---

## 一、Play框架简介

[Play](https://www.playframework.com/)是一个Full-Stack的Web应用开发框架，使用它可以快速编写自己的Web应用，也可以使用它来编写RESTful API。与现在非常流行的Spring全家桶相比，Play略显小众，但它的设计思想天生就是分布式、异步的，也得到许多开发者的认可，在实际生产环境中也有像Linkedin这样的大公司采用。对于一个没有开发过Web应用或者后台应用的开发者来讲，学习和使用Play框架也许是一个不错的选择。

<!-- more -->

可能有些接触过Scala语言的开发者，或多或少听过Play这个框架，也知道Play框架大部分代码是使用Scala开发的。不禁会有疑问，是不是一定要学会Scala才可以使用Play呢？这个疑问我也有过，但是据使用过Play的开发者介绍，在Play框架中使用Java语言开发Web应用的过程中，基本上是不会需要你真正去学习Scala的。Play框架本身对Java的支持非常全面，不用担心自己不会Scala。我举一个例子，做过Android开发的同学，一定记得在2014年，我们从Eclipse切换的Android Studio时，需要将自己的ant.xml打包脚本移植到gradle脚本。当时我还在实习，完全是一个菜鸟，也不懂groovy，而且那时候网上关于gradle的资料也非常的少，但是即使这样，我依然能够把一个ant编写的打包脚本移植到gradle，如此类比一下，相信大家应该心里有数了。

## 二、环境搭建

我使用的Play版本是2.6.x，系统是macosx 10.12.5。配置环境的时候，推荐按照[官方文档](https://www.playframework.com/documentation/2.6.x/Home)操作。

### JDK

Play 2.6.x要求必须是Java 8，现在使用到的SBT版本是1.x。

### SBT

SBT(simple-build tool)，Play默认使用的构建工具。可以用homebrew安装，也可以手动下载sbt程序到电脑。

> 提醒一下，我在mac osx 10.12上运行`brew install sbt@1`时，提示需要安装9.x版本的xcode，才可以用brew安装sbt。所以我采用了手动下载sbt，然后配置环境变量的方法，使得电脑上可以使用`sbt`命令。但是请记住一点，手动安装的sbt，目录是你自己放的文件目录，而不是默认的`usr/local/sbt`，如果你需要修改`sbtopt`这种配置文件的话，记得在自己正确的目录下操作。

### Intellij IDEA

安装IDEA后，还需要安装Scala插件。关于IDE的设置，请参考[这里](https://www.playframework.com/documentation/2.6.x/IDE)。

> 尽量安装新版的IDEA和对应版本的Scala插件，亲测2017.2的IDEA安装插件后可能会有BUG。

## 三、认识Play项目的结构

Play框架的文档很全面，如果要全面深入地学习Play，可以仔细阅读官方的Document。这里可以从Play的示例[play-java-starter-example](https://github.com/playframework/play-java-starter-example)入手，开始认识Play框架的结构，并了解如何使用Play来开发Web应用。

使用idea的`Import Project`导入play-java-started-example工程，记得在import wizard中选择`Import project from external model`的`SBT project`，然后点击下一步。导入项目后，进入终端输入`sbt run`就可以让这个项目运行起来了。

> 注意：sbt第一次运行时比较慢，需要耐心等待。

根据console的提示，在浏览器中打开`http://localhost:9000`，我们可以看到一个Web欢迎页面。

---

搞定上面的准备工作之后，我们开始看Play应用的目录是如何组织起来的，进而分析一下我们在浏览器中输入`http://localhost:9000`之后，Play是如何工作的。

下面是来自[Play官方文档中的Anatomy of a Play application](https://www.playframework.com/documentation/2.6.x/Anatomy)。

```
app                      → Application sources
 └ assets                → Compiled asset sources
    └ stylesheets        → Typically LESS CSS sources
    └ javascripts        → Typically CoffeeScript sources
 └ controllers           → Application controllers
 └ models                → Application business layer
 └ views                 → Templates
build.sbt                → Application build script
conf                     → Configurations files and other non-compiled resources (on classpath)
 └ application.conf      → Main configuration file
 └ routes                → Routes definition
dist                     → Arbitrary files to be included in your projects distribution
public                   → Public assets
 └ stylesheets           → CSS files
 └ javascripts           → Javascript files
 └ images                → Image files
project                  → sbt configuration files
 └ build.properties      → Marker for sbt project
 └ plugins.sbt           → sbt plugins including the declaration for Play itself
lib                      → Unmanaged libraries dependencies
logs                     → Logs folder
 └ application.log       → Default log file
target                   → Generated stuff
 └ resolution-cache      → Info about dependencies
 └ scala-2.11
    └ api                → Generated API docs
    └ classes            → Compiled class files
    └ routes             → Sources generated from routes
    └ twirl              → Sources generated from templates
 └ universal             → Application packaging
 └ web                   → Compiled web assets
test                     → source folder for unit or functional tests
```

Play框架采用了MVC架构，把Web应用分成模型层、控制层和视图层。每个层次对应的文件存放在不同的目录下面，下面依次介绍一下关键的几个目录。

### `app/`目录

存放着所有的Java源代码代码、Scala源代码、模板和编译后的资源文件（如LESS CSS、CoffeeScript）。在这个目录下，一般会有三个默认的package：controllers、models、views。

> 和以往我们看到的代码package不同的是，play默认的package layout没有`com.youcompany`这样的前缀。不过我们如果想添加前缀也完全是OK的。包括`app/assets`目录，完全也是可选的。

### `public/`目录

`public/`目录下面存放的是一些静态资源文件，这个目录默认被分成3个子目录，分别用来存放我们的js、css、image这3类文件。

> 在一个新创建的应用中，`public/`目录默认会被映射到`assets`这个URL路径下，需要的话我们也可以手动修改。

### `conf/`目录

顾名思义，这个目录下面放的是配置文件，Play中主要用两类配置文件：

- `application.conf`：整个应用的配置参数，例如db连接参数、缓存策略等配置
- `routes`：路由定义


### `build.sbt`文件

项目的构建脚本，主要的构建配置都在这里，例如项目依赖的jar包、应用的版本号等等。不过`project/`目录下的`.scala`文件也会对项目的配置起作用。这个文件有点类似于我们的`build.gradle`。

### `lib/`目录

这个目录用于存放一些以来的外部jar包等等，是一个可选目录。不过一般我们都可以在`build.sbt`中添加依赖。

### `project/`目录

前面就说到这个目录下的scala文件也会对项目配置起作用。这里包含两个sbt构建配置文件：

- `plugin.sbt`：定义了需要用到哪些sbt插件。我感觉有点类似于`build.gradle`里面写的`apply 'idea'`。
- `build.properties`：定义了sbt的版本。我感觉有点类似于的`gradle-wrapper.properties`。

### `target/`目录

这个目录存放着工程构建完以后生成的文件，从这里可以了解到代码经过sbt构建后，最终变成了什么样的结果。主要包括下面几个子目录：

- `classes/`：编译后的class文件（来自Java和Scala源码）
- `classes_managed`：`classes/`目录下组织好的其他子目录，包含框架生成的class文件，例如routes和template引擎生成的class文件。
- `resource_managed`：组织好的、生成的资源，比如编译过的LESS CSS、CoffeeScript结果。
- `src_managed`：组织好的生成的代码文件，例如模板引擎生成的Scala文件。
- `web/`：sbt-web任务生成的资源文件，例如来自`app/assets`和`public`文件夹里面的文件。

> Play应用默认的文件组织结构与SBT默认的并不相同。如果我们想用SBT默认的代码和文件组织结构，可以禁用掉PlayLayoutPlugin。但是这有一定的风险。

通过上面应用组织结构，可以看到其实Play框架采用的也是**约定优于配置**的规范，让我们集中精力在程序的开发上面，而不是去写太多的配置文件。


## 四、使用Play开发Web应用

在上面我们运行`sbt run`命令后，可以访问[http://localhost:9000](http://localhost:9000)来访问这个示例程序，看到了一个Welcome Page。

Play框架是一个典型的MVC架构，下面分析一下这个示例工程是工作起来的，从而了解怎么使用Play开发Web应用。

### 定义路由和Controller

前面介绍了`conf/routes`文件定义了整个应用的路由，也就是说我们在浏览器输入的url（request），经过这个文件的映射，会交给相应的Controller处理，然后返回结果给浏览器。看一下`routes`文件的内容：

```
# Routes
# This file defines all application routes (Higher priority routes first)
# ~~~~

# An example controller showing a sample home page
GET     /                           controllers.HomeController.index

# 省略其他路由配置
... 
```

所以当访问`http://localhost:9000`时，会交给`app/controllers/HomeController.index()`方法来处理这个请求，并得到结果返回给浏览器。

### 模板引擎

下面看一下`HomeController.index`方法是如何渲染出Html页面来的。

```
public Result index() {
    return ok(index.render("Your new application is ready."));
}
```

这里调用了`ok`方法来返回一个`Result`对象，我们仔细分析一下这个方法的签名。

先看`ok`方法的返回值类型`Result`，在Play中，`Result`类可以理解为一个定义了http响应的数据结构，包含了响应的HTTP Status Code和的content。那`ok`方法是哪里定义的呢？原来`HomeController`继承了`Controller`类，`Controller`类继承了`Results`类，而`ok`方法是`Results`类里面定义的，这个方法可以返回一个Result对象。

再看`ok`方法的参数，这个参数是`Content`类，它定义了响应的contentType和body。这里的Content是由`index.render`方法生成的。

接着我们再看`index.render`是如何生成的`Content`的，这里的`index`类，并不是我们写的，而是Play生成的，使用idea可以很容易找到它在`target/scala-2.12/twirl/main/views/html/index.template.scala`这个scala文件里，这个文件定义了一个scala里面的单例（我们可以暂时不要纠结这个Scala文件里面`object index ....`是怎么生成一个单例对象的，只需要知道这个文件是从`app/views/index.scala.html`生成的）。好了，我们知道`index`对象是Play生成的，看看它的`render`方法，这个方法接收一个`String类型`的参数，然后返回了前面的`Content`。

所以我们浏览器展示Html的是`app/views/index.scala.html`这个模板渲染出来，这是[Play模板引擎](https://www.playframework.com/documentation/2.6.13/ScalaTemplates)的工作了，我们看下里面有什么内容。

```
@*
 * This template takes a single argument, a String containing a
 * message to display.
 *@
@(message: String)

@*
 * Call the `main` template with two arguments. The first
 * argument is a `String` with the title of the page, the second
 * argument is an `Html` object containing the body of the page.
 *@
@main("Welcome to Play") {

    @*
     * Get an `Html` object by calling the built-in Play welcome
     * template and passing a `String` message.
     *@
    @welcome(message, style = "java")

}
```

Play内置了基于Scala语言的Twirl模板引擎，跟着注释就可以了解这个模板是怎么被渲染成我们看到的html的。

首先`index`模板接收了一个字符串作为参数（`Your new application is ready.`），然后把两个参数传递给了`main`模板，一个是字符串`Welcome to Play`，另一个是调用`welcome`模板生成的`Html`对象。`welcome`模板接收了`Your new application is ready.`参数和`style="java"`参数，然后渲染成了Html对象。打开`Welcome.scala.html`可以看到一些Html标签以及传入的第一个参数被放在了最上方的<section>里面。

> 我们可以看到其实Play框架帮我们生成的很多文件，都是基于Scala的，因此学习一下Scala对我们理解Play框架非常有帮助。


### 小结

以上就是从play-java-start-example这个示例工程来了解如何使用Play开发Web应用，这个示例比较简单，甚至没有涉及到model层的代码。开发一个完整的应用，还需要学习更多知识，比如[filters拦截器](https://www.playframework.com/documentation/2.6.x/Filters)、异步处理、状态保持、如何集成ORM框架、支持WebSocket、编写RESTFul APIs、安全以及使用其他的模板引擎等等。

## 五、其他话题

### Play框架与React集成

现在前端开发中有很多都转向了React，我们也可以使用Play框架和React一起编写Web应用。毕竟模板引擎这么多，如果如果已经会使用React开发Web前端，没有必要非得用到Play里面的模板引擎。因此使用React替换掉Play的模板引擎做的工作，是完全没有问题的。

### 相关资料

- [官方文档Home](https://www.playframework.com/documentation/2.6.x/Home)
- [官方的Tutorials](https://www.playframework.com/documentation/2.6.x/Tutorials)
- [Play Starter Examples](https://playframework.com/download/#starters)
- [Play工程的结构剖析](https://www.playframework.com/documentation/2.6.x/Anatomy)

### 全栈开发

关于全栈，一个对技术有追求，乐于学习进步的人，不应拘泥于只学习某一项技能。有机会要多去熟悉一些其他的技术，做一个“T型”人才，全栈开发是很好的一个实践。除了像Play这样的框架，Ruby on Rails、Python Django，甚至NodeJS，都可以值得了解和学习的全栈框架。

我认为在学习某一个方向的技术时，需要学习它里面编程的思维方式，并找到一个方向上不同技术之间的共同点，比如学习编程语言的时候，我们都会学习数据类型、流程、异常处理等等；客户端或者前端，核心的基础知识是图形界面开发，从大学时候学的MFC，到现在的Android、iOS、小程序、快应用、React、Vue、Flutter，都需要处理界面的生命周期，比如`初始化 -> 展现 - > 消失 -> 销毁`；而后端开发，一般都是在处理请求的生命周期，即把一个request变成一个response，比如`路由 -> 处理请求参数 -> 处理响应 -> 返回给客户端`。希望大家都可以学习进步，走在技术时代的前面！

最后放上两个我个人觉得不错的学习资料，来自极客时间（InfoQ下面的极客帮面向广告互联网从业者做的一个应用，里面有一些老司机的分享，也有一些付费的知识分析）。最近我看到两个不错的、体系化的分享，与大家分享一下，下面是我的邀请链接，我已经开始学习，觉得帮助还是挺大的。

![](https://ws3.sinaimg.cn/large/006tKfTcly1fr9zxjidkgj31gi1a64bg.jpg)



