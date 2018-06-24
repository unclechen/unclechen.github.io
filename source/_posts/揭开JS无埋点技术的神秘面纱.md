---
layout: post
title: 揭开JS无埋点技术的神秘面纱
date: '2018-06-24'
tags:
  - JS
  - 埋点
  - 无埋点
  - 前端
categories: 
  - 技术
---


## 一、背景

相信很多人都接触过**“埋点”**这个概念，无论是前端还是后端开发，我们都可以使用这门技术来生产出一些运营性质的原始数据（接口耗时、程序安装/启动、用户交互行为等等），然后分析它们得到一些抽象指标（例如留存率、转化率），进而决定产品运营或者代码优化的方向。现在业界有许多比较知名数据平台，比如Google Analytics、Facebook Pixel、Mixpanel、GrowingIO、诸葛IO、TalkingData、神策数据等数不胜数一大票，这些平台有单纯做数据分析的，也有服务于特定领域例如广告监测转化的，都提供了多端（Android、iOS、Web、小程序、ReactNative）的埋点SDK和比较全面的BI服务。这一两年，不少平台都开始宣传一种叫**“无埋点”**的技术，下面以Web端为例，揭开它的神秘面纱。

<!-- more -->

## 二、什么是无埋点？

**“无埋点”**在国外一些平台被叫做`Codeless Tracking`，顾名思义就是可以写“更少”的埋点代码。而**“代码埋点”**一般需要开发人员编写代码，监听某个html元素的产生的事件，然后调用上报数据的接口，发送数据。而无埋点则可以由非技术人员（例如运营、产品），在可视化的工具中作出配置，然后就可以将html元素中产生的行为上报到后台。下面是Mixpanel平台的可视化工具的截图。

