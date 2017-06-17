---
layout: post
title: 使用React.js开发Chrome插件
date: '2017-06-16'
tags:
  - React
  - Chrome插件
  - Web
categories: 
  - 技术
---

# 一、背景

相信看到这篇文章的人应该都用过Chrome插件吧，最近刚好有个这方面的需求，我就把Chrome插件的相关知识学习了一下，发现其实Chrome插件的开发和大前端Web开发的底子是一样的，无非就是runtime只限于Chrome浏览器，并且可以调用Chrome提供的一些`chrome.*` API来实现一些基于Chrome浏览器的小功能。这里非要类比的话，我理解`chrome.*` API就像我们开发Hybird应用一样，需要有一个bridge层来提供底层原生的能力给js。我是做Android开发出生的，这只是我的个人理解，可能对大Web技术的理解还是不够。

其实Chrome上的插件，从UI上主要分成两类：一类是浏览器按钮（[BrowserAction](https://crxdoc-zh.appspot.com/extensions/browserAction)），另一类是页面按钮（[PageAction](https://crxdoc-zh.appspot.com/extensions/pageAction)）。两者的开发大同小异，我这里今天主要介绍的主角不是Chrome插件开发，而是**如何使用React.js来开发Chrome插件**，本文先简单介绍下Chrome插件的开发和ReactJS，最后介绍如何采用Facebook官方推荐的creat-react-app脚手架来开发Chrome插件。

<!-- more -->

# 二、Chrome插件开发基础知识

下面是我看的几篇教程，简单看一下应该就可以算Chrome插件速成了：

- [入门：建立 Chrome 扩展程序](https://crxdoc-zh.appspot.com/extensions/getstarted)
- [Chrome 扩展开发文档](https://wizardforcel.gitbooks.io/chrome-doc/content/)
- [Chrome扩展及应用开发](http://www.ituring.com.cn/book/miniarticle/60223)

简单来说，一个最基本Chrome插件应用需要有一个manifest.json清单文件，这个文件一般长这样：

```json
{
  "manifest_version": 2,

  "name": "One-click Kittens",
  "description": "This extension demonstrates a browser action with kittens.",
  "version": "1.0",

  "permissions": [
    "https://secure.flickr.com/"
  ],
  "browser_action": {
    "default_icon": "icon.png",
    "default_popup": "popup.html"
  }
}
```

这个文件里描述了插件应用的一些属性，如名称、版本、需要的权限、界面的对应的html文件名等等。额！！乍一看怎么和AndroidManifest.xml的**功能**这么像啊？是的大兄弟！！恭喜你对技术的理解已经融会贯通了！

根据manifest.json文件可以看到，一个Chrome插件最少得有：manifest.json文件，icon.png图标和popup.html文件。当然文件名可以随便改，只要和manifest.json里声明的一致就行。

这里就不浪费时间具体说怎么开发插件了，各路前端大牛比我强100倍。但我只强调一点，那就是popup.html中引用的js文件只能是外部引入，不能在popup.html文件里面写js代码。所以一般我们还有见到popup.js文件。另外如果你想知道自己使用的插件有什么秘密，完全可以去Chrome浏览器的安装目录下面把它们给扒出来。。

# 三、React JS基础知识

React.js不需要多说了吧，从React这个词在技术界诞生起，就是一颗明星，连我这种死抱着Native技术的人都不得不去学习它。。

简单扯两句React JS的话题（React Native下次再说），作为一个Android App/SDK开发，我没有开发过太多传统意义上的Web页面，但是经过我学习了大概一周多的时间，我发现React JS开发Web页面的思路其实和客户端很像，不去用jQuery/Zepto啊操作DOM，而是关注数据本身，以数据驱动去改变界面。重构写好了静态html后，哪块地方需要变化，你就把哪里变成一个变量放到组件的State/Props里面（至于组件怎么切分，哪个数据放State，哪个放Prop不是今天要讨论的话题），然后就只用关注数据的变化，然后setState一下界面就可以刷新了。理解了这一点，就会发现其实开发Web页面很简单。比起操作DOM，一些模板引擎之类的东西，我认为React这个思想非常容易接受，写起来也很舒服，完全没有那种混乱的感觉，而且现在ReactJS生态圈非常大，诸如Redux这类的库使得ReactJS越发的犀利，很多公司早就用得飞起了。

扯得有点远了，ReactJS开发我推荐大家就看[Facebook官方的示例](https://facebook.github.io/react/docs/hello-world.html)就够了。英文不好的朋友可以看看[阮一峰老师的博客](http://www.ruanyifeng.com/blog/2015/03/react)，或者看看[这篇入门教程](https://github.com/kdchang/reactjs101)也是阔以的。

# 四、应该用哪个脚手架？

当然是Facebook官方推荐的[creat-react-app](https://github.com/facebookincubator/create-react-app)。打开终端，依次输入：

```shell
npm install -g create-react-app

create-react-app my-app
cd my-app/
```

然后就在`my-app`下面看到这些文件了。

```shell
my-app/
  README.md
  node_modules/
  package.json
  .gitignore
  public/
    favicon.ico
    index.html
    manifest.json
  src/
    App.css
    App.js
    App.test.js
    index.css
    index.js
    logo.svg
    registerServiceWorker.js
```

到此为止，是一个标准的ReactJS编写WebApp的步骤，在终端输入`npm start`，就可以在浏览器中访问本地的localServer了。

## 1.怎么让这个项目支持Chrome插件开发呢？

前面介绍了，Chrome插件最重要的文件就是manifest.json清单文件。我们先看下脚手架给我们默认生成的manifest.json长啥样：

```json
{
  "short_name": "React App",
  "name": "Create React App Sample",
  "icons": [
    {
      "src": "favicon.ico",
      "sizes": "192x192",
      "type": "image/png"
    }
  ],
  "start_url": "./index.html",
  "display": "standalone",
  "theme_color": "#000000",
  "background_color": "#ffffff"
}
```

对于一个普通的WebApp来说，manifest.json文件在缓存、离线模式以及最新的PWA场景下会起作用，但是这里我们是要开发Chrome插件，那么把它原来的内容通通删掉，改成你的Chrome插件所需要的格式和内容就好了。例如可以改成这样：

```json
{
  "manifest_version": 2,
  "name": "MyChromeExt",
  "description": "My first chrome extension.",
  "version": "1.0.0",
  "icons": {
		"16": "img/icon-16.png",
		"128": "img/icon-128.png"
	},
  "browser_action": {
    "default_icon": {
			"19": "img/icon-19.png",
			"38": "img/icon-38.png"
		},
    "default_title": "MyChromeExt",
    "default_popup": "index.html"
  },
  "permissions": [
    "tabs"
  ],
  "background": {
    "scripts": ["background.js"]
  }
}
```

这里尽可能对脚手架的东西做最小的改动，把default_popup的文件名改成了`index.html`，因为脚手架默认会把js文件都打包到一个main.js文件中，并在index.html中插入这个main.js。

我们运行一下`npm run build`命令，就会发现生成了一个`my-app/build`目录，这个目录就是我们可以在[chrome://extensions/](chrome://extensions/)去加载的插件目录，当然也可以用Chrome把这个目录打包成一个crx插件。

使用creat-react-app脚手架开发Chrome插件的基本方法就是这样了，但是在实际中我们会遇到很多的问题，有时甚至会想要运行[npm run eject](https://github.com/facebookincubator/create-react-app/blob/master/packages/react-scripts/template/README.md#available-scripts)，然后去完全自定义`webpack.config.js`来实现打包。

## 2.background.js怎么打包？

我们在开发插件的时候，非常可能需要用到后台的background.js，原因如下：

> 注意：不要在popup页面的js空间变量中保存数据。由于popup页面只在用户点击图标时才会开启，当用户关闭这个页面时就会停止，并没有一个从始至终的实例分配给popup页面。所以每当用户打开popup页面时，它都是崭新的，之前保存在变量中的数据都会消失。如果需要通过popup页面保存用户的数据，可以通过通信将数据交给后台页面（background页面）处理，或者通过localStorage和chrome.storage将数据保存在用户的硬盘上。


所以background.js最后也是要进入到我们的发布文件夹下面的，这里建议还是要坚持**最低程度地修改**脚手架的设置，建议不要npm run eject之后来修改webpack的配置，因为实在是真的有点复杂。

这次修改下`package.json`文件：

```json
{
  "name": "my-app",
  "version": "0.1.0",
  "private": true,
  "devDependencies": {
    "react-scripts": "1.0.7"
  },
  "dependencies": {
    "react": "^15.6.1",
    "react-dom": "^15.6.1"
  },
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test --env=jsdom",
    "eject": "react-scripts eject",
    "build-chrome-ext": "react-scripts build && cp src/background.js build/background.js"
  }
}
```

可以看到我们添加了一个命令`npm run build-chrome-ext`，并把background.js丢到了build目录下。如果你还有其他的js，我建议在`my-app/src`下建立一个`my-app/src/chrome`文件夹，专用于存在chrome相关其他js代码，然后在build的时候统一丢到build里面。如果要minify这些js，同样可以采用`&&`方式去添加命令。修改

## 3.需要注意的细节

由于使用了一些`chrome.*` API，我们需要在编译js的时候将`chrome`这个全局对象声明一下。

creat-react-app这个脚手架在**非eject模式**下，没办法修改ESlint的配置来添加global对象，只能在用到了 `chrome.*` API的代码处添加 `// eslint-disable-line` 注释来实现保证编译通过。

如果你已经`npm run eject`了，在**eject模式**下，可以在`package.json`文件里配置ESLint：

```
"eslintConfig": {
"extends": "react-app",
"globals": {
  "chrome": true
  }
}
```


# 五、其他脚手架推荐

除了自己改造Facebook推荐的[creat-react-app](https://github.com/facebookincubator/create-react-app)外，下面两个脚手架也算是用户比较多的，专门用于开发Chrome插件的脚手架。

- [https://github.com/jhen0409/react-chrome-extension-boilerplate](https://github.com/jhen0409/react-chrome-extension-boilerplate)：默认支持ReactJS，基于webpack。
- [https://github.com/yeoman/generator-chrome-extension](https://github.com/yeoman/generator-chrome-extension)：没有默认支持ReactJS，需要自己修改，基于gulp打包。




