---
layout: post
title: Android埋点技术分析
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

埋点，是对网站、App或者后台等应用程序进行数据采集的一种方法。通过埋点，可以收集用户在应用中的产生行为，进而用于分析和优化产品后续的体验，也可以为产品的运营提供数据支撑，其中常见的指标有PV、UV、页面时长和按钮的点击等。

采集行为数据时，通常需要在Web页面/App里面添加一些代码，当用户的行为达到某种条件时，就会向服务器上报用户的行为。其实添加这些代码的过程就可以叫做“埋点”，在很久以前就已经出现了这种技术。随着技术的发展和大家对数据采集要求的不断提高，我认为埋点的技术方案走过了下面几个阶段：

- **代码埋点：**代码埋点是指开发人员按照产品/运营的需求，在Web页面/App的源码里面添加行为上报的代码，当用户的行为满足某一个条件时，这些代码就会被执行，向服务器上报行为数据。这种方案是最基础的方案，每次增加或者修改数据上报的条件，都需要开发人员的参与，并且只能在下一个版本上线后才能看到效果。很多公司都提供了这类数据上报的SDK，将行为上报的后台服务器接口封装成了简单的客户端SDK接口。开发者可以通过嵌入这类SDK，在埋点的地方调用少量的代码就可以上报行为数据。

- **全埋点：**全埋点指的是将Web页面/App内产生的所有的、满足某个条件的行为，全部上报到后台服务器。例如把App中所有的按钮点击都进行上报，然后由产品/运营去后台筛选所需要的行为数据。这种方案的优点非常明显，就是可以不用在新增/修改行为上报条件时，再找开发人员去修改埋点的代码。然而它的缺点也和优点一样明显，那就是上报的数据量比代码埋点大很多，里面可能很多是没有价值的数据。此外，这种方案更倾向于独立去看待用户的行为，而没有关注行为的上下文，给数据分析带来了一些难度。很多公司也提供了这类功能的SDK，通过静态或者动态的方式，“Hook”了原有的App代码，从而实现了行为的监测，在数据上报时通常是采用累积多条再上报的方案来合并请求。

> hook直译是钩子的意思，以前学信息安全的时候在windows上听到过，大体意思是通过某种手段去改变系统API的一个行为，绕过系统的某个方法，或者改变系统的工作流程。在这里其实是指把本来要执行某个方法的对象替换成另一个，一般用的是反射或者代理，需要找到hook的代码位置，甚至还可以在编译阶段实现替换。

- **可视化埋点：**可视化埋点是指产品/运营在Web页面/App的界面上进行圈选，配置需要监测界面上哪一个元素，然后保存这个配置，当App启动时会从后台服务器获得产品/运营预先圈选好的配置，然后根据这份配置监测App界面上的元素，当某一个元素满足条件时，就会上报行为数据到后台服务器。有了全埋点技术方案，从体验优化的角度很容易想到按需埋点，可视化埋点就是一种按需配置埋点的方案。现在也有一些公司提供了这类SDK，圈选监测元素时，一般都是提供一个Web管理界面，手机在安装并初始化了SDK之后，可以和管理界面了连接，让用户在Web管理界面上配置需要监测的元素。


