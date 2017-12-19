---
layout: post
title: Android埋点SDK技术分析
date: '2017-12-18'
tags:
  - Android
  - SDK
  - 埋点
  - 无埋点
  - 可视化埋点
categories:
  - 技术
---

# 一、概念

埋点，是对Web网站、App进行数据采集的一种方法。通过埋点，可以收集用户在应用中的产生行为，进而用于分析和优化产品后续的体验，也可以为产品的运营提供数据支撑，其中常见的指标有PV、UV、页面时长和按钮的点击等。

采集行为数据时，通常需要在Web页面/App里面添加一些代码，当用户的行为达到某种条件时，就会向服务器上报用户的行为。其实添加这些代码的过程就可以叫做“埋点”，在很久以前就已经出现了这种技术。随着技术的发展和大家对数据采集要求的不断提高，我认为埋点的技术方案走过了下面几个阶段：

- **代码埋点：**代码埋点是指开发人员按照产品/运营的需求，在Web页面/App的源码里面添加行为上报的代码，当用户的行为满足某一个条件时，这些代码就会被执行，向服务器上报行为数据。这种方案是最基础的方案，每次增加或者修改数据上报的条件，都需要开发人员的参与，并且只能在下一个版本上线后才能看到效果。很多公司都提供了这类数据上报的SDK，将行为上报的后台服务器接口封装成了简单的客户端SDK接口。开发者可以通过嵌入这类SDK，在埋点的地方调用少量的代码就可以上报行为数据。

- **全埋点：**全埋点指的是将Web页面/App内产生的所有的、满足某个条件的行为，全部上报到后台服务器。例如把App中所有的按钮点击都进行上报，然后由产品/运营去后台筛选所需要的行为数据。这种方案的优点非常明显，就是可以不用在新增/修改行为上报条件时，再找开发人员去修改埋点的代码。然而它的缺点也和优点一样明显，那就是上报的数据量比代码埋点大很多，里面可能很多是没有价值的数据。此外，这种方案更倾向于独立去看待用户的行为，而没有关注行为的上下文，给数据分析带来了一些难度。很多公司也提供了这类功能的SDK，通过静态或者动态的方式，“Hook”了原有的App代码，从而实现了行为的监测，在数据上报时通常是采用累积多条再上报的方案来合并请求。

- **可视化埋点：**可视化埋点是指产品/运营在Web页面/App的界面上进行圈选，配置需要监测界面上哪一个元素，然后保存这个配置，当App启动时会从后台服务器获得产品/运营预先圈选好的配置，然后根据这份配置监测App界面上的元素，当某一个元素满足条件时，就会上报行为数据到后台服务器。有了全埋点技术方案，从体验优化的角度很容易想到按需埋点，可视化埋点就是一种按需配置埋点的方案。现在也有一些公司提供了这类SDK，圈选监测元素时，一般都是提供一个Web管理界面，手机在安装并初始化了SDK之后，可以和管理界面了连接，让用户在Web管理界面上配置需要监测的元素。


