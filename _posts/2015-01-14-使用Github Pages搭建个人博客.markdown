---
layout: post
title: 使用GitHub Pages搭建博客（上）
---

新的一年开始，让我们做点美好的事情。[GitHub](https://github.com/) 是一个开源项目的托管网站，相信很多人都听过。在上面有很多高质量的项目代码，我们也可以把自己的项目代码托管到GitHub，与朋友们共享交流。

[GitHub Pages](https://pages.github.com/) 是Github为大家提供的一项服务，不仅能为托管的项目建立主页，还可以为我们建立个人博客。下面我就分别介绍个人博客和项目主页是如何建立的。这里我们先假设大家已经了解Git一些基本使用，将来我也会发布另外一篇Git基本安装和使用教程。

在使用GitHub Pages建立个人博客，我们还是简单了解一下GitHub Pages建立的页面有哪些优点：

- 极简的配置，轻量级系统，无需数据库
- 使用标记语言，如Markdown
- 使用GitHub托管服务，免费300MB空间
- 可以绑定自己的域名
- 新版的GitHub Pages支持CDN服务


要说缺点的话，其实也是有一些的，例如：

- 使用[Jekyll](https://github.com/jekyll/jekyll) 模板系统，属于静态页面。
- 基于Git操作，需要有一定的动手能力。
- 动态性不好，没有评论系统，不过可以自己配置[Disqus](https://disqus.com/) 扩展。

总得来说，使用起来是非常简单的。配置好了之后，只需要使用例如Markdown编写自己的博客内容，扔到文件夹里面即可发布了。完全不需要管理网站相关的事务，例如数据库、安全性等问题。

开始正题，下面的参考来自于[Github Pages](https://pages.github.com/)的官方网站，再结合我自己的实际操作介绍各个步骤，并说一下可能遇到的问题和解决办法。

首先说明下，下面以Window 7环境为例，介绍操作步骤。至于OS X下，我操作的时候感觉还更方便一些。。另外我采用的是Git Bash终端来操作，这在任何系统下是一样的。

## 创建一个新的项目托管仓库 （Create a repository）

![](/content/images/Create a repository.png)

如图所示，登录自己的GitHub主页，从右上角新建一个仓库，仓库名（`Repository name`）设置为你自己的用户名为前缀，例如我的是`unclechen.github.io`。

在GitHub上，每个用户只允许拥有唯一的一个以个人用户名为前缀，且符合`username.github.io`命名的仓库。这个仓库也就是Github Pages说的个人/组织主页（`User or organization site`）。因此当我再次想要建立一个同样名字的仓库时，上图中显示我的账号下已存在同名的仓库了。


## 把仓库拉到本地 （Clone the repository）

![](/content/images/Clone the repository.png)

使用任何一种方式将刚才建立的仓库拉取到本地。如上图所示，我喜欢使用Git Bash终端，在Git Bash中输入`git clone git@github.com:unclechen/unclechen.github.io.git`，即可将刚才建立的仓库拉取到本地。


## 建立index.html文件 （Hello World）

![](/content/images/Create index.html.png)

如上图所示，继续在Git Bash中输入`cd unclechen.github.io/`命令，进入刚才拉取的仓库。然后在Git Bash中继续输入`echo "Hello World" > index.html`命令，建立index.html文件。


## 提交到GitHub服务器

Let's get back to the new Lollipop functionality. As mentioned you can now specify a theme directly onto a View in your layout.  The most common use of this will (probably) be using Toolbar:

{% highlight xml %}
<Toolbar  
    android:layout_height="?android:attr/actionBarSize"
    android:layout_width="match_parent"
    android:background="?android:attr/colorPrimaryDark"
    android:theme="@android:style/ThemeOverlay.Material.Dark.ActionBar" />
{% endhighlight %}

Hopefully you can now see how it all comes together. We have now made the Toolbar have a dark theme, ensuring that it's content is light in color and has contrast againt the dark background.

One thing to note is that `android:theme` in Lollipop propogates to all children declared in the layout:

{% highlight xml %}
<LinearLayout
    android:theme="@android:style/ThemeOverlay.Material.Dark">
    
    <!-- Anything here will also have a dark theme -->
    
</LinearLayout>
{% endhighlight %}

Your children can set their own theme if needed.

## Example

Let's wrap this up with a question that I've been asked recently:

> How do I set android:colorEdgeEffect so that it only takes effect on a single view?

The `colorEdgeEffect` attribute is a new theme attribute in Android 5.0, which is used to customize the color of the overscroll effect for lists, etc.

As this is a theme attribute you can not just set it directly on the view. Instead we need to use `android:theme` with a custom ThemeOverlay. Our custom overlay just sets `android:colorEdgeEffect` to be red. We then set this theme on to the view so that it takes effect.

**res/values/themes.xml**

{% highlight xml %}
<style name="RedThemeOverlay" parent="android:ThemeOverlay.Material">
    <item name="android:colorEdgeEffect">#FF0000</item>
</style>
{% endhighlight %}

**res/layout/fragment_list.xml**

{% highlight xml %}
<ListView
    ...
    android:theme="RedThemeOverlay" />
{% endhighlight %}

Just to note, colorEdgeEffect was just an example here, this technique can be used with **all** theme attributes.

## Theme vs Style

So what exactly is the difference? Well they are both declared in exactly the same way (which you already know), the difference comes in how they're used.

Themes are meant to be the global source of styling for your app. The new functionality doesn't change that, it just allows you to tweak it per view.

Styles are meant to be applied at a view level. Internally, when you set `style` on a View, the LayoutInflater will read the style and apply it to the [AttributeSet](https://developer.android.com/reference/android/util/AttributeSet.html) before any explicit attributes (this allows you to override style values on a view).

Values in an attribute set can reference values from the View's theme.

TL;DR: Themes are global, styles are local.

## AppCompat

So how does AppCompat fit into this? Well it obviously backports some of the new color theme attributes.

It also backports the `android:theme` functionality for certain widgets, currently **only** `android.support.v7.widget.Toolbar`.

See my [AppCompat v21](https://chris.banes.me/2014/10/17/appcompat-v21/) blog post for more information.