业界有多家SDK都支持上面介绍的3种埋点方案中的一种或者全部，例如Mixpanel、Sensorsdata、TalkingData、GrowingIO、诸葛IO、Heap Analytics、MTA、Umeng Analytics、百度，只是大家对后两种埋点的称呼不完全相同，有的叫无埋点或者codeless埋点。由于[Mixpanel](https://github.com/mixpanel/mixpanel-android)（支持代码埋点、可视化埋点）和[Sensorsdata](https://github.com/sensorsdata/sa-sdk-android)（全部支持）都开源了自己的全部SDK，技术方案也比较类似，下面以它们的Android SDK为例，简单分析一下3种埋点方案的技术实现。关于JS的SDK技术实现，可以看下我的另一篇博客-[JS埋点SDK技术分析](http://unclechen.github.io/2017/12/24/JS%E5%9F%8B%E7%82%B9SDK%E6%8A%80%E6%9C%AF%E5%88%86%E6%9E%90/)。

<!-- more -->

# 二、代码埋点

包含Mixpanel SDK在内的大部分SDK，都会把这种埋点方案封装成一个比较简单的接口，在这里是`track(String eventName, JSONObject properties)`，开发者在调用这个接口时，可以把一个事件名称和事件的属性传入，然后就可以上报到后台了。

在实现上，Mixpanel SDK默认采用一条HandlerThread线程来处理事件，当开发者调用`track(String eventName, JSONObject properties)`方法时，**主线程切换到HandlerThread**当中，并先将事件存入数据库。然后看SDK中是否累计到了40个事件，如果累计到40个事件的话，就合并它们上报到后台。

当开发者设置为debug模式，或者手动调用`flush`接口时，可以立即上报累计的所有事件，不过由于只有一条线程，所以如果在flush的时候，前面的事件还没有处理完成，SDK会间隔1分钟再次去处理后面的这些事件。

开发者可以设置累计上报的事件数量阈值、事件阻塞时再次尝试上报的时间间隔等。这种方案比较基础，相信大部分开发者都接触过，不需要过多分析。

# 三、全埋点

## 3.1 AOP基础

Mixpanel现在的Android SDK没有提供这个功能，但是神策Android SDK提供了，实现方式是AOP。

> 那么什么是AOP？AOP为Aspect Oriented Programming的缩写，意为：面向切面编程，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。AOP是OOP的延续，是软件开发中的一个热点，也是Spring框架中的一个重要内容，是函数式编程的一种衍生范型。利用AOP可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合度降低，提高程序的可重用性，同时提高了开发的效率。（from baidu baike）

> 简而言之，AOP是可以通过预编译方式和运行期动态代理实现在不修改源代码的情况下给程序动态统一添加功能的一种技术。

**Sensors Analytics AndroidSDK全埋点的实现就是通过在代码编译阶段，找到源代码中需要上报事件的位置，插入SDK的事件上报代码。它用到的框架是[AspectJ](https://www.eclipse.org/aspectj/)。**

说到这里，需要简单了解一下AspectJ以及它里面的一些概念.它是AOP的领跑者，在很多地方我们可以看到它的身影，例如JakeWharton大神贡献的一个注解日志和性能调优框架[Hugo](https://github.com/JakeWharton/hugo)，在Spring框架里面也有应用到AspectJ的概念（虽然Spring AOP的实现是用的动态代理）。我理解AspectJ里面的主要几个概念有：

- **JPoint：**代码切点（就是我们要插入代码的地方）
- **Aspect：**代码切点的描述
	- **Pointcut：**描述切点具体是什么样的点，如函数被调用的地方（`Call(MethodSignature)`）、函数执行的内部（`execution(MethodSignature)`）
	- **Advice：**描述在切点的什么位置插入代码，如在Pointcut前面（`@Before`）还是后面（`@After`），还是环绕整个Pointcut（`@Around`）

由此可见，在实现AOP功能时，需要做下面几件事：

- 定义一个Aspect，这个Aspect里面必须有Pointcut和Advice两个属性
- 编写在匹配到符合Pointcut和Advice描述的代码时，需要注入的代码
- 在代码编译时，通过特殊的java编译器（Aspect的ajc编译器），找到符合我们定义的Aspect的代码，将需要注入的代码插入到Advice指定的位置。

如果你对AspectJ有了解的话，已经可以猜到SDK内部是怎么实现全埋点的了；如果没有接触，我觉得也不用急于全面地去学习AspectJ，毕竟AspectJ的功能很强大（可远不止前置、后置这么简单的增强），埋点这种业务只用到了AspectJ当中的一小部分功能而已，可以直接看下面的分析。

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

采用AspectJ框架编写的代码，想要注入原来的工程的代码，需要在`/app/build.gradle`中引用ajc编译器，脚本如下：

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

完成上面两步，就可以实现在`android.view.View.OnClickListener.onClick(android.view.View)`方法中插入我们的数据上报代码了。我们在demo代码中加一个Button，并给它set一个OnClickListener，编译一下代码，查看`/build/intermediates/classes/debug/`里面class文件，经过ajc编译之后，原始代码中插入了Aspect的代码，并调用了`ViewOnClickListenerAspectj`里面的`onViewClickAOP`方法。

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

AspectJ的基本用法就是这样，除了对`OnClickListener`进行替换，理论上可以对任何已知的方法进行替换，所以在埋点SDK中还可以采用对RatingBar、CheckBox、RadioButton等控件的点击进行监听。

神策AndroidSDK借助AspectJ插入Aspect代码，是一种静态代理的方式。静态的全埋点方案，本质上是在程序没有运行之前，通常是编译或者链接的阶段，对字节码进行修改，插入事件上报的代码。

修改字节码除了这种方案之外，还有Android Gradle插件提供的trasform api（1.5.0版本以上）、ASM、Javassist。在网易乐得的埋点方案，Nuwa热修复项目都可以见到这些技术的实践。

## 3.3 AspectJ相关资料

- Aspect Oriented Programming in Android：[https://fernandocejas.com/2014/08/03/aspect-oriented-programming-in-android/](https://fernandocejas.com/2014/08/03/aspect-oriented-programming-in-android/)
- AOP之AspectJ全面剖析in Android：[http://www.jianshu.com/p/f90e04bcb326](http://www.jianshu.com/p/f90e04bcb326)
- 沪江开源了一个叫做AspectJX的插件，扩展了AspectJ，除了对src代码进行AOP，还支持kotlin、工程中引用的jar和aar进行AOP：[https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx](https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx)
- 关于 Spring AOP (AspectJ) 你该知晓的一切：[http://blog.csdn.net/javazejian/article/details/56267036](http://blog.csdn.net/javazejian/article/details/56267036)

## 3.4 其他思路

上面分析了以AspectJ为代表的**“静态Hook”**实现方案，有没有其他办法可以不修改源代码，只是在App运行的时候去**“动态Hook”**点击行为的处理呢？答案是肯定的，JDK里面已经有动态代理了，底层通过反射大法实现，可以从这个角度出发，看下怎么实现点击事件的监测上报。

在[android.view.View.java](https://github.com/aosp-mirror/platform_frameworks_base/blob/master/core/java/android/view/View.java)的源码（`API>=14`）中，有这么几个关键的方法：

```java
    // getListenerInfo方法：返回所有的监听器信息mListenerInfo
    ListenerInfo getListenerInfo() {
        if (mListenerInfo != null) {
            return mListenerInfo;
        }
        mListenerInfo = new ListenerInfo();
        return mListenerInfo;
    }
    
    // 监听器信息
    static class ListenerInfo {
        ... // 此处省略各种xxxListener
        /**
         * Listener used to dispatch click events.
         * This field should be made private, so it is hidden from the SDK.
         * {@hide}
         */
        public OnClickListener mOnClickListener;

        /**
         * Listener used to dispatch long click events.
         * This field should be made private, so it is hidden from the SDK.
         * {@hide}
         */
        protected OnLongClickListener mOnLongClickListener;

        ...
    }
    ListenerInfo mListenerInfo;
    
    // 我们非常熟悉的方法，内部其实是把mListenerInfo的mOnClickListener设成了我们创建的OnclickListner对象
    public void setOnClickListener(@Nullable OnClickListener l) {
        if (!isClickable()) {
            setClickable(true);
        }
        getListenerInfo().mOnClickListener = l;
    }
    
    /**
     * 判断这个View是否设置了点击监听器
     * Return whether this view has an attached OnClickListener.  Returns
     * true if there is a listener, false if there is none.
     */
    public boolean hasOnClickListeners() {
        ListenerInfo li = mListenerInfo;
        return (li != null && li.mOnClickListener != null);
    }
```

通过上面几个方法可以看到，点击监听器其实被保存在了**`mListenerInfo.mOnClickListener`**里面。那么实现**Hook点击监听器**时，只要将这个`mOnClickListener`替换成我们包装的**点击监听器代理对象**就行了。简单看一下实现思路：

**1. 创建点击监听器的代理类**

```
    // 点击监听器的代理类，具有上报点击行为的功能
    class OnClickListenerWrapper implements View.OnClickListener {
        // 原始的点击监听器对象
        private View.OnClickListener onClickListener;

        public OnClickListenerWrapper(View.OnClickListener onClickListener) {
            this.onClickListener = onClickListener;
        }

        @Override
        public void onClick(View view) {
            // 让原来的点击监听器正常工作
            if(onClickListener != null){
                onClickListener.onClick(view);
            }
            // 点击事件上报，可以获取被点击view的一些属性
            track(APP_CLICK_EVENT_NAME, getSomeProperties(view));
        }
    }
```

**2. 反射获取一个View的mListenerInfo.mOnClickListener，替换成代理的点击监听器**

```
    // 对一个View的点击监听器进行hook
    public void hookView(View view) {
        // 1. 反射调用View的getListenerInfo方法（API>=14），获得mListenerInfo对象
        Class viewClazz = Class.forName("android.view.View");
        Method getListenerInfoMethod = viewClazz.getDeclaredMethod("getListenerInfo");
        if (!getListenerInfoMethod.isAccessible()) {
            getListenerInfoMethod.setAccessible(true);
        }
        Object mListenerInfo = listenerInfoMethod.invoke(view);
        
        // 2. 然后从mListenerInfo中反射获取mOnClickListener对象
        Class listenerInfoClazz = Class.forName("android.view.View$ListenerInfo");
        Field onClickListenerField = listenerInfoClazz.getDeclaredField("mOnClickListener");
        if (!onClickListenerField.isAccessible()) {
            onClickListenerField.setAccessible(true);
        }
        View.OnClickListener mOnClickListener = (View.OnClickListener) onClickListenerField.get(mListenerInfo);
        
        // 3. 创建代理的点击监听器对象
        View.OnClickListener mOnClickListenerWrapper = new OnClickListenerWrapper(mOnClickListener);
        
        // 4. 把mListenerInfo的mOnClickListener设成新的onClickListenerWrapper
        onClickListenerField.set(mListenerInfo, mOnClickListenerWrapper);
        // 用这个似乎也可以：view.setOnClickListener(mOnClickListenerWrapper);     
    }
```

注意，如果是`API<14`的话，mOnClickListener直接是直接以一个Field保存在View对象中的，没有ListenerInfo，因此反射的次数要更少一些。

**3. 对App中所有的View进行Hook**

我们在分析的是全埋点，那么怎样把App里面所有的View点击都Hook到呢？有两种方式：

- 第一种：当Activity创建完成后，开始从Activity的DecorView开始自顶向下深度遍历ViewTree，遍历到一个View的时候，对它进行hookView操作。这种方式有点暴力，由于这里面遍历ViewTree的时候用到了大量反射，性能会有影响。

- 第二种：比第一种方式稍微优秀一些，来源是一个Github上的开源库[AndroidTracker](https://github.com/foolchen/AndroidTracker)（Kotlin实现）。他的处理方式是当Activity创建完成后，在DecorView中添加一个透明的View作为子View，在这个子View的onTouchEvent方法中，根据触摸坐标找到屏幕中包含了这个坐标的View，再对这些View尝试进行hookView操作。**这种方式比较取巧，首先是拿到了手指按下的位置，根据这个位置来找需要被Hook的View，避免了在遍历ViewTree的同时对View进行反射。具体实现是在遍历ViewTree中的每个View时，判断这个View的坐标是否包含手指按下的坐标，以及View是否Visible，如果满足这两个条件，就把这个View保存到一个ArrayList<View>hitViews。然后再遍历这个ArrayList里面的View，如果一个View#hasOnClickListeners返回true，那么才对他进行hookView操作。**

整体来看，动态Hook的思路用到了反射，难免对程序性能产生影响，如果要采用这种方式实现全埋点方案，还需要好好评估。


# 四、可视化埋点

第三章介绍的是App全埋点，显然这种方式产生的数据太多，无论是对用户资源的节约，还是后续的数据分析都不太好。那么能否也借助动态代理这类的Hook技术，只对我们感兴趣的控件进行埋点呢？这就是可视化埋点。

## 4.1 可视化埋点-技术实现

可视化埋点，需要经过两个步骤，可以由非技术人员操作完成。

- 第一步，使用已经嵌入了SDK的App连接管理界面，当手机App与后台同步时，后台管理界面上会显示和手机App一样的界面，用户可以在管理界面上用鼠标选择需要监测的元素，设置事件名称，保存这个配置。（也有一些SDK，比如GrowingIO的SDK圈选操作是在手机悬浮了一个原点，拖动圆点到需要监测的元素上来设置埋点位置的，不管是什么方式本质上是一样的，需要保存一份配置到后台）。
- 第二步，嵌入了SDK的App启动时，会从服务器获取到一份配置，再根据这份配置去检测App中的界面及其元素，满足配置的条件时向服务器上报事件。

这里面最重要的技术点就是如何把手机上需要埋点的元素记录下来，然后根据配置信息找到需要埋点的控件，再替换这个控件的交互事件处理方法（如点击、长按等）。下面以Mixpanel、SensorsdataSDK为例（这两个SDK实现是一样的），简单分析一下技术方案的实现。

### 4.1.1 圈选需要监测的元素，保存配置

**1.创建WebSocket连接后台**

采用WebSocket连接是因为要让手机和后台长时间保持连接，是一个**持续的、实时的双向通信**，WebSocket正适合这种场景。

> 在Mixpanel和神策SDK里面其实都用到了开源的[Java-WebSocket](https://github.com/TooTallNate/Java-WebSocket)实现。此外，还有一个非常著名的Android同屏工具[Vysor](https://www.vysor.io/)，里面也有一个基于WebSocket的网络框架[AndroidAsync](https://github.com/koush/AndroidAsync)。如果对WebSocket感兴趣，可以看看它们。这里其实只要是用Java实现的WebSocket通信就行。

**2.把App界面截图和里面的子View信息发送到后台**

创建WebSocket连接后，SDK会在主线程中，对App中启动的Activity进行扫描，找到界面的RootView（其实是DecorView）。在查找RootView的同时，会采用反射调用View类`createSnapshot`方法对RootView进行截图，从而实现了对屏幕的截图。

截图之后，SDK内部会判断图片的hash值，如果图片发生了变化，会采用**先序**的方式遍历Activity的ViewTree，遍历同时读取View的属性（id、top、left、width、height、class名称、layoutRules等等）。下面举一个栗子：

![](https://ws1.sinaimg.cn/large/006tKfTcly1fn1f3fcrrkj30rs06twfj.jpg)

一个简单的Activity，ContentView里面有一个LineaLayout，LinearLayout里面放了一个Button。先序遍历Activity的ViewTree后，SDK会把下面这些数据传到WebSocket的服务器（数据有点多，大概有13k，数据主要来自截图）：

```
{
    "type": "snapshot_response", 
    "payload": {
        "activities": [
            {
                "activity": "com.sensorsdata.analytics.android.demo.MainActivity", 
                "scale": 0.3809524, 
                "serialized_objects": {
                    "rootObject": 88528516, 
                    "objects": [
                        {
                            "hashCode": 88528516, 
                            "id": -1, 
                            "index": -1, 
                            "sa_id_name": null, 
                            "top": 0, 
                            "left": 0, 
                            "width": 1080, 
                            "height": 1920, 
                            "scrollX": 0, 
                            "scrollY": 0, 
                            "visibility": 0, 
                            "translationX": 0, 
                            "translationY": 0, 
                            "classes": [
                                "com.android.internal.policy.DecorView", 
                                "android.widget.FrameLayout", 
                                "android.view.ViewGroup", 
                                "android.view.View"
                            ], 
                            "subviews": [
                                57495077, 
                                150453242
                            ]
                        }, 
                        {
                            "hashCode": 57495077, 
                            "id": 16908822, 
                            "index": 0, 
                            "sa_id_name": null, 
                            "top": 0, 
                            "left": 0, 
                            "width": 1080, 
                            "height": 1920, 
                            "scrollX": 0, 
                            "scrollY": 0, 
                            "visibility": 0, 
                            "translationX": 0, 
                            "translationY": 0, 
                            "classes": [
                                "com.android.internal.widget.ActionBarOverlayLayout", 
                                "android.view.ViewGroup", 
                                "android.view.View"
                            ], 
                            "subviews": [
                                12620808, 
                                88713121
                            ]
                        }, 
                        {
                            "hashCode": 12620808, 
                            "id": 16908290, 
                            "index": 0, 
                            "sa_id_name": "android:content", 
                            "top": 210, 
                            "left": 0, 
                            "width": 1080, 
                            "height": 1710, 
                            "scrollX": 0, 
                            "scrollY": 0, 
                            "visibility": 0, 
                            "translationX": 0, 
                            "translationY": 0, 
                            "classes": [
                                "android.widget.FrameLayout", 
                                "android.view.ViewGroup", 
                                "android.view.View"
                            ], 
                            "subviews": [
                                150314438
                            ]
                        }, 
                        {
                            "hashCode": 150314438, 
                            "id": -1, 
                            "index": 0, 
                            "sa_id_name": null, 
                            "top": 0, 
                            "left": 0, 
                            "width": 1080, 
                            "height": 1710, 
                            "scrollX": 0, 
                            "scrollY": 0, 
                            "visibility": 0, 
                            "translationX": 0, 
                            "translationY": 0, 
                            "classes": [
                                "android.widget.LinearLayout", 
                                "android.view.ViewGroup", 
                                "android.view.View"
                            ], 
                            "subviews": [
                                104340701
                            ]
                        }, 
                        {
                            "hashCode": 104340701, 
                            "id": 2131427422, 
                            "index": 0, 
                            "sa_id_name": "buttonTest", 
                            "top": 0, 
                            "left": 0, 
                            "width": 1080, 
                            "height": 126, 
                            "scrollX": 0, 
                            "scrollY": 0, 
                            "visibility": 0, 
                            "translationX": 0, 
                            "translationY": 0, 
                            "classes": [
                                "android.widget.Button", 
                                "android.widget.TextView", 
                                "android.view.View"
                            ], 
                            "subviews": [ ]
                        }, 
                        {
                            "hashCode": 88713121, 
                            "id": 16908669, 
                            "index": 0, 
                            "sa_id_name": null, 
                            "top": 63, 
                            "left": 0, 
                            "width": 1080, 
                            "height": 147, 
                            "scrollX": 0, 
                            "scrollY": 0, 
                            "visibility": 0, 
                            "translationX": 0, 
                            "translationY": 0, 
                            "classes": [
                                "com.android.internal.widget.ActionBarContainer", 
                                "android.widget.FrameLayout", 
                                "android.view.ViewGroup", 
                                "android.view.View"
                            ], 
                            "subviews": [
                                164355104, 
                                161393113
                            ]
                        }, 
                        {
                            "hashCode": 164355104, 
                            "id": 16908668, 
                            "index": 0, 
                            "sa_id_name": null, 
                            "top": 0, 
                            "left": 0, 
                            "width": 1080, 
                            "height": 147, 
                            "scrollX": 0, 
                            "scrollY": 0, 
                            "visibility": 0, 
                            "translationX": 0, 
                            "translationY": 0, 
                            "classes": [
                                "android.widget.Toolbar", 
                                "android.view.ViewGroup", 
                                "android.view.View"
                            ], 
                            "subviews": [
                                222758006, 
                                64817783
                            ]
                        }, 
                        {
                            "hashCode": 222758006, 
                            "id": -1, 
                            "index": 0, 
                            "sa_id_name": null, 
                            "top": 38, 
                            "left": 42, 
                            "width": 553, 
                            "height": 71, 
                            "scrollX": 0, 
                            "scrollY": 0, 
                            "visibility": 0, 
                            "translationX": 0, 
                            "translationY": 0, 
                            "classes": [
                                "android.widget.TextView", 
                                "android.view.View"
                            ], 
                            "subviews": [ ]
                        }, 
                        {
                            "hashCode": 64817783, 
                            "id": -1, 
                            "index": 0, 
                            "sa_id_name": null, 
                            "top": 0, 
                            "left": 1080, 
                            "width": 0, 
                            "height": 147, 
                            "scrollX": 0, 
                            "scrollY": 0, 
                            "visibility": 0, 
                            "translationX": 0, 
                            "translationY": 0, 
                            "classes": [
                                "android.widget.ActionMenuView", 
                                "android.widget.LinearLayout", 
                                "android.view.ViewGroup", 
                                "android.view.View"
                            ], 
                            "subviews": [ ]
                        }, 
                        {
                            "hashCode": 161393113, 
                            "id": 16908673, 
                            "index": 0, 
                            "sa_id_name": null, 
                            "top": 0, 
                            "left": 0, 
                            "width": 0, 
                            "height": 0, 
                            "scrollX": 0, 
                            "scrollY": 0, 
                            "visibility": 8, 
                            "translationX": 0, 
                            "translationY": 0, 
                            "classes": [
                                "com.android.internal.widget.ActionBarContextView", 
                                "com.android.internal.widget.AbsActionBarView", 
                                "android.view.ViewGroup", 
                                "android.view.View"
                            ], 
                            "subviews": [ ]
                        }, 
                        {
                            "hashCode": 150453242, 
                            "id": 16908335, 
                            "index": 0, 
                            "sa_id_name": "android:statusBarBackground", 
                            "top": 0, 
                            "left": 0, 
                            "width": 1080, 
                            "height": 63, 
                            "scrollX": 0, 
                            "scrollY": 0, 
                            "visibility": 0, 
                            "translationX": 0, 
                            "translationY": 0, 
                            "classes": [
                                "android.view.View"
                            ], 
                            "subviews": [ ]
                        }
                    ]
                }, 
                "image_hash": "785C4DC3B01B4AFA56BA0E3A56CE8657", 
                "screenshot": "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fMttK5ojA0AORQNyKBqQQ9GAHIoG5FA0IIeiATkUDcihaEAORQNyKBrw7TZw4MZfc9nxuU6AbzVjNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuRQNCCHogE5FA3IoWhADkUDcigakEPRgByKBuT4f1ZVvXKwGu9EAAAAAElFTkSuQmCC"
            }
        ], 
        "snapshot_time_millis": 403
    }
}
```

最后面的`screenshot`就是手机的截图，以base64编码。

> **为了简化分析，在上面的数据里面没有体现View的一些属性，例如Button上显示的text文字，实际上在遍历ViewTree里面每一个View的同时也会上报这个信息，因为我们的Activity和里面View大部分情况下都会是复用的，一个购物的Activity界面，里面的按钮可以显示不同的文字，我们需要统计不同商品的点击次数，就必须要知道按钮上显示的文字是什么。**

对于View来讲，关键信息有这些：

- activity：Activity类名
- hashcode：view的hashcode
- id：在Apk中的id
- index：在父控件中的同类元素的顺序，如果是根View，那么为-1，如果父View没有多个同类型的子View，那么为0（例如LinearLayout中只有一个Button）
- sa_id_name：在Apk中的控件的id的字符串名称，例如android:id="@+id/button2"，结果就是`button2`
- top：距离屏幕上边距
- left：距离屏幕的左边距
- width：宽
- height：高
- classes：View自身以及所有的父类类名，是一个数组，这里决定了一个View到底可以有哪些交互，比如点击、长按等
- subviews：子View的hashcode，是一个数组

**3.保存待监测的元素的关键信息**

将上面收集到数据发送到连接的WebSocket后台，由后台解析之后，可以把App界面的截图展示在Web页面。然后把可以监测的元素以方框的形式添加在界面上提示用户（web页面实现时，我推测只需要用到这个View的left、top、width、height属性在html上加一个div标签，然后设置一个有颜色的border属性即可）。用户可以在这个Web页面点击需要监测的元素，设置这个元素的事件名称（event_type和event_name），点击保存。保存一个需要监测的元素时，需要保存这个元素在当前Activity的ViewTree的路径`path`，以及这个View在父控件中的`index`，具体有下面几个信息：

- target_activity：View所在的Activity类名
- event_type：事件类型，例如点击事件
- event_name：事件名称
- trigger_id：事件id
- path：View在ViewTree中查找路径
	- prefix：表示是否需要监测这个View的兄弟元素，当为`shortest`时，表示只匹配到索引为index那一个元素，否则匹配所有的父控件下面所有的同类子元素
	- view_class：view的类名
	- index：View在父控件中同类元素的下标索引，**这个属性一定程序上可以对抗ViewTree的更新导致的元素监测失效问题，因为父控件加入一个不同类的元素时，index的值不会发生改变**
	- id：View在Apk中的id
	- sa_id_name：View在Apk中的id的字符串名称


### 4.1.2 获取配置，监测元素的行为，上报事件

**1.获取配置**

SDK启动时，会从服务器拉取一份JSON格式的配置，保存到sharedPreference里，同时SDK会扫描`android.R`文件里面的资源id和资源的name并缓存起来。

SDK得到配置之后，解析成JSON对象，读取`event_bindings`字段，再进一步读取`events`字段，这个字段下面包含了一个数组，数组的每个元素都描述了一类事件，并包含了这类事件需要监测的元素所在的Activity和元素的路径。这份配置基本上是这样的一个结构：

```json
event_bindings: {
    events:[
        {
            target_activity: ""
            event_name: "",
            event_type: "",
            ...
            path: [
                {
                    prefix:
                    view_class:
                    index:
                    id:
                    sa_id_name:
                }, 
                {
                    ...
                }
                ...
            ]
        }
    ]
}
```

收到了这份配置之后，SDK会把根据每个event信息，生成一个`ViewVisitor`。`ViewVisitor`的作用就是把`path`数组里面指向的所有View元素都找到，并且根据event_type，**给这个View设置相应的行为监测器**，当这个View发生指定行为时，监测器就会监测到，并上报行为。

在生成ViewVisitor之后，SDK内部是以`Map<activity, ViewVisitor>`结构保存它们的，这也比较容易理解，毕竟我们的界面是随着一个一个的Activity被create，onResume之后才被用户看见的嘛。其实在ViewVisitor对象中还有一个`PathFinder`对象，这个对象负责在ViewTree中根据path去查找View（这里其实是一个树的查找问题）。

**2.监测元素，上报事件**

`ViewVisitor`是怎么给View设置监听器，监测元素的产生的行为呢？**答案就是`View.AccessibilityDelegate`。**

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

**以点击事件`TYPE_VIEW_CLICKED`为例**，当Activity界面的RootView开始绘制的时候（ViewTreeObserver.OnGlobalLayoutListener的onGlobalLayout回调时），ViewVisitor也会开始寻找指定的View，并给这个View设置新的AccessibilityDelegate。简单看一下这个新的View.AccessibilityDelegate是怎么写的：

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

这么说View的点击处理方法中应该要调用`sendAccessibilityEvent`才行，那么View在点击方法的内部实现里有调用`sendAccessibilityEvent`方法吗？看一下View处理点击事件 - `View.performClick`的源码：

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

由此可见View的点击处理内部确实调用到了`sendAccessibilityEvent `，所以在RootView开始绘制的时候，给View注册AccessibilityDelegate可以监测到它的点击事件。可视化埋点这里也是一种**“动态Hook”**的实现，不过没有采用第三章中介绍的反射方案，而是采用了AccessibilityDelegate来实现，这种方式效率上显然会更好一些。

> 在网上看到有网友提出，setAccessibilityDelegate来监测View的点击对大多数厂商的机型和版本都是可以的，但是有部分机型是无法成功捕获监控到点击事件。从View的标识生成，以及监测原理来讲，这个方案的稳定性存在一些疑问。

## 4.2 可视化埋点的难点和优化

上面简单分析了Mixpanel和SensorsSDK可视化埋点的基本实现，里面最重要有一个技术点值得仔细琢磨，那就是**如何唯一标识App中的一个View？由于View是长在ViewTree上的一个节点，那么用纵向的路径，以及横向的下标应该可以标识一个View。**

- 纵向的路径：是指从根View到这个View的父控件的路径上经过的每一个节点
- 横向的下标：是指这个View在父控件中的同类元素的下标索引（例如一个LinearLayout中有两个Button，那么第一个Button的下标就是0，第二个Button的下标就是1，这种方式可以抵抗父控件中加入一个非Button类型的元素时对ViewTree的改变，保证仍然可以找到Button，但是无法抵抗父控件中加入同类型的元素）

上面仅仅提到了标识一个View的基本方法，但是有很多实际场景，会对View的查找造成毁灭性的影响，例如Fragment的变化，ListView中控件的复用等等，这里有两篇网易的博客，里面对一些场景的优化做了详细地说明，可以仔细看看：

- [http://www.infoq.com/cn/presentations/netease-happy-to-no-burial-point-data-collection-practice-road](http://www.infoq.com/cn/presentations/netease-happy-to-no-burial-point-data-collection-practice-road)
- [http://www.jianshu.com/p/b5ffe845fe2d](http://www.jianshu.com/p/b5ffe845fe2d)


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
