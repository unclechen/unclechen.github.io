---
layout: post
title: 使用前端开发利器Fiddler调试手机程序
date: '2015-04-30'
photo: '/content/images/cover/helloFiddler.jpg'
tags:
  - 调试
  - 网络
categories: 
  - 技术
---


[Fiddler](http://www.telerik.com/fiddler)是一个非常好用的web前端调试工具，它能记录客户端和服务器的http和https所有请求和响应，允许监视、设置断点，修改输入输出数据。与其他的抓包工具，例如wireshark、firebug等不同，Fiddler可以允许你在调试CGI接口时，修改返回的数据，也就是可以**构造请求**和**模拟响应**。

<!-- more -->

此外，Fiddler还可以支持模拟低速网络（如手机网络）过滤请求等等，安装了**Willow**插件以后你还可以轻松实现修改Host等操作。可惜的是目前Fiddler只支持Windows系统，没办法，毕竟是基于`.net框架`开发的嘛。

# 1. Fiddler的安装和配置

* 下载安装：从官网下载Fiddler，现在的版本应该是Fiddler4：[http://www.telerik.com/download/fiddler](http://www.telerik.com/download/fiddler)。

**注意**：由于Fiddler4是基于`.net框架`的，所以需要在自己的电脑上先安装`.NetFrameWork`，安装好了以后，就可以下载Fiddler4进行安装了。

* 配置：打开将Fiddler，在菜单中选择`Tools->Fiddler Options`，如下图所示把Fiddler设为全局的监听，再把浏览器或者软件的的http proxy设置为`127.0.0.1`，端口设为`8888`。选择ok后，关闭Fiddler并重新打开Fiddler，就可以用Fiddler抓取本地所有的流量了。

![fiddler setup](/content/images/fiddlerSetup.png)

# 2. 抓取手机数据包

抓取手机数据包和抓去电脑上的数据包一样，只需要将手机的代理设置为Fiddler。

具体操作：让手机连接的wifi和你安装Fiddler的电脑处于同一网段，然后在手机的wifi设置中，选择高级选项，设置代理，指向你电脑的ip，端口设置为8888即可。

![fiddler Wifi Setup](/content/images/fiddlerWifiSetup.png)

如上图所示，我电脑的ip是`10.4.66.135`，于是在手机连上wifi以后，勾选**`高级选项`**，**代理**选择**`手动`**，**代理服务器主机名**输入`10.4.66.135`，**代理服务器端口**输入`8888`，点击保存即可。

配置好以后，手机上所有网络请求和响应都会走Fiddler代理，这样就可以分析手机的网络流量了。我们在手机上打开一个大家熟悉的地址`www.baidu.com`，可以看到抓取的数据流量包了，Fiddler的工具栏看起来很复杂，如下图所示，稍微熟悉一下之后就会发现其实很简单。左侧界面是数据包按照时间顺序的列表，右边是对应每一个包的解析，我们可以看到详细的http header头文件以及表单、json数据等等。

![fiddler baidu](/content/images/fiddlerBaidu.png)

# 3. 修改网络响应response

有的时候我们调试程序的时候，需要服务器返回新格式的数据，或者有时候发现原来的服务器上的某个js/css文件有问题，需要修改。如果这时我们要求同事帮忙修改文件，重新发布的话，将会非常麻烦，也可能会影响到现有的线上环境。对大公司来说，这不仅效率低下，而且一不小心就可能酿成大事故。所以通常的做法是在测试环境进行修改，然后等测试通过以后，再部署到线上环境中去。

但是有了Fiddler之后，我们可以直接在本地客户端进行调试了。通过Fiddler修改HTTP数据的特性，替换服务器发给我们的回包，等本地客户端调试通过以后再确认发布。说了一堆没用的，我们直接进入实战。

使用Fiddler修改网络响应包有两种操作：

* 使用**AutoResponder**对回包进行重定向
* 使用**Willow**插件管理重定向规则

这两种操作方法是一样的，都是对服务器返回的数据包（下面简称**回包**）进行规则的设置，使得回包被替换成我们指定的文件。不过Willow插件用起来比较方便，所以我们一般都会安装Willow插件。

现在我们以Willow插件为例介绍这个非常好用的回包替换功能，我这里安装的是1.4版的Willow，支持Fiddler4.0版本。安装了Willow插件的Fiddler，在右侧的网络数据解析界面上会多出一个Willow标签菜单，如下图所示。

![fiddler willow](/content/images/fiddlerWillow.png)

从图上看出，Willow的图标是一个小树，当回包重定向功能开启时，这颗小树会变成**绿色**，普通状态下小树是**灰色**的。

在下面的列表中，**Fiddler**、**Temp 1**、**unclenought**等都是一个一个的`Willow project`，这些`project`对应的是一组一组的规则，这里我们添加一个`unclenought`的project。在Willow菜单内右键可以选择**Add Project**、**Edit Project**以及**Add Rule**等等。

![fiddler willow menus](/content/images/fiddlerWillowMenus.png)

其中我们最常用就是**Add Rule**功能了，通过这个我们可以设置一些规则，将回包进行重定向。右键选择**Add Rule**以后，我们在**Match**栏填写**正则表达式**来匹配网络请求，**Action**栏选择我们本地的一个文件来替换**match**栏对应的请求的回包,这里我选择了自己写的一个`hello fiddler.html`测试文件。记住，规则保存好了以后，**必须勾选Willow菜单左上角的小勾**，使得回包替换功能开启，确保Willow小树的图标变成了**绿色**的！

![fiddler willow rule](/content/images/fiddlerWillowRule.png)

`hello fiddler.html`文件的代码如下：

```
<!DOCTYPE html>
<html lang="zh-CN">
    <head>
        <meta charset="utf-8">
        <title>hello fiddler</title>
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
    </head>
    
    <body>
    hello fiddler
    </body>
</html>
```

此时我们在打开手机浏览器输入`m.baidu.com`以后，不会再看到正常的百度首页，而是本地文件的`hello fiddler.html`测试页面了。

![hello fiddler](/content/images/helloFiddler.png)

再回到Fiddler左侧的流量包界面，我们可以看到命中的数据包被标注为**黄色**了。因此我们判断自己定义的规则是否生效，可以看看数据包是不是被标为**黄色**了。此外由于，Fiddler回包替换的规则支持正则表达式，所以有时写的规则不一定是完全正确的，大家要多检查下rule中设置规则。

![hello fiddler](/content/images/fiddlerCatchU.png)

此外Fiddler还支持修改**Host**的功能，通过Willow插件可以一键修改，方法也是在Willow菜单下，右键点选一个project，选择**Add Host**，填写需要替换**domain**和**ip**地址即可。关于Fiddler的基本使用就介绍这些，至于断点调试等等，以后有机会再补充！
