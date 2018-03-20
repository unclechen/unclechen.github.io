---
layout: post
title: JS埋点技术分析
date: '2017-12-24'
tags:
  - JS
  - SDK
  - 埋点
  - 无埋点
  - 可视化埋点
  - 监测
  - 数据
categories:
  - 技术
---

## 一、背景

[上一篇博客](http://unclechen.github.io/2017/12/18/Android%E5%9F%8B%E7%82%B9SDK%E6%8A%80%E6%9C%AF%E5%88%86%E6%9E%90/)分析了Android上的埋点SDK技术原理，这次我看看Web页面上的埋点。Web页面上的埋点主要通过JS完成，在JS里面同样有代码埋点、全埋点、可视化埋点三种方案，如果对这几种方案的概念不了解可以看下上一篇博客。由于[mixpanel-js](https://github.com/mixpanel/mixpanel-js)和[Sensors Analytics JavaScript SDK](https://github.com/sensorsdata/sa-sdk-javascript)都开源了自己的SDK，就以它们为例进行分析。

<!-- more -->

## 二、代码埋点

以Mixpanel为例（源码位于`/src/mixpanel-core.js`），看一下里面的实现。

### 2.1 基本用法

埋点之前，需要在head部分嵌入SDK，并调用SDK的初始化接口。以Mixpanel为例，官方介入文档提供的加载、初始化SDK代码如下：

```
<!-- start Mixpanel --><script type="text/javascript">(function(e,a){if(!a.__SV){var b=window;try{var c,l,i,j=b.location,g=j.hash;c=function(a,b){return(l=a.match(RegExp(b+"=([^&]*)")))?l[1]:null};g&&c(g,"state")&&(i=JSON.parse(decodeURIComponent(c(g,"state"))),"mpeditor"===i.action&&(b.sessionStorage.setItem("_mpcehash",g),history.replaceState(i.desiredHash||"",e.title,j.pathname+j.search)))}catch(m){}var k,h;window.mixpanel=a;a._i=[];a.init=function(b,c,f){function e(b,a){var c=a.split(".");2==c.length&&(b=b[c[0]],a=c[1]);b[a]=function(){b.push([a].concat(Array.prototype.slice.call(arguments,
0)))}}var d=a;"undefined"!==typeof f?d=a[f]=[]:f="mixpanel";d.people=d.people||[];d.toString=function(b){var a="mixpanel";"mixpanel"!==f&&(a+="."+f);b||(a+=" (stub)");return a};d.people.toString=function(){return d.toString(1)+".people (stub)"};k="disable time_event track track_pageview track_links track_forms register register_once alias unregister identify name_tag set_config reset people.set people.set_once people.unset people.increment people.append people.union people.track_charge people.clear_charges people.delete_user".split(" ");
for(h=0;h<k.length;h++)e(d,k[h]);a._i.push([b,c,f])};a.__SV=1.2;b=e.createElement("script");b.type="text/javascript";b.async=!0;b.src="undefined"!==typeof MIXPANEL_CUSTOM_LIB_URL?MIXPANEL_CUSTOM_LIB_URL:"file:"===e.location.protocol&&"//cdn.mxpnl.com/libs/mixpanel-2-latest.min.js".match(/^\/\//)?"https://cdn.mxpnl.com/libs/mixpanel-2-latest.min.js":"//cdn.mxpnl.com/libs/mixpanel-2-latest.min.js";c=e.getElementsByTagName("script")[0];c.parentNode.insertBefore(b,c)}})(document,window.mixpanel||[]);
mixpanel.init("YOUR TOKEN");</script><!-- end Mixpanel -->
```

这是一段立即执行的js代码，作用通常是去异步加载真正的JS SDK，然后调用SDK的初始化接口init方法，完成初始化的操作。

初始化的核心代码为

```js
mixpanel.init('YOUR TOKEN', { your: 'config' }, 'library_name')
```

也可以简写为

```js
mixpanel.init("YOUR TOKEN")
```


看下init方里面几个参数的含义：

- 第一个参数是你在后台注册的app token
- 第二个参数是SDK的配置，传入了一堆key-value，如果不传，SDK内部也有个默认配置，长下面这样：

```js
var DEFAULT_CONFIG = {
    'api_host': HTTP_PROTOCOL + 'api.mixpanel.com',
    'app_host': HTTP_PROTOCOL + 'mixpanel.com',
    'autotrack': true, // 是否打开全埋点监测
    'cdn': HTTP_PROTOCOL + 'cdn.mxpnl.com',
    'cross_subdomain_cookie': true,
    'persistence': 'cookie',
    'persistence_name': '',
    'cookie_name': '',
    'loaded': function() {},
    'store_google': true,
    'save_referrer': true,
    'test': false,
    'verbose': false,
    'img': false,
    'track_pageview': true,
    'debug': false,
    'track_links_timeout': 300,
    'cookie_expiration': 365,
    'upgrade': false,
    'disable_persistence': false,
    'disable_cookie': false,
    'secure_cookie': false,
    'ip': true,
    'property_blacklist': []
};
```

- 第三个参数是SDK全局变量名

> Mixpanel接入文档：[https://mixpanel.com/help/reference/javascript](https://mixpanel.com/help/reference/javascript)

### 2.2 上报的基本实现

代码埋点的方式通常都会被封装成类似`track(eventName, properties)`的接口，例如在Mixpanel中，可以用`mixpanel.track("Played song", {"genre": "hip-hop"});`来上报事件。

这里是整个SDK中最重要的地方，使用频率也是最高的。代码位于`/src/mixpanel-core.js`里面，先撇开复杂的逻辑和条件控制，看一下track的基本实现，我稍微加了点注释：

```js
//  track方法实现
MixpanelLib.prototype.track = function(event_name, properties, callback) {
    // 各种边界判断
    ...
    // 获取一些公共参数，和用户传入的properties一起encode
    var truncated_data = _.truncate(data, 255);
    var json_data = _.JSONEncode(truncated_data);
    var encoded_data = _.base64Encode(json_data);
    console.log('MIXPANEL REQUEST:');
    console.log(truncated_data);
    // 调用_send_request发送请求
    this._send_request(
        this.get_config('api_host') + '/track/',
        { 'data': encoded_data },
        this._prepare_callback(callback, truncated_data)
    );
    return truncated_data;
};

// 发送请求的实现，主要用的是XMLHttpRequest，如果浏览器不支持XMLHttpRequest，那么用动态添加img/script标签的方式
MixpanelLib.prototype._send_request = function(url, data, callback) {
    // 一些特殊情况的处理
    ...
    if ('img' in data) {
        var img = document.createElement('img');
        img.src = url;
        document.body.appendChild(img);
    } else if (USE_XHR) {
        try {
            var req = new XMLHttpRequest();
            req.open('GET', url, true);
            // send the mp_optout cookie
            // withCredentials cannot be modified until after calling .open on Android and Mobile Safari
            req.withCredentials = true;
            req.onreadystatechange = function () {
                if (req.readyState === 4) { // XMLHttpRequest.DONE == 4, except in safari 4
                    if (req.status === 200) {
                        if (callback) {
                            if (verbose_mode) {
                                callback(_.JSONDecode(req.responseText));
                            } else {
                                callback(Number(req.responseText));
                            }
                        }
                    } else {
                        var error = 'Bad HTTP status: ' + req.status + ' ' + req.statusText;
                        console.error(error);
                        if (callback) {
                            if (verbose_mode) {
                                callback({status: 0, error: error});
                            } else {
                                callback(0);
                            }
                        }
                    }
                }
            };
            req.send(null); // 发送异步请求
        } catch (e) {
            console.error(e);
        }
    } else {
        var script = document.createElement('script');
        script.type = 'text/javascript';
        script.async = true;
        script.defer = true;
        script.src = url;
        var s = document.getElementsByTagName('script')[0];
        s.parentNode.insertBefore(script, s);
    }
};
```

上面就是事件上报代码的核心实现。但是由于Web应用自身的一些特性，比如在追踪页面跳转行为（链接的点击、表单的提交等）时，为了防止数据发送不及时导致的数据丢失，SDK中提供一些诸如`track_links`和`track_forms`特殊方法，这些方法内部用的其实是setTimeout或者等待服务器返回结果之后再让页面跳转。

## 三、全埋点

Mixpanel和神策都提供了名为**“AutoTrack”**的方案，只需要在初始化SDK的时候，传入一个参数即可打开这个功能。JS SDK可以自动监测网页中所有的点击、表单submit等事件，这和AndroidSDK里面监听所有按钮的点击有些类似。

### 3.1 自动监测的元素、事件类型

- **神策JS：**设置AutoTrack之后，SDK就会自动追踪页面上的按钮(`a`、`button`、`input`) 这种html标签类型的点击情况，一旦页面某一个按钮发生了点击行为，SDK就会去采集此按钮的一些信息，例如: 这个按钮的标签类型，这个按钮的文本内容，这个按钮的`name`，这个按钮的`id`、`class`名，还有一些按钮特有的属性如`href`等。

- **MixpanelJS：**设置AutoTrack之后，SDK会监测页面上的所有`form表单`、`input标签`、`select和textarea标签`产生的`submit`、`change`、`click`事件，并采集这些标签上的属性一起上报。

### 3.2 全埋点监测的实现

以Mixpanel为例，在`/src/autotrack.js`代码中，把几个关键的方法扣出来看一下（不要问我为什么以Mixpanel为例，因为代码少一些。。。）。

当SDK初始化的时候，会执行autotrack里面的`_addDomEventHandlers`方法，给整个document的`submit`、`change`、`click`事件设置监听器。当监听到这几类事件时，会执行`_trackEvent`方法。

直接看代码，我给代码里面加了一点注释，来说明自动监测上报的过程。

```js
// SDK初始化时，通过register_event设置需要监听了submit、change、click这3类事件
// Mixpanel的js sdk代码里面自己封装了一个underscore模块，里面有一些工具方法
_addDomEventHandlers: function(instance) {
        var handler = _.bind(function(e) {
            e = e || window.event;
            this._trackEvent(e, instance);
        }, this);
        _.register_event(document, 'submit', handler, false, true);
        _.register_event(document, 'change', handler, false, true);
        _.register_event(document, 'click', handler, false, true);
    },

// register_event的实现，优先采用addEventListener的方式，如果浏览器不支持会尝试使用onXXX的方式
var register_event = function(element, type, handler, oldSchool, useCapture) {
        if (!element) {
            console.error('No valid element provided to register_event');
            return;
        }
        if (element.addEventListener && !oldSchool) {
            element.addEventListener(type, handler, !!useCapture);
        } else {
            var ontype = 'on' + type;
            var old_handler = element[ontype]; // can be undefined
            element[ontype] = makeHandler(element, handler, old_handler);
        }
    };

// 监听到事件发生后，调用_trackEvent方法来上报
_trackEvent: function(e, instance) {
        // 首先找到这个事件的target
        var target = this._getEventTarget(e);
        if (target.nodeType === TEXT_NODE) {
            target = target.parentNode;
        }
        // 然后判断是不是autotrack要监测的事件，如果不是的话，啥也不干直接返回。
        if (this._shouldTrackDomEvent(target, e)) {
            // 如果满足监测条件，那么从当前标签开始，向上追溯到body标签，并记录这条路径上所有的元素到一个数组中
            var targetElementList = [target];
            var curEl = target;
            while (curEl.parentNode && !this._isTag(curEl, 'body')) {
                targetElementList.push(curEl.parentNode);
                curEl = curEl.parentNode;
            }
            // 按照刚才记录的路径开始遍历（相当于自底向上）
            var elementsJson = [];
            var href, elementText, form, explicitNoTrack = false;
            _.each(targetElementList, function(el, idx) {
                // if the element or a parent element is an anchor tag
                // include the href as a property
                // 读取到a标签或者form标签时，记录它们的属性。
                if (el.tagName.toLowerCase() === 'a') {
                    href = el.getAttribute('href');
                } else if (el.tagName.toLowerCase() === 'form') {
                    form = el;
                }
                // crawl up to max of 5 nodes to populate text content
                // 读取节点的文本内容，最多往上读个5层
                if (!elementText && idx < 5 && el.textContent) {
                    var textContent = _.trim(el.textContent);
                    if (textContent) {
                        elementText = textContent.replace(/[\r\n]/g, ' ').replace(/[ ]+/g, ' ').substring(0, 255);
                    }
                }
                // allow users to programatically prevent tracking of elements by adding class 'mp-no-track'
                // 如果不希望某个节点被监测，开发者可以设置一个名为`mp-no-track`的css class
                var classes = this._getClassName(el).split(' ');
                if (_.includes(classes, 'mp-no-track')) {
                    explicitNoTrack = true;
                }
                // 读取每个标签的属性，最后这条路径上所有的标签都会被记录下来保存在elementsJson数组中
                elementsJson.push(this._getPropertiesFromElement(el));
            }, this);

            // 如果是一个开发者设置了不需要监测的标签，那么直接返回，不上报了
            if (explicitNoTrack) {
                return false;
            }

            // 处理采集到的属性，这里面有几个getXXXProperties(element/elements)方法（_getPropertiesFromElement、_getDefaultProperties、_getCustomProperties、_getFormFieldProperties），就是在读取各种属性
            var props = _.extend(
                this._getDefaultProperties(e.type), // 事件的基本属性，包含事件名称、window.location.host、window.location.pathname
                {
                    '$elements': elementsJson, // target标签到body标签这条路径上的所有标签及其属性
                    '$el_attr__href': href, // 采集到的href链接
                    '$el_text': elementText // target标签的文本内容
                },
                this._getCustomProperties(targetElementList) // 读取自定义属性，这里应该是指用户在后台管理界面配置的属性
            );
            if (form && (e.type === 'submit' || e.type === 'click')) {
                _.extend(props, this._getFormFieldProperties(form)); // 读取表单的一些属性
            }
            // 调用了代码埋点中介绍的track方法上报一个名为`$web_event`的事件，并带上采集的到的属性
            instance.track('$web_event', props);
            return true;
        }
    },

// _trackEvent之前，需要判断标签上的发生的事件是不是应该被autotrack监测上报
_shouldTrackDomEvent: function(element, event) {
        // html根节点下面的事件不需要监测
        if (!element || this._isTag(element, 'html') || element.nodeType !== ELEMENT_NODE) {
            return false;
        }
        var tag = element.tagName.toLowerCase();
        // 查看标签的名字
        // 如果是html则不监听
        // 如果是form标签下的submit事件，或者是input->button、input->submit标签的change、click事件，或者是select、textarea标签下的change、click事件，可以监听
        // 如果是其他标签，监听click事件
        switch (tag) {
            case 'html':
                return false;
            case 'form':
                return event.type === 'submit';
            case 'input':
                if (['button', 'submit'].indexOf(element.getAttribute('type')) === -1) {
                    return event.type === 'change';
                } else {
                    return event.type === 'click';
                }
            case 'select':
            case 'textarea':
                return event.type === 'change';
            default:
                return event.type === 'click';
        }
    },
```

### 3.3 全埋点小结

可以看到全埋点还是有点暴力的，会采集的数据量也挺大，并且采集到的属性也比较多，可以看到在MixpanelSDK中，如果页面结构比较深，那么数据报过去分析起来可能还是需要花点时间的，同时也会产生大量可能不会使用的数据，对资源也是一种浪费。在神策SDK的接入文档中也提到，建议那些按钮不是很多的，相对简单的页面可以采用这个方法。一般情况下，如果网页上的按钮比较多的话，因为每次按钮的点击都会发数据，数据量很大。

## 四、可视化埋点

Mixpanel和神策等平台，都提供了JS可视化埋点功能，与全埋点相比，这种方式可以指定自己想要监测的元素和属性（所有可以点击的元素），既可以做到动态配置，又不会像全埋点那样产生大量的数据（但也有例外，比如Mixpanel的可视化埋点仍然上报了全量点击数据，只是在后台根据可视化配置过滤出实际的数据）。

可视化埋点首先需要进入埋点模式，以Mixpanel为例，可视化埋点的入口在后台管理界面，需要在后台输入需要埋点的页面url，然后再进入我的Web页面，此时就会加载可视化标记的编辑器（代码见`autotrack.js`中的`_maybeLoadEditor`方法，需要注意的是这个页面必须已经嵌入了JS SDK）。

> 这里一定要从平台登录才可以进入可视化编辑状态，这实际上是对安全性的一个保证，试想如果只要嵌入SDK就可以做可视化埋点，那岂不是我的Web应用随便就可以被别人埋点，对我的数据产生干扰了。在Mixpanel JS SDK内部，通常会**判断当前页面的sessionStorge/localStorage中是否有一个开启可视化编辑器标志字段（例如Mixpanel是`_mpcehash`字段）**，读取这个字段，解析到其中的打开可视化编辑器的开关开启之后，就会加载可视化编辑器。由此可见其实从SDK后台管理界面跳转到可视化标记页面时，就是向SessionStorage中写入了相应的标志。

**可视化埋点的两个关键点是：**

- **标记元素，保存配置**：这一步要保存好需要追踪的元素的element_path，以及需要追踪的元素。
- **下发配置，查找元素，监听点击，上报行为**：这一步要通过element_path找到元素，给它添加一个点击监听器，当点击事件发生时SDK上报事件。

这里最重要的就是：**元素的标记和查找**，不同的SDK就是实现标记和查找的时候稍微有一些差异。

### 4.1 标记元素，保存配置

> MixpanelJS加载可视化编辑器时，需要从`//mixpanel.com/js-bundle/reports/collect-everything/editor.js?_ts={$timestamp}`去加载一个js文件，**这个js差不多可以看成一个独立的标记SDK，**最后这个请求会被重定向到一个cdn地址（`https://cdn4.mxpnl.com/static/asset-cache/3fc4abfdcebcb5121f1ebf143415b232/compiled/reports/collect-everything/editor.min.js`），随便打开这个js看下就有两万多行，因此单独做成了一个按需加载的模块。

由于Mixpanel就没有提供标记SDK的源码，不过从体验和**抓包分析后台下发的配置**，我仍然可以推测出技术实现的细节。

**从体验的角度来讲，**当进入可视化编辑状态时，在开发者web页面上，用户的鼠标经过**可以被点击**的元素（例如a、button标签等）时，这个元素会被一种颜色高亮提示，此时点击一下这个元素，就会弹出一个浮窗，用户填写信息，设置一个事件和一些属性，保存之后就算完成对这个元素的标记操作了，当标记过的元素的配置保存好了以后，这个元素会用另外一种特殊的颜色高亮标识起来。

**从技术的角度来讲，**我看下神策JS SDK中的`vtrack.sdk.js`这个文件，当神策SDK进入可视化标记模式的时候，会去加载`vendor.js`和`vendor.css`，这两个文件可以看作一个标记SDK。那么`vendor.js`代码里是**如何标记需要追踪的元素的**呢？

在`vendor.js`中，有一个`EventDefine`模块，这个模块负责把一个标签处理成我要保存的selector。

**EventDefine**有三个方法：

- getSelfAttr：获取一个标签内的文本内容，举例来说，一个`<p>This is another paragraph.</p>`得到的内容是`This is another paragraph.`。
- toSelector：把一个标签的tagName、id、classNames解析出来，拼成一个串。举例来说，一个`<div id="test" class="uncle chen"></div>`标签，它的selector是`div#id.uncle.chen`，这个selector是可以直接给jQuery用来查找元素的。
- toAllSelector：选择一个需要追踪的标签，并给这个标签定义点击时上报的事件（EventDefine），最后将这个事件转成一个selector保存下来，selector就是用于给jQuery来查找元素的选择器，这里需要注意，如果一个元素是在iFrame里面的，那么SDK保存的选择器路径是相对iFrame内部的，而不是最外层的document。

前两个方法都是给`toAllSelector`方法调用的，`toAllSelector`方法是神策的标记SDK的重点，这个方法的实现如下：

```js
  toAllSelector: function($target, outDocuemnt) {
      outDocuemnt = outDocuemnt ? $(outDocuemnt) : $(document);
      var $parent, newSelSize, newSelector, parts, selSize, selector, targetSel;
      selector = this.toSelector($target, outDocuemnt);
      $parent = $target.parent();
      selSize = outDocuemnt.find(selector).length;
      while ($parent.prop('tagName') !== 'BODY' && selSize !== 1) {
        newSelector = '' + (this.toSelector($parent)) + ' ' + selector; // 如果向上回溯的话，selector会用空格分开保存
        newSelSize = outDocuemnt.find(newSelector).length;
        if (newSelSize < selSize) {
          selector = newSelector;
          selSize = newSelSize;
        }
        $parent = $parent.parent();
      }
      var nthEle = selector;
      var selfAttr = this.getSelfAttr($target);
      return {
        nthEle: nthEle,
        selfAttr: selfAttr
      };
    }
```

当选中一个标签时，SDK会提取出这个标签的selector，然后用jQuery选择器查找这个selector指向的元素，如果这个selector指向的元素有多个（`selSize !== 1`，也就是说这个元素有着多个兄弟标签），那么还需要进一步去提取其父标签的selector，直到找出可以**唯一**标识这个元素的selector为止，最后将需要追踪的这个元素以{nthEle: nthEle, selfAttr: selfAttr}`，nthEle是selector，selfAttr是文本内容。

**简单总结一下元素的标记，在Web页面中，一个元素的唯一css选择器生成算法，应该记录了从body到这个元素的完整路径，并记录每一个节点是其父亲节点的第几个孩子节点，即这个元素在整个Dom Tree中的深度和下标。此外，为了在一定程度上抵抗Dom Tree的变化，下标应该记录的是这个元素在父节点中相同类型元素的index（nth-of-type），而不是其父节点下面所有孩子节点的index（nth-child）。**

### 4.2 查找元素，监听上报

标记元素，保存配置之后，SDK如何根据配置来监测配置好的元素，并进行上报呢？前面我说到Mixpanel在可视化埋点的上报实现里，仍然保持了全量点击事件上报，并在每个上报中把元素在Dom Tree的完整路径一起上报到了后台，由后台去过滤出可视化事件。

所以这里我看下神策js的代码，在可视化模块`vtrack.sdk.js`中，正常模式下，会去解析后台下发的配置，找到标记过的元素，绑定事件。

**1.下发配置**

```js
// 进入普通模式时，会从后台的一个接口去拉去标记过的元素（这里也叫“部署”过的元素）的关键信息，然后进行解析
  enterNormalMode: function() {
    sd.vtrack_mode = 'normalMode';
    var me = this;
    this.getDeployFile().then(function() {
      me.parseDeployFile(); // 解析配置
    });
  },
```

由于神策的后台代码是走私有化部署的，我没有办法体验，这里看一份诸葛IO平台可视化配置：

```json
{
  "code": 10001,
  "msg": "Request success",
  "visual_events": [
    {
      "event_name": "button3333",
      "url": "http://localhost:63343/sa-sdk-javascript/zhuge.html?_ijt=kc0vtnal8qahrd2tq5obomukh8", 
      "element": [
        "#b"
      ],
      "attr": [
        {
          "name": "ppp",
          "selector": "body>p:eq(0)"
        }
      ],
      "app_id": 56070,
      "platform": 3,
      "create_date_time": "2018-01-23 17:45:53",
      "hidden": null,
      "stop": null,
      "alias_name": null,
      "edit_url": "http://localhost:63343/sa-sdk-javascript/zhuge.html?_ijt=kc0vtnal8qahrd2tq5obomukh8"
    },
    {
      "event_name": "button点击2222",
      "url": "",
      "element": [
        "#b"
      ],
      "attr": [],
      "app_id": 56070,
      "platform": 3,
      "create_date_time": "2018-01-23 17:39:16",
      "hidden": null,
      "stop": null,
      "alias_name": null,
      "edit_url": "http://localhost:63343/sa-sdk-javascript/zhuge.html?_ijt=k9d5hjmkqt1ethu1ao51cf4t9e"
    },
    {
      "event_name": "20180123",
      "url": "",
      "element": [
        "#b"
      ],
      "attr": [],
      "app_id": 56070,
      "platform": 3,
      "create_date_time": "2018-01-23 17:08:04",
      "hidden": null,
      "stop": null,
      "alias_name": null,
      "edit_url": "http://localhost:63343/sa-sdk-javascript/zhuge.html?_ijt=ir7e9n0scm88b0k2uvotnc4ntd"
    },
    {
      "event_name": "hello链接点击",
      "url": "http://localhost:63342/sa-sdk-javascript/zhuge.html?_ijt=hg3r25mmicg2icf0jtndtaskh7#",
      "element": [
        "body>a:eq(0)"
      ],
      "attr": [
        {
          "name": "自定义属性",
          "selector": "body>p:eq(0)"
        }
      ],
      "app_id": 56070,
      "platform": 3,
      "create_date_time": "2017-12-22 16:25:29",
      "hidden": false,
      "stop": false,
      "alias_name": null,
      "edit_url": "http://localhost:63342/sa-sdk-javascript/zhuge.html?_ijt=hg3r25mmicg2icf0jtndtaskh7#"
    }
  ]
}
```

注意，上面这份配置有两个字段，一个是url，另一个是editUrl，editUrl表示标记元素的时候，是在哪一个页面里操作的。url表示应该去哪个url下面查找标记的元素。因为有些情况下，虽然我们是在某一个页面标记的元素，但是我们有很多其他页面和这个页面长得类似，比如商品详情类的页面，所以我们其实希望在所有的商品详情页都可以上报某些事件。所以，如果url为一个具体的值，例如"[http://localhost:63342/sa-sdk-javascript/zhuge.html?_ijt=hg3r25mmicg2icf0jtndtaskh7#](http://localhost:63342/sa-sdk-javascript/zhuge.html?_ijt=hg3r25mmicg2icf0jtndtaskh7#)"，说明只应该在这个url对应的页面中查找元素，上报行为即可；如果url=""，说明我们应该在整个Web应用中的所有页面都去依据路径查找元素，上报事件。

**2.解析配置，监测元素**

还是直接看神策代码实现，里面加了点注释：

```js
  // 解析配置，查看当前页面中是否有元素需要被追踪，把需要追踪的元素的配置保存到requiredData变量中
  parseDeployFile: function() {
    this.requireData = this.checkUrl(this.deployData);
    this.listenEvents();
  },
  // 找到元素，绑定点击事件的处理，当元素被点击时，上报事件
  listenEvents: function() {
    var data = this.requireData;
    var me = this;
    for (var i = 0; i < data.length; i++) {
      this.getEle(data[i]).on('click', function(ev) {
        return function() {
          me.doVTrackAction(ev);
        }
      }(data[i]));
    }
  },
  doVTrackAction: function(data) {
    sd.track(
      data.eventName,
      {
        $from_vtrack: String(data.trigger_id)
      },
      {
        $lib_method: 'vtrack',
        $lib_detail: String(data.trigger_id)
      }
    );
  },
  // 通过jQuery的选择器来找到元素，我在前一节的标记操作中知道，标记SDK会把一个定义好的事件eventDefine转化成一个{nthEle: nthEle, selfAttr: selfAttr}结构保存起来，这里去寻找元素的时候和标记那里的逻辑其实是一个逆操作。
  // 这里要注意，和标记时一样元素，碰到iframe时要特殊处理一下。
  getEle: function(data) {
    var ele;
    if ($(data.nthEle[0]) && $(data.nthEle[0]).prop('tagName') === 'IFRAME') {
      try {
        ele = $(data.nthEle[0]).contents().find(data.nthEle.slice(1).join(' '));
      } catch (e) {
      }
      ;
    } else {
      ele = $(data.nthEle.join(' '));
    }
    if (data.selfAttr && data.selfAttr.text !== void 0) {
      ele = ele.filter(':contains(' + data.selfAttr.text + ')');
    }
    return ele;
  },
```

神策查找元素的时候用到了jQuery，而在Mixpanel中没有用jQuery，而是用的`Document.querySelectorAll`这个API。毕竟有很多移动页面为了优化加载速度，不会用jQuery这么重的库。

此外，当追踪一些特殊的标签时，可以考虑用[XPath](http://www.w3school.com.cn/xpath/)去定位，今日头条的广告监测插件其实就用到了XPath。

**3.给事件上报添加属性**

单独把添加属性拿出来讲，是因为它的原理是类似的。前面我们只提到标记一个元素，当它被点击的时候上报事件，但是这样没有在上报事件的同时带上自定义的一些属性。

其实只要是Web页面上出现了的元素，我们都可以把它记录下来，然后在事件发生的时候，查找到这些元素，并把它们的内容作为事件的属性上报上来。我们可以在标记了一个元素的时候，再去标记其他的一些元素（例如一些文本标签），并设置其他这些元素各自所对应的key，当事件发生时，我们可以找到其他这些元素，并获取到其中显示的文本内容，作为各自参数的值，上报到后台。


### 4.3 可视化埋点小结

可以看出，在JS上实现可视化埋点不是一件太麻烦的事情，我认为最关键的两件事就是标记元素和查找元素。不过它缺点是只会读取页面上的标签元素的展示出来的属性，也不会像代码埋点的方案那样去理解业务场景，获取上下文（通常在内存里）的一些属性；另外，当页面的结构发生变化的时候，可能要重新进行一次标记操作。有些平台是通过对事件监测的告警来提醒用户的，当事件数量同比大幅减少的时候，大概率是因为某次改版导致页面Dom Tree产生了变化，通过配置下发里面的元素路径找不到之前标记的元素了，这时就应该提醒用户重新标记。


## 五、总结

本文从代码埋点、全埋点、可视化埋点三个角度，以Mixpanel、神策数据的JS SDK的源代码，分析了Web页面埋点的实现方案的实现。在流量红利逐渐消失的现在，数据的采集、分析和精细化的运营显得更加重要，下面简单列一个表格对以上三种方式的埋点方案进行对比，还是那句话，三种埋点方式相辅相成，结合业务需求搭配使用，适合自己的才是最好的。

|埋点方案|优点|缺点|适用场景|
|:---:|:---|:---|:---|
|代码埋点|可以按照业务上报详细、定制化的数据|需要开发人员参与，更新维护成本高，无法获得历史数据|对上下文理解要求较高的业务数据|
|全埋点|对发人员依赖低，仅需嵌入一次SDK，可以全量上报通用数据，可以拿到历史数据|数量量太大，占用更多资源，且无法收集业务上下文数据，给后续数据筛选和分析带来一定的难度|上下文相对独立的、通用的数据|
|可视化埋点|对开发人员依赖低，可以按照业务需求上报数据，对上下文数据有一定收集能力|标记事件有一定的操作难度，事件需要被更新时无法获得历史数据，界面变化时标记的元素可能失效|业务上下文数据相对简单，操作交互比较固定的界面|
