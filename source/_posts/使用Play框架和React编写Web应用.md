---
layout: post
title: 使用Play框架和React编写Web应用
date: '2018-05-20'
tags:
  - Java Web
  - 后端
  - React
categories:
  - 技术
---

## 一、背景

上一篇文章提到[使用Play框架编写Web应用](http://unclechen.github.io/2018/05/13/%E4%BD%BF%E7%94%A8Play%E6%A1%86%E6%9E%B6%E7%BC%96%E5%86%99Web%E5%BA%94%E7%94%A8/)，Play框架内置了模板引擎，支持MVC架构，但这本质还是一种Server Rendering。现在越来越多的网站（尤其是不需要seo的一些商业平台系统），都在变成SPA（Single Page Application），使用React、Vue、Angular进行开发。我们先不讨论哪种方式更好，只看看它们到底是怎么做的。

<!-- more -->

## 二、环境搭建

扯了这么多，下面看一下如何配置我们的Play工程，来使得我们可以方便编写ReactJS代码构建WebApp，同时方便地把Java代码和我们的前端代码打包到一起，进行发布。

### Play

我使用的Play版本是2.6.x，关于这个Play的Starter工程，请参考前一篇文章[使用Play框架编写Web应用](http://unclechen.github.io/2018/05/13/%E4%BD%BF%E7%94%A8Play%E6%A1%86%E6%9E%B6%E7%BC%96%E5%86%99Web%E5%BA%94%E7%94%A8/)，我们就以这个工程作为基础。

### React

> 关于如何使用React开发Web应用，我觉得只要你稍微懂一点js和html，你就可以对着React官方的教程开始编写应用。因为React支持ES6标准去写代码，而ES6已经具备一个主流的高级编程语言该有面向对象、FP（Functional Programing）等特性，简单来说就是对于有过类似Java编程基础的人来说非常友好。

之前我也有一篇博客介绍了[使用React开发Chrome插件](http://unclechen.github.io/2017/06/16/%E4%BD%BF%E7%94%A8ReactJS%E5%BC%80%E5%8F%91Chrome%E6%8F%92%E4%BB%B6/)，里面提到创建一个React应用最好的脚手架工具就是Facebook官方的[Create-React-App](https://github.com/facebook/create-react-app)，这里我们还继续用它，关于这个脚手架的用法和React工程配置的基础知识，这次暂时略过。如果想深入了解这个脚手架的用法，请参考[https://github.com/facebook/create-react-app](https://github.com/facebook/create-react-app)。


## 三、整合React代码和Play工程的代码

**下面说正题，现在我们已经会写Play代码，也会写ReactJS代码了。要怎么组织我们的代码结构，才可以使得React的JS代码和Play的Java代码整合起来呢？**

### 1.新建一个React项目

我们进入命令行，打开之前的Play工程，然后输入命令创建一个react-app：

```shell
cd play-java-starter-example
npx create-react-app web-app
cd web-app
npm start
```

此时，我们的工程目录下面就多出了一个`web-app`的文件夹，我们就在这个文件夹里面编写JS代码。

运行上面的命令之后，控制台里面会提示我们可以访问[localhost:3000](http://localhost:3000)。

### 2.配置development server的proxy

我们知道上面启动React程序时，其实没有启动后端的服务器，而是脚手架里面默认配置好的一个Webpack development server。而前面我们知道Play框架启动的服务器监听的是**9000**端口，并非是**3000**端口。而且这两个程序完全是两个进程，通常是没法监听一个端口的。

所以为了在开发环境下，方便我们同时编写前端的JS代码和后端的Java代码，我们需要把development server接收到的请求转发到启动的Play项目的那个Server去。

我们需要修改`/web-app/package.json`文件，添加下面这行配置：

```json
"proxy": "http://localhost:9000/"
```

### 3.启动Play项目作为后端服务器

这里还是和前一篇博客一样，我们运行`sbt run`启动了一个Server，监听着`9000`端口的请求。我们在开发的时候，一个request会经过Webpack dev server的代理，走到我们用Play编写的API Server。

![Webpack dev server PROXY](https://ws3.sinaimg.cn/large/006tKfTcly1frq2914y76j30rs051wga.jpg)

如此，我们就可以不用使用Play内置的模板引擎去编写Web App了，完全可以使用我们已经学会的ReactJS去写应用。

**这里必须强调的一点是：由于我们分离了JS代码和Java代码，JS使用的框架和Webpack开发环境支持热加载，我们在修改JS代码后，浏览器会自动刷新应用到最新的JS代码；Java使用的框架是Play，这个框架编写程序的时候也可以支持热加载，我们在修改Java代码后，当有请求过来时，也会重新编译最新的Java代码并应用它们。这可以使得我们的开发效率得到极大的提升！**

### 4.验证效果

现在我们已经分别启动了React应用和Play编写的后端应用，恰好这个Play的Starter工程下面还有一个`CountController.java`，负责处理`routes`中定义的`localhost:9000/count`请求，我们在JS中发一个请求到这里试试效果。打开`/web-app/src/App.js`，添加一个方法：

```
componentDidMount() {
  fetch('/count')
  .then(response => {console.log(response)})
}
```

在这里我们向`localhost:3000/count`发送了请求，由于我们配置了Webpack Dev server的`proxy`，请求会被转发到`localhost:9000/count`。

然后打开Chrome浏览器，右键打开`inspect`，进入`network`，然后在地址栏输入`http://localhost:3000`，我们可以看到chrome调试器的network记录中出现了下面的请求，并且返回了`200 OK`。

![chrome inspection proxy](https://ws3.sinaimg.cn/large/006tKfTcly1frq2os3885j31e00ltdpe.jpg)


通常我们和后端接口通信用的都是JSON协议，这里我们也可以改造一下这个Controller。参考[https://www.playframework.com/documentation/2.6.13/JavaJsonActions](https://www.playframework.com/documentation/2.6.13/JavaJsonActions)，我们把`count`方法改成下面这样：

```java
public Result count() {
    ObjectNode result = Json.newObject(); // play框架使用的是Jackson处理json
    result.put("code", 0);
    result.put("message", "OK");
    result.put("data", counter.nextCount());
    return ok(result);
}
```

即可返回JSON格式的response了。

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

{"code":0,"message":"OK","data":1}
```

### 4.打包和发布

上面讲的是开发模式里我们怎么配置代码的结构，其实`web-app`这个目录本质上放在任何一个地方都可以，尤其是那种JS和Java开发完全是不同的人在负责的团队。但是有些团队（比如我所在的团队），写JS的和写Java的就是一个人或者这个团队里的人，同时负责这两份代码，那么把JS代码和Java代码放在一起还是更好一些。

打包时，我们首先应该打包JS代码，create-react-app已经为我们准备了一个命令`npm run build`，即可把我们的js代码、css代码和index.html静态文件统一打包输出在`/web-app/build`目录。然后我们再把Java代码打包，在Play工程里面，运行`sbt package`，也可以把程序打包成一个可执行的jar包。

分别打包JS代码和Java代码之后，我们可以根据自己的需要，把它们放在同一个服务器上或者不同的路径下甚至是不同的服务器上，这里本质上其实是把两个程序在物理上分离了。其实还有一种方式就是把js、css和index.html这种静态文件也一起打入到jar里面，然后统一由这个jar来提供访问，这时在物理上两个程序是一起部署的。


## 四、小结

上面介绍了使用Play框架做后端 + React框架做前端来编写一个Web App。但通常在一个更大型的业务系统中，这里介绍到东西只能算是前端(Web、Native App) -> 中台，一般这个中台还会依赖一个更加纯粹的后台Service。中台代码负责处理请求校验和一些缓存，然后中台再向后台请求，后台的代码会操作数据库。有的团队（比如我所在的团队），在分工方面，前端和中台是一个人在负责，后台是一个人在负责。

本质上这里我们其实就是使用Play框架编写了一个后台API而已，其实用其他的框架也是一样OK的，比如SpringBoot，这里也有一篇文章介绍[https://spring.io/guides/tutorials/react-and-spring-data-rest/](https://spring.io/guides/tutorials/react-and-spring-data-rest/)。

下面是两篇参考资料：

- [http://ticofab.io/react-js-tutorial-with-play_scala_webjars/](http://ticofab.io/react-js-tutorial-with-play_scala_webjars/)
- [https://www.fullstackreact.com/articles/using-create-react-app-with-a-server/](https://www.fullstackreact.com/articles/using-create-react-app-with-a-server/)
