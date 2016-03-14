---
layout: post
title: Android N App分屏模式完全解析（下）
date: '2016-03-12 10:00:00'
---

在[上篇](http://unclechen.github.io/2016/03/12/Android-N-App分屏模式完全解析-上篇/)中，介绍了什么是App分屏模式，以及如何设置我们的App来进入分屏模式。这次我们看一下，作为开发者，我们应该如何让自己的App进入分屏模式，当App进入分屏模式时，我们注意哪些问题。

简单地说，我认为除了保证分屏时App功能、性能正常以外，我们需要重点学习 ***如何在分屏模式下打开新的Activity*** 以及 ***如何实现跨App/Activity的拖拽功能***。

# 用分屏模式运行你的App

Android N中新增了一些方法来支持App的分屏模式。同时在分屏模式下，也禁用了App一些特性。

## 分屏模式下被禁用的特性

- 自定义[系统UI](http://developer.android.com/training/system-ui/index.html)，例如分屏模式下无法隐藏系统的状态栏。
- 无法根据屏幕方向来旋转App的界面，也就是说`android:screenOrientation`属性会被系统忽略。

## 分屏模式的通知回调、查询App是否处于分屏状态

最新的[Android N SDK](http://developer.android.com/preview/setup-sdk.html#docs-dl)中，`Activity`类中增加了下面的方法。

- inMultiWindow()：返回值为boolean，调用此方法可以知道App是否处于分屏模式。
- inPictureInPicture()：返回值为boolean，调用此方法可以知道App是否处于画中画模式。

> 注意：`画中画模式`其实是一个**特殊的**`分屏模式`，如果`mActivity.inPictureInPicture()`返回`true`，那么`mActivity.inMultiWindow()`一定也是返回`true`。

- onMultiWindowChanged(boolean inMultiWindow)：当Activity进入或者退出分屏模式时，系统会回调这个方法来通知开发者。回调的参数`inMultiWindow`为boolean类型，如果`inMultiWindow`为true，表示Activity进入分屏模式；如果`inMultiWindow`为false，表示退出分屏模式。
- onPictureInPictureChanged(boolean inPictureInPicture)：当Activity进入画中画模式时，系统会回调这个方法。回调参数`inPictureInPicture`为`true`时，表示进入了画中画模式；`inPictureInPicture`为`false`时，表示退出了画中画模式。

`Fragment`类中，同样增加了以上支持分屏模式的方法，例如`Fragment.inMultiWindow()`。

## 如何进入画中画模式

调用`Activity`类的`enterPictureInPicture()`方法，可以使得我们的App进入画中画模式。如果运行的设备不支持画中画模式，调用这个方法将不会有任何效果。更多画中画模式的资料，请参考[picture-in-picture](http://developer.android.com/intl/zh-cn/preview/features/picture-in-picture.html)。

## 在分屏模式下打开新的Activity

当你打开一个新的Activity时，只需要给Intent添加`Intent.FLAG_ACTIVITY_LAUNCH_TO_ADJACENT`，系统将***尝试***将它设置为与当前的Activity共同以分屏的模式显示在屏幕上。

**注意：**这里只是尝试，但这不一定是100%生效的，前一篇博客里也说过，假如新打开的Activity的`android:resizeableActivity`属性设置为`false`，就会禁止分屏浏览这个Activity。所以系统只是尝试去以分屏模式打开一个新的Activity，如果条件不满足，将不会生效！此外，我实际用`Android N Preview SDK`实践的时候发现这个`FLAG`实际得值是`FLAG_ACTIVITY_LAUNCH_ADJACENT`，并非是`FLAG_ACTIVITY_LAUNCH_TO_ADJACENT`。

当满足下面的条件，系统会让这两个Activity进入分屏模式：

- 当前Activity已经进入到分屏模式。
- 新打开的Activity支持分屏浏览（即**android:resizeableActivity=true**）。

此时，给新打开的Activity，设置`                intent.addFlags(Intent.FLAG_ACTIVITY_LAUNCH_ADJACENT | Intent.FLAG_ACTIVITY_NEW_TASK);
`才会有效果。

![two-acts](/content/images/two-acts.png)

建议参考官方的Sample：[MultiWindow Playground Sample](https://github.com/googlesamples/android-MultiWindowPlayground)

那么为何还需要添加`FLAG_ACTIVITY_NEW_TASK`？看一下官方解释：

> 注意：在同一个Activity返回栈中，打开一个新的Activity时，这个Activity将会继承上一个Activity所有和`分屏模式`有关的属性。如果你想要在一个独立的窗口以分屏模式打开一个新的Activity，那么必须新建一个Activity返回栈。

此外，如果你的设备支持`自由模式`（官方名字叫**freeform**，暂且就这么翻译它，其实我认为这算也是一种尺寸更自由的分屏模式，上一篇博客里提到过如果设备厂商支持用户可以自由改变Activity的尺寸，那么就相当于支持`自由模式`，这将比普通的分屏模式更加自由），打开一个Activity时，还可通过`ActivityOptions.setLaunchBounds()`来指定新的Activity的尺寸和在屏幕中的位置。同样，这个方法也需要你的Activity已经处于分屏模式时，调用它才会生效。

## 支持拖拽

在[上一篇](http://unclechen.github.io/2016/03/12/Android-N-App分屏模式完全解析-上篇/)博客里也提到过，现在我们可以实现在两个分屏模式的App之间拖动内容了。Android N Preview SDK中，`View`已经增加支持App之间拖动的API。具体的类和方法，可以参考[N Preview SDK Reference](http://developer.android.com/preview/setup-sdk.html#docs-dl)

- android.view.DropPermissions：允许App接收拖拽的权限的token。
- View.startDragAndDrop()：[View.startDrag()](http://developer.android.com/intl/zh-cn/reference/android/view/View.html#startDrag(android.content.ClipData,%20android.view.View.DragShadowBuilder,%20java.lang.Object,%20int)) 的替代方法，需要传递`View.DRAG_FLAG_GLOBAL`来实现跨Activity拖拽。如果需要将URI的权限传递给接收方Activity，请根据需要设置`View.DRAG_FLAG_GLOBAL_URI_READ`或者`View.DRAG_FLAG_GLOBAL_URI_WRITE`。
- View.cancelDragAndDrop()：由拖拽的发起方调用，取消当前进行中的拖拽。
- View.updateDragShadow()：有拖拽的发起方调用，可以给当前进行的拖拽设置阴影。
- Activity.requestDropPermissions()：通过传递[DragEvent](http://developer.android.com/reference/android/view/DragEvent.html)中的[ClipData](http://developer.android.com/reference/android/content/ClipData.html)来请求内容URI的权限。

下面是我自己写的一个demo，实现了在分屏模式下，从一个App拖拽一个ImageView到另外一个App。

```
to be completed
```

# 在分屏模式下测试你的App

无论你是否将自己的App适配到了Android N，或者是支持分屏模式，都应该找个Android N的设备，来测试一下自己的App在分屏模式下会变成什么样。

## 设置你的测试设备

如果你有一台运行Android N的设备，它是默认支持分屏模式的。

## 如果你的App不是用Android N Preview SDK打包的

如果你的App是用`低于Android N Preview SDK`打包的，且你的Activity支持`横竖屏切换`。那么当用户在尝试使用分屏模式时，系统会强制将你的App进入分屏模式。（我在第一篇博客里提到过这个，Android N Preview的介绍视频中，很多Google家的App都可以进入分屏模式，但是打开它们的xml一看，其实`targetSDKVersion = 23`）

因此，如果你的App/Activity支持横竖屏切换，那么你应该尝试一下让自己的App分屏，看看当系统强制改变你的App尺寸时，用户是否还可以接受这种体验。如果你的App/Activity不支持横竖屏切换，那么你可以确认一下，看看当尝试进入分屏时，你的App是不是仍然能够保持全屏模式。

## 如果你给App设置了支持分屏模式

如果你使用了`Android N Preview SDK`来开发自己的App，那么应该按照下面的要点检查一下自己的App。

- 启动App，长按系统导航栏右下角的小方块（Google官方把这个叫做**Overview Button**），确保你的App可以进入分屏模式，且尺寸改变后仍然能正常工作。
- 启动任务管理器（即单击右下角的小方块），然后长按你App的标题栏，将它拖动到屏幕上的高亮区域。确保你的App可以进入分屏模式，且尺寸改变后仍然能正常工作。

这两点在[上一篇博客](http://unclechen.github.io/2016/03/12/Android-N-App分屏模式完全解析-上篇/)中介绍过，让自己的App进入分屏模式有三种方法。第三种方法，就是在打开自己的App时，用手指从右下角的小方块向上滑动，这样也可以使得正在浏览的App进入分屏模式。这种方法目前属于实验性功能，正式版不一定保留。

- 当你的App进入分屏后，通过拖动两个App中间的分栏上面的小白线，从而改变App的尺寸，观察App中各个UI元素是否正常显示。
- 如果你给自己的App/Activity设置了**最小尺寸**，可以尝试在改变App尺寸时，低于这个最小尺寸，观察App是不是会回到设定好的最小尺寸。
- 在进行上面几项测试时，请同时验证自己的App功能和性能是否正常，并注意一下自己的App在更新UI时是否花费了太长的时间。

这几项测试，其实主要强调的是，我们的App可以顺利的进入/退出分屏模式，且改变App的尺寸时，UI依然可以也非常顺滑。

这里我想多说一句，如果进入了分屏模式，要注意下App弹出的对话框，因为屏幕被两个App分成两块之后，对话框也是可以弹出两个的。这时对话框上的UI元素可能就会变得比较小了，如果我们的代码是写死的大小，例如对话框是一个WebView，就需要特别注意了，搞不好显示出来就缺了一块了，这里需要我们做好适配。

### 测试清单

关于功能、性能方面测试，还可以按照下面的操作来进行。

- 让App进入，再退出分屏模式，确保此时App功能正常。
- 让App进入分屏模式，激活屏幕上的另外一个App，让自己的App进入`可见、paused`状态。举了例子来讲，如果你的App是一个视频播放器，那么当用户点击了屏幕上另外一个App时，你的App不应该停止播放视频，即使此时你的Activity/Fragment已经接到了`onPaused()`回调。
- 让App进入分屏模式，拖动分栏上的小白线，改变App的尺寸。请在竖屏（两个App一上一下布局）和横屏（两个App一左一右布局）模式下分别进行改变尺寸的操作。确保App不会崩溃，各项功能正常，且UI的刷新没有花费太多时间。
- 在短时间内、多次、迅速地改变App尺寸，确保App没有崩溃，且没有发生内存泄露。关于内存使用方面的更详细注意事项，请参考[Investigating Your RAM Usage](http://developer.android.com/tools/debugging/debugging-memory.html)。
- 在不同的窗口设置的情况下，正常使用App，确保App功能正常，文字仍然可读，其他的UI元素也没有变得太小，用户仍然可以舒适地操作App。

这几项测试，其实主要是说当App在分屏模式下运行时，仍然可以保持性能的稳定，不会Crash也不会OOM。


## 如果你给App设置了禁止分屏模式

如果你给App/Activity设置了`android:resizableActivity="false"`，你应该试试当用户在Android N的设备上，尝试分屏浏览你的App时，它是否仍然能保持全屏模式。

以上就是参考Google最新的[multi-window](http://developer.android.com/intl/zh-cn/preview/features/multi-window.html)进行的实践，总结下，我认为有3点比较重要：

1. 如何让自己的App/Activity顺利的进入和退出分屏模式，这里的顺利主要是指功能、性能正常。
2. 如何在分屏模式下打开新的Activity。
3. 如何实现跨App/Activity的拖拽功能。

关于App分屏模式的学习就到这里了，欢迎大家一起交流。我们还发挥更多的想象力，比如是否可以利用跨应用拖拽实现更方便操作，更好的用户体验。