![](https://ws1.sinaimg.cn/large/006tNc79gy1fsl72i76vnj30tk0d8dhs.jpg)

在这个工具里，需要首先输入页面的url，页面加载完成后，会出现可视化配置的工具条。点击创建事件，就可以进入元素选择模式，用鼠标点击页面上的某个元素（例如button、a这些element），就可以在弹出的对话框里面，设置这个事件的名称（比如叫`TEST`）。保存这个配置之后，如果页面在浏览器中被浏览，刚才配置的那个按钮发生点击时，就会向后台上报一个`TEST`事件。我们还可以设置上报`TEST`事件的时候，带上一些属性（properties），这些属性同样也是在页面中用鼠标去选择，然后保存起来的。

看到这里，首先从产品层面上，我们比较具体的了解到“无埋点”到底是干什么的了，无埋点就是用可视化工具配置页面中需要被监测的元素，并设置这个元素产生行为的时候需要上报的数据。**但是还有非常关键的一点必须提到，要让“无埋点”工作起来，页面里面还是必须嵌入了一段JS SDK的基础代码，只是不需要再去调用SDK具体的数据上报接口罢了。**

**所以，“无埋点”技术的关键是：**

- **操作可视化配置工具，保存配置**
- **SDK基础代码如何根据配置上报行为**

下面介绍一下如何实现这两个关键。

## 三、关键技术

### 1. 基础代码

**和代码埋点一样**，要让“无埋点”工作起来，网页里也必须有一段“基础代码”。

```js
<!-- start Mixpanel --><script type="text/javascript">(function(e,a){if(!a.__SV){var b=window;try{var c,l,i,j=b.location,g=j.hash;c=function(a,b){return(l=a.match(RegExp(b+"=([^&]*)")))?l[1]:null};g&&c(g,"state")&&(i=JSON.parse(decodeURIComponent(c(g,"state"))),"mpeditor"===i.action&&(b.sessionStorage.setItem("_mpcehash",g),history.replaceState(i.desiredHash||"",e.title,j.pathname+j.search)))}catch(m){}var k,h;window.mixpanel=a;a._i=[];a.init=function(b,c,f){function e(b,a){var c=a.split(".");2==c.length&&(b=b[c[0]],a=c[1]);b[a]=function(){b.push([a].concat(Array.prototype.slice.call(arguments,
0)))}}var d=a;"undefined"!==typeof f?d=a[f]=[]:f="mixpanel";d.people=d.people||[];d.toString=function(b){var a="mixpanel";"mixpanel"!==f&&(a+="."+f);b||(a+=" (stub)");return a};d.people.toString=function(){return d.toString(1)+".people (stub)"};k="disable time_event track track_pageview track_links track_forms register register_once alias unregister identify name_tag set_config reset opt_in_tracking opt_out_tracking has_opted_in_tracking has_opted_out_tracking clear_opt_in_out_tracking people.set people.set_once people.unset people.increment people.append people.union people.track_charge people.clear_charges people.delete_user".split(" ");
for(h=0;h<k.length;h++)e(d,k[h]);a._i.push([b,c,f])};a.__SV=1.2;b=e.createElement("script");b.type="text/javascript";b.async=!0;b.src="undefined"!==typeof MIXPANEL_CUSTOM_LIB_URL?MIXPANEL_CUSTOM_LIB_URL:"file:"===e.location.protocol&&"//cdn4.mxpnl.com/libs/mixpanel-2-latest.min.js".match(/^\/\//)?"https://cdn4.mxpnl.com/libs/mixpanel-2-latest.min.js":"//cdn4.mxpnl.com/libs/mixpanel-2-latest.min.js";c=e.getElementsByTagName("script")[0];c.parentNode.insertBefore(b,c)}})(document,window.mixpanel||[]);
mixpanel.init("46042714e64a7536dde6f02af1aec923");</script><!-- end Mixpanel -->
```

上面是Mixpanel平台的基础代码，不同平台家的这段**基础代码**，大同小异，都是一段IIFE形式的、压缩过的js代码，执行完成之后，在head里面插入了一个新的script标签，**异步**去下载真正的核心SDK代码下来工作。所以并不是基础代码可以根据配置上报行为，而是基础代码会下载一段**“更大”**的SDK核心代码，这段代码才是SDK真正的功能实现。

**这样子做的好处是，基础代码很短，加载的时候不会影响到网页的性能，而且核心SDK代码的更新也不需要用户去更新这段基础代码。**

### 2. 页面的唯一标识

在配置元素行为的时候，需要唯一标识一个页面，这样才能保证A页面的配置，不会下发给在B页面，不会导致B页面产生出A页面里配置的行为。在Web里面标识页面靠的是url，url由protocol、domain、port、path和参数组成，存储配置的时候要将url的参数提出来再存。而url的参数位置是可以变化的，比如urlA（`http://a.b.com/c.html?pa=1&pb=2`）和urlB（`http://a.b.com/c.html?pb=2&pa=1`）虽然`urlA ！== urlB`，但是其实它们是一个页面。

### 3. 元素的唯一标识

唯一标识页面后，接下来就要唯一标识页面里面的元素，这样才能保证A页面中配置的元素A1可以被SDK找到，从而监听它产生的事件。

在html里面，元素是以DOM Tree组织的，如果沿着元素A1出发，一直向上记录它的parent和它在parent中的index，直到根节点body，那么就可以得到元素A1在DOM Tree中的唯一路径。

html的元素还会拥有很多属性，例如css class、id可以用来定位元素。通过Chrome开发者工具可以看到Mixpanel的可视化工具在配置元素的时候，使用的是[https://github.com/Autarc/optimal-select](https://github.com/Autarc/optimal-select)这个库来生成element的唯一标识的。而Github上还有[https://github.com/rowthan/whats-element](https://github.com/rowthan/whats-element)这样的库，也可以生成元素在DOM Tree中的唯一标识。

此外，还有平台在标识元素的时候，采用了`xpath`，这也是一个思路。

### 4. 如何查找元素

上面说到元素可以有唯一标识，那么有了唯一标识，就可以利用它的原理，找到这个元素。有一个很好用的API是`document.querySelector()`，这个API可以根据CSS选择器找到对应的元素。此外，根据元素的标识方法，还可以使用`document.getElementById()`、`document.getElementByName()`来实现元素的查找。

**这里需要重点强调的是，如果页面在配置完成之后又发生了修改，导致DOM Tree发生变化，此时需要被监测的元素的唯一标识可能也会发生改变。很可能导致根据之前的配置无法找到该元素了，或者找到的并不是我们希望监测的元素，从而导致产生的事件数量发生比较明显的变化。为了数据的稳定性和准确性，应该设有相应的监测告警处理这种case，并提示用户去重新配置页面。我个人认为这是无埋点最大的缺点。**

### 5. 标记元素时的高亮效果和可视化交互实现

这是一个比较细节的点，其实熟悉js的大牛们都知道，有无数种方式去实现鼠标移动到元素上时的`类hover`效果，点击元素后弹出一个对话框，让用户输入配置的信息也so easy。但是我想说的是，一旦我们采用向页面中动态添加元素的方式去实现可视化工具的交互界面，那么有可能会破坏掉页面原来的DOM Tree结构。从而导致生成元素唯一标识的时候出现误差，所以这里必须要好好处理，保证生成的元素标识不会受到影响。

我看到Mixpanel采用了`CustomElement`和`ShadowDOM`，把可视化工具所有的功能都用自定义的`Web Component`实现了，虽然目前只有Chrome支持`Web Component`，但是真的有点叼。。这样自定义的元素和交互不会对用户的网页DOM产生影响。当然，如果你的可视化工具实现做的很轻，比如只是将用户的网页放在一个`iframe`里面，大部分交互都交给iframe的parent页面去处理，那也可以在配置的时候，最小程度的破坏用户的网页了。

### 6. 配置工具中如何控制页面的跳转

当进入可视化配置状态时，我们可以让用户点击一个元素，然后弹一个对话框，让用户对这个元素进行配置。此时，如果这个元素本身的`click`行为是页面跳转呢？我们应该怎么处理？

这里本质上是一个交互设计的问题。在可视化配置工具中，应该有两种基本交互操作。一种是让用户选中某一个元素，进行配置；另一种，是让用户可以触发页面原有的行为。

为什么要有第二种交互？因为我们的工具肯定要支持用户进行二级页面的可视化配置对不对？或者说，用户的页面中可能会弹出一个对话框，对话框里面有一个按钮，用户对监测这个按钮，对它做配置，对不对？简单来说，就是用户页面中原有的点击行为，可能会导致页面结构产生变化，例如跳转，页面内弹出对话框等等。

那问题就好解了，除了点击，再设计一种交互来支持用户网页中原有的点击行为不就好了。用“右键点击”或者“按住shift+点击”之类都可以。反正不要再和网页默认的交互很容易产生冲突的方式就行。

最后再提一下，之前想很久没有想明白，如何能够能防止用户点击的时候页面产生跳转。后来才知道，DOM的事件流分三个阶段：捕获、目标、冒泡。所以为了避免用户的点击产身页面跳转，给document在捕获阶段加一个listener，拦截掉这个事件的继续分发就行了。

![DOM Event Mode](https://ws4.sinaimg.cn/large/006tKfTcgy1fsl9liu6oej30l40jen1i.jpg)

简单的示例代码如下：

```js
document.addEventListener('click', e => {
  // 如果是按住shift的点击，那么保持原有的行为
  if (e.shiftKey) {
    return;
  }
  // 如果是单纯的点击，那么拦截分发
  e.preventDefault();
  e.stopImmediatePropagation();
  // 获取元素的唯一标识，然后让用户进行配置等等
  this._selectElement(e.target);
}, true); // useCapture必须为true
```

## 四、总结

可以看到“无埋点”并不是**零侵入**，用户的网页中依然需要加载SDK的代码（除非你是浏览器厂商，可以在加载网页的时候，给网页加inject基础代码）。只是每一个行为事件的上报代码不需要开发人员手动编写，而是由运营人员用可视化工具配置，所以叫它**“可视化埋点”**也许更加合适。我们知道数据采集是数据分析的基础和先决条件，数据采集做不好，其他的东西都是空中楼阁。

这里可以小结一下“无埋点”技术的优劣。无埋点的好处是技术成本低，对用户非常友好，不需要重新部署，配置完成就可以生效。但是其缺点也非常明显，不具有代码埋点的灵活性和深度，只能采集到用户肉眼可见的数据，无法获取内存里的数据，同时也无法适应页面结构的变化，所以在实际生产中，要选择性地在合适的地方使用无埋点技术。

多扯一点产品设计和技术方案的选择，产品上是否可以支持采集内存数据呢？当然可以，比如微信小程序的[“自定义分析”](https://developers.weixin.qq.com/miniprogram/analysis/custom/#%E5%AE%9A%E4%B9%89%E4%BA%8B%E4%BB%B6)，就可以支持上报页面`data`下面的属性，这时虽然同样是可视化配置，运营人员肯定不会知道代码里面的变量名字，必须得有开发人员参与配置才行。关于页面结构发生变化之后的数据丢失，也是有方案可以破的。比如Mixpanel平台的Codeless Tracking，实际上采集了页面中所有页面的点击事件上报，然后在后台再去根据用户的配置计算转化数量。这样做的好处就是如果页面变化后，用户接到告警，修改了配置，那么用于数据上报方案是全量的，所以平台是由能力将过去的数据回溯出来的。而上面我们说的根据配置下发，查找监测指定元素，再上报数据的方案属于按需上报，数据出现误差是无法回溯的。不过全量上报数据大家也知道，太不友好了，这个数据量太大，不仅前端消耗资源多，如果为了做数据回溯，后台的存储压力也会加大，而存储的数据大部分还是无效的，这个成本有点高了。



## 五、参考资料

- [JS埋点技术分析](http://unclechen.github.io/2017/12/24/JS%E5%9F%8B%E7%82%B9%E6%8A%80%E6%9C%AF%E5%88%86%E6%9E%90/)
- [https://github.com/Autarc/optimal-select](https://github.com/Autarc/optimal-select)
- [https://github.com/rowthan/whats-element](https://github.com/rowthan/whats-element)
- [https://www.zhihu.com/question/38000812](https://www.zhihu.com/question/38000812)