业界有多家SDK都支持上面介绍的3种埋点方案中的一种或者全部，例如Mixpanel、Sensorsdata、TalkingData、GrowingIO、诸葛IO、Heap Analytics、MTA、Umeng Analytics、百度，只是大家对后两种埋点的称呼不完全相同，有的叫无埋点或者codeless埋点。由于[Mixpanel](https://github.com/mixpanel/mixpanel-android)（支持代码埋点、可视化埋点）和[Sensorsdata](https://github.com/sensorsdata/sa-sdk-android)（全部支持）都开源了自己的全部SDK，技术方案也比较类似，下面以它们的Android SDK为例，简单分析一下3种埋点方案的技术实现。

<!-- more -->

# 二、代码埋点

包含Mixpanel SDK在内的大部分SDK，都会把这种埋点方案封装成一个比较简单的接口，在这里是`track(String eventName, JSONObject properties)`，开发者在调用这个接口时，可以把一个事件名称和事件的属性传入，然后就可以上报到后台了。

在实现上，Mixpanel SDK默认采用一条HandlerThread线程来处理事件，当开发者调用`track(String eventName, JSONObject properties)`方法时，**主线程切换到HandlerThread**当中，并先将事件存入数据库。然后看SDK中是否累计到了40个事件，如果累计到40个事件的话，就合并它们上报到后台。

当开发者设置为debug模式，或者手动调用`flush`接口时，可以立即上报累计的所有事件，不过由于只有一条线程，所以如果在flush的时候，前面的事件还没有处理完成，SDK会间隔1分钟再次去处理后面的这些事件。

开发者可以设置累计上报的事件数量阈值、事件阻塞时再次尝试上报的时间间隔等。这种方案比较基础，相信大部分开发者都接触过，不需要过多分析。

# 三、全埋点

## 3.1 AOP基础

Mixpanel现在的Android SDK没有提供这个功能，但是神策Android SDK提供了，实现方式是依赖AOP。那么什么是AOP？

> 在软件业，AOP为Aspect Oriented Programming的缩写，意为：面向切面编程，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。AOP是OOP的延续，是软件开发中的一个热点，也是Spring框架中的一个重要内容，是函数式编程的一种衍生范型。利用AOP可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合度降低，提高程序的可重用性，同时提高了开发的效率。（from baidu baike）

简而言之，AOP是可以通过预编译方式和运行期动态代理实现在不修改源代码的情况下给程序动态统一添加功能的一种技术。

**Sensors Analytics AndroidSDK全埋点的实现就是通过在代码编译阶段，找到源代码中需要上报事件的位置，插入SDK的事件上报代码。它用到的框架是[AspectJ](https://www.eclipse.org/aspectj/)。**

说到这里，必须简单了解一下AspectJ以及它里面的一些概念.它是AOP的领跑者，在很多地方我们可以看到它的身影，例如JakeWartson大神贡献的一个注解日志和性能调优框架[Hugo](https://github.com/JakeWharton/hugo)，在Spring框架里面也大量应用到AspectJ。我理解AspectJ里面的主要几个概念有：

- **JPoint：**代码切点（就是我们要插入代码的地方）
- **Aspect：**代码切点的描述
	- **Pointcut：**描述切点具体是什么样的点，如函数被调用的地方（`Call(MethodSignature)`）、函数执行的内部（`execution(MethodSignature)`）
	- **Advice：**描述在切点的什么位置插入代码，如在Pointcut前面（`@Before`）还是后面（`@After`），还是环绕整个Pointcut（`@Around`）

由此可见，在实现AOP功能时，需要做下面几件事：

- 定义一个Aspect，这个Aspect里面必须有Pointcut和Advice两个属性
- 编写在匹配到符合Pointcut和Advice描述的代码时，需要注入的代码
- 在代码编译时，通过特殊的java编译器（Aspect的ajc编译器），找到符合我们定义的Aspect的代码，将需要注入的代码插入到Advice指定的位置。

如果你对AspectJ有了解的话，已经可以猜到SDK内部是怎么实现全埋点的了；如果没有接触，我觉得也不用急于全面地去学习AspectJ，因为SDK内部只用到了AspectJ当中的一小部分功能而已，可以直接看下面的分析。

## 3.2 全埋点-技术实现

神策SDK里面是如何监测View点击事件呢？我把SDK代码简化一下进行分析，有下面几个步骤：

### 3.2.1 定义一个Aspect

```java
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.After;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;

@Aspect
public class ViewOnClickListenerAspectj {

    /**
     * android.view.View.OnClickListener.onClick(android.view.View)
     *
     * @param joinPoint JoinPoint
     * @throws Throwable Exception
     */
    @After("execution(* android.view.View.OnClickListener.onClick(android.view.View))")
    public void onViewClickAOP(final JoinPoint joinPoint) throws Throwable {
        AopUtil.sendTrackEventToSDK(joinPoint, "onViewOnClick");
    }
}

```

这段Aspect的代码定义了：**在执行android.view.View.OnClickListener.onClick(android.view.View)方法原有的实现后面，需要插入`AopUtil.sendTrackEventToSDK(joinPoint, "onViewOnClick");`这段代码。**

`AopUtil.sendTrackEventToSDK(joinPoint, "onViewOnClick");`这段代码做的事情就是点击事件的上报。因为神策SDK将全埋点功能和主SDK包分离成了两个jar包，所以通过AopUtil工具去调用真正的事件上报代码，这里不细述其实现，下面直接看这段代码背后真正的点击上报实现。

```java
SensorsDataAPI.sharedInstance().track(AopConstants.APP_CLICK_EVENT_NAME, properties);

```

可以看到AOP实现的点击监测，最后也走`track`方法进行上报了。

### 3.2.2 使用ajc编译器向源代码中插入Aspect代码

采用AspectJ框架编写的代码，想要注入原来的工程的代码，需要在`app/build.gradle`中引用ajc编译器，脚本如下：

```groovy
...
import org.aspectj.bridge.IMessage
import org.aspectj.bridge.MessageHandler
import org.aspectj.tools.ajc.Main

buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath 'org.aspectj:aspectjtools:1.8.10'
    }
}

repositories {
    mavenCentral()
}

android {
    ...
}

dependencies {
    ...
    compile 'org.aspectj:aspectjrt:1.8.10'
}

final def log = project.logger
final def variants = project.android.applicationVariants

variants.all { variant ->
    if (!variant.buildType.isDebuggable()) {
        log.debug("Skipping non-debuggable build type '${variant.buildType.name}'.")
        return;
    }

    JavaCompile javaCompile = variant.javaCompile
    javaCompile.doLast {
        String[] args = ["-showWeaveInfo",
                     "-1.5",
                     "-inpath", javaCompile.destinationDir.toString(),
                     "-aspectpath", javaCompile.classpath.asPath,
                     "-d", javaCompile.destinationDir.toString(),
                     "-classpath", javaCompile.classpath.asPath,
                     "-bootclasspath", project.android.bootClasspath.join(File.pathSeparator)]
        log.debug "ajc args: " + Arrays.toString(args)

        MessageHandler handler = new MessageHandler(true);
        new Main().run(args, handler);
        for (IMessage message : handler.getMessages(null, true)) {
           switch (message.getKind()) {
                case IMessage.ABORT:
                case IMessage.ERROR:
                case IMessage.FAIL:
                    log.error message.message, message.thrown
                    break;
                case IMessage.WARNING:
                    log.warn message.message, message.thrown
                    break;
                case IMessage.INFO:
                    log.info message.message, message.thrown
                    break;
                case IMessage.DEBUG:
                    log.debug message.message, message.thrown
                    break;
            }
        }
    }
}
```

在SensorsAndroidSDK中，把上面这段脚本编写成了一个[gradle插件](https://github.com/sensorsdata/sa-sdk-android-plugin2)，开发者只需要在`app/build.gradle`引用这个插件即可。

```groovy
apply plugin: 'com.sensorsdata.analytics.android'
```

### 3.2.3 完成代码插入，查看插入之后的效果

完成上面两步，就可以实现在`android.view.View.OnClickListener.onClick(android.view.View)`方法中插入我们的数据上报代码了。我们在demo代码中加一个Button，并给它set一个OnClickListener，编译一下代码，查看`build/intermediates/classes/debug/`里面class文件，经过ajc编译之后，原始代码中插入了Aspect的代码，并调用了`ViewOnClickListenerAspectj`里面的`onViewClickAOP`方法。

```java
public class MainActivity extends Activity {
    public MainActivity() {
    }

    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        this.setContentView(2130968603);
        Button btnTst = (Button)this.findViewById(2131427422);
        btnTst.setOnClickListener(new OnClickListener() {
            public void onClick(View v) {
                JoinPoint var2 = Factory.makeJP(ajc$tjp_0, this, this, v);

                try {
                    Log.i("MainActivity", "button clicked");
                } catch (Throwable var5) {
                    ViewOnClickListenerAspectj.aspectOf().onViewClickAOP(var2);
                    throw var5;
                }

                ViewOnClickListenerAspectj.aspectOf().onViewClickAOP(var2);
            }

            static {
                ajc$preClinit();
            }
        });
    }
}
```

AspectJ的基本用法就是这样，SensorsAndroidSDK借助AspectJ插入了Aspect代码，这是一种静态的方式。静态的全埋点方案，本质上是对字节码进行修改，插入事件上报的代码。

修改字节码，除了这种方案之外，还有Android Gradle插件提供的trasform api（1.5.0版本以上）、ASM、Javassist。在网易乐得的埋点方案，Nuwa热修复项目都可以见到这些技术的实践。

## 3.3 AspectJ相关资料

- Aspect Oriented Programming in Android：[https://fernandocejas.com/2014/08/03/aspect-oriented-programming-in-android/](https://fernandocejas.com/2014/08/03/aspect-oriented-programming-in-android/)
- AOP之AspectJ全面剖析in Android：[http://www.jianshu.com/p/f90e04bcb326](http://www.jianshu.com/p/f90e04bcb326)
- 沪江开源了一个叫做AspectJX的插件，扩展了AspectJ，除了对src代码进行AOP，还支持kotlin、工程中引用的jar和aar进行AOP：[https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx](https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx)
-  关于 Spring AOP (AspectJ) 你该知晓的一切：[http://blog.csdn.net/javazejian/article/details/56267036](http://blog.csdn.net/javazejian/article/details/56267036)


# 四、可视化埋点

## 4.1 可视化埋点-技术实现

可视化埋点，需要经过两个步骤，可以由非技术人员操作完成。第一步，使用嵌入了Mixpanel/SensorsSDK的App连接后台，当手机App与后台同步时，后台管理界面上会显示和手机App一样的界面，用户可以在管理界面上用鼠标选择需要监测的元素，设置事件名称，需要监测的元素属性等（据说有些SDK的圈选操作是在手机上进行的，不管是什么方式本质上是一样的，需要保存一份配置到后台）。第二步，嵌入了SDK的App启动时，会从服务器获取到一份配置，再根据这份配置去检测App中的界面及其元素，满足配置的条件时向服务器上报事件。下面以Mixpanel、SensorsdataSDK为例，简单分析一下技术方案的实现。

### 4.1.1 圈选需要监测的元素，保存配置

**1.创建WebSocket连接后台**

采用WebSocket连接是因为要让手机和后台长时间保持连接，是一个持续的双向通信。连接到后台时，把手机的设备信息发送到后台。

**2.把App界面截图发送到后台**

创建Socket连接后，在主线程中，对App中启动的Activity进行扫描，找到界面的RootView（其实是DecorView）。在查找RootView的同时，会对RootView进行截图，截图时采用反射调用View类`createSnapshot`方法。

截图之后，SDK内部会判断图片的hash值，如果图片发生了变化，会采用递归的方法深度遍历Activity的ViewTree，遍历同时读取View的属性（id、top、left、width、height、class名称、layoutRules等等）。

最后，将上面收集到数据发送到连接的后台，由后台解析之后，把App界面展示在Web页面。用户可以在这个Web页面圈选需要监测的元素，设置这个元素的时间名称（event_type和event_name），并保存这个配置。


### 4.1.2 获取配置，监测元素的行为，上报事件

**1.获取配置**

SDK启动时，会从服务器拉取一份JSON格式的配置，保存到sharedPreference里，同时SDK会扫描`android.R`文件里面的资源id和资源的name并保存起来。

SDK得到配置之后，解析成JSON对象，读取`event_bindings`字段，再进一步读取`events`字段，这个字段下面包含了一个数组，数组的每个元素都描述了一类事件，并包含了这类事件需要监测的元素所在的Activity和元素的路径。这份配置基本上是这样的一个结构：

```json
event_bindings: {
    events:[
        {
            target_activity: ""
            event_name: ""
            event_type: ""
            path: [
                {
                    prefix:
                    view_class:
                    index:
                    id:
                    id_name:
                }, 
                ...
            ]
        }
    ]
}
```

收到了这份配置之后，SDK会把根据每个event信息，生成一个`ViewVisitor`。`ViewVisitor`的作用就是把`path`数组里面指向的所有View元素都找到，并且根据event_type，给这个View元素设置相应的行为监测器，当这个View发生指定行为时，监测器就会监测到，并上报行为。

生成ViewVisitor之后，SDK内部是以`Map<activity, ViewVisitor>`结构保存它们的，这也比较容易理解。

**2.监测元素，上报事件**

`ViewVisitor`是怎么监测元素的产生的行为呢？答案就是`View.AccessibilityDelegate`。

在Android SDK里面，AccessibilityService)为我们提供了一系列的事件回调，帮助我们指示一些用户界面的状态变化。我们可以派生辅助功能类，进而对不同的AccessibilityEvent进行处理，我们看下AccessibilityEvent里面有哪些事件类型：

```java
    /**
     * Represents the event of clicking on a {@link android.view.View} like
     * {@link android.widget.Button}, {@link android.widget.CompoundButton}, etc.
     */
    public static final int TYPE_VIEW_CLICKED = 0x00000001;

    /**
     * Represents the event of long clicking on a {@link android.view.View} like
     * {@link android.widget.Button}, {@link android.widget.CompoundButton}, etc.
     */
    public static final int TYPE_VIEW_LONG_CLICKED = 0x00000002;

    /**
     * Represents the event of selecting an item usually in the context of an
     * {@link android.widget.AdapterView}.
     */
    public static final int TYPE_VIEW_SELECTED = 0x00000004;

    /**
     * Represents the event of setting input focus of a {@link android.view.View}.
     */
    public static final int TYPE_VIEW_FOCUSED = 0x00000008;

    /**
     * Represents the event of changing the text of an {@link android.widget.EditText}.
     */
    public static final int TYPE_VIEW_TEXT_CHANGED = 0x00000010;
    ...
```

**以点击事件`TYPE_VIEW_CLICKED `为例**，当Activity界面的RootView开始绘制的时候（ViewTreeObserver.OnGlobalLayoutListener的onGlobalLayout回调时），ViewVisitor也会开始寻找指定的View，并给这个View设置新的AccessibilityDelegate。简单看一下这个新的View.AccessibilityDelegate是怎么写的：

```java
private class TrackingAccessibilityDelegate extends View.AccessibilityDelegate {
...
            public TrackingAccessibilityDelegate(View.AccessibilityDelegate realDelegate) {
                mRealDelegate = realDelegate;
            }

            public View.AccessibilityDelegate getRealDelegate() {
                return mRealDelegate;
            }

            ...
            
            @Override
            public void sendAccessibilityEvent(View host, int eventType) {
                if (eventType == mEventType) {
                    fireEvent(host); // 事件上报
                }

                if (null != mRealDelegate) {
                    mRealDelegate.sendAccessibilityEvent(host, eventType);
                }
            }

            private View.AccessibilityDelegate mRealDelegate;
        }
        ...
```

可以看到在SDK的`TrackingAccessibilityDelegate#sendAccessibilityEvent`方法里面，发出了事件上报。

那么View在点击方法的内部实现里有调用`sendAccessibilityEvent`方法吗？通过View处理点击事件时调用的`View.performClick`方法，看一下源码：

```java
public boolean performClick() {
    final boolean result;
    final ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnClickListener != null) {
        playSoundEffect(SoundEffectConstants.CLICK);
        li.mOnClickListener.onClick(this);
        result = true;
    } else {
        result = false;
    }
    sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
    return result;
}
...
public void sendAccessibilityEvent(int eventType) {
    if (mAccessibilityDelegate != null) {
        mAccessibilityDelegate.sendAccessibilityEvent(this, eventType);
    } else {
        sendAccessibilityEventInternal(eventType);
    }
}
...
public void setAccessibilityDelegate(@Nullable AccessibilityDelegate delegate) {
    mAccessibilityDelegate = delegate;
}
```

由此可以见在RootView开始绘制的时候，给View注册AccessibilityDelegate可以监测到它的点击事件。



## 4.2 可视化埋点的难点和问题

上面简单分析了Mixpanel和SensorsSDK可视化埋点的基本实现，里面还有一个难点需要仔细琢磨，那就是**如何唯一标识App中的一个View？**需要记录View的哪些信息，如何生成View的唯一ID，保证在不同手机上这些ID是固定的，而且保证App每次启动，ID也不会变化，同时ID也要能应对一定程度的界面调整。这里有两篇网易的博客，可以仔细看看。

- [http://www.infoq.com/cn/presentations/netease-happy-to-no-burial-point-data-collection-practice-road](http://www.infoq.com/cn/presentations/netease-happy-to-no-burial-point-data-collection-practice-road)
- [http://www.jianshu.com/p/b5ffe845fe2d](http://www.jianshu.com/p/b5ffe845fe2d)


另外在网上看到有网友提出，setAccessibilityDelegate来监测View的点击对大多数厂商的机型和版本都是可以的，但是有部分机型是无法成功捕获监控到点击事件。从View的标识生成，以及监测原理来讲，这个方案的稳定性存在一些疑问。

## 4.3 参考资料

- sensorsdata git，包含了Android、iOS、js、JAVA等多个版本的SDK：[https://github.com/sensorsdata](https://github.com/sensorsdata)
- Mixpanel git，包含了Android、iOS、js、JAVA等多个版本的SDK：[https://github.com/mixpanel](https://github.com/mixpanel)
- 网易移动端数据收集和分析博客：[http://www.jianshu.com/c/ee326e36f556](http://www.jianshu.com/c/ee326e36f556)

# 五、总结

最后简单总结一下几种方案的优缺点和使用场景，在实际应用中多种方式配合使用，平衡效率和可靠性，适合自己的业务才是最好的。

|埋点方案|优点|缺点|适用场景|
|:---:|:---|:---|:---|
|代码埋点|可以按照业务上报详细、定制化的数据|需要开发人员参与，更新维护成本高，无法获得历史数据|对上下文理解要求较高的业务数据|
|全埋点|对发人员依赖低，可以全量上报一类通用数据，可以拿到历史数据|数量量太大，占用更多资源，且无法收集业务上下文数据|上下文相对独立的、通用的数据|
|可视化埋点|对开发人员依赖低，可以按照业务需求上报数据，对上下文数据有一定收集能力|圈选事件有一定的操作难度，事件需要被更新时无法获得历史数据，界面变化时圈选的元素可能失效|业务上下文数据相对简单，操作交互比较固定的界面|
