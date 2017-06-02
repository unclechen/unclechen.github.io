---
layout: post
title: Android地理位置服务解析
date: '2016-09-02'
tags:
  - Android
  - 定位
categories: 
  - 技术
---

# 手机设备有哪几种定位方式

## GPS

基于卫星发射的信号，可以推算出手机到每颗卫星的距离，根据卫星的位置，推测出手机的位置。

这是一张简单的GPS定位原理图，需要一点数学知识，先不讨论这个细节，需要的同学看[这里](http://baike.baidu.com/view/193655.htm)。

![gps定位原理图](http://ww1.sinaimg.cn/large/65e4f1e6gw1f7fbixzcaxj20dw0bo750.jpg)

现在卫星信号全球都覆盖了，手机一般都有GPS芯片，因此可以实现定位。GPS方式准确度是最高的，走卫星通道，不需要联网就可以要使用。但是它的缺点也非常明显：

- 1.比较耗电; 
- 2.绝大部分用户默认不开启GPS模块，也不会长时间开着; 
- 3.从GPS模块启动到获取第一次定位数据，可能需要**比较长的时间**; 
- 4.**只能在户外使用**，当有遮挡物干扰时，几乎无法使用，如城市大楼密集的地方。

<!-- more -->

## WiFi

通过获取当前所连接的WiFi热点的一些信息，然后访问定位服务以获得经纬度坐标。

这是一张简单的WiFi定位原理图。

![WiFi定位原理图](http://ww3.sinaimg.cn/large/65e4f1e6gw1f7fbjjbmesj20b408a0t0.jpg)

因为WiFi热点一般都是固定位置，所以只要能知道手机连接的WiFi热点的位置，也就可以推算出手机的位置。而且由于手机一般连接的WiFi不会太远，所以其实精确度也不会太差。也不会像GPS那样需要耗时比较久才能获得位置信息。


## Cell-ID

采集到手机所连接的基站ID号(cellid)和其它的一些信息(MNC，MCC，LAC等)，然后通过网络访问定位服务，获取并返回对应的经纬度坐标。

这是一张简单的基站定位原理图。

![基站定位原理](http://ww2.sinaimg.cn/large/65e4f1e6gw1f7fbjts47yj20b40bqq3o.jpg)

现在各大运营商的基站已经覆盖了全国大部分地区，每个基站的ID号是全球唯一的，只要有手机信号，就能接收到周围基站的信号。基站定位的精确度不如GPS，但优点是能够在室内用，只要网络通畅就行。


其实各种定位方式，大体都是基于三角定位的原理，不过计算的时候会有一些自己的特点，这里先不深究背景知识了。下面进入正题。


# Android系统上如何获取地理位置

## 方法1：Google Play Service提供的API

这个不多说，因为国内不可用！！！ 

需要的同学可以自己爬梯子看下用法，比较简单：[https://developer.android.com/google/play-services/location.html](https://developer.android.com/google/play-services/location.html)

## 方法2：系统提供的原生API：主要就是系统的`android.location`中提供的两个类。

- **LocationManager：**和大多数系统提供的**SystemService**一样是单例，通过`locationManager = (LocationManager) context.getSystemService(Context.LOCATION_SERVICE);`来获取。
- **LocationListener：**非常典型的观察者模式，需要监听地理位置的时候，创建一个**Listener**，实现LocationListener中的几个回调方法。把Listener传给LocationManager，当地理位置变化的时候就会回调`onLocationChanged(Location location)`发出通知。
- 官方指导：[https://developer.android.com/guide/topics/location/strategies.html](https://developer.android.com/guide/topics/location/strategies.html)

## 方法3：使用百度、高德之类的地图SDK。

这简直就是大招了，各家都有自己的数据库，比起系统提供的API强太多了。这个这次也不说，各家的接入文档写的很清楚。


# 使用原生API采集地理位置的方法

下面介绍一下我对使用原生API的理解，毕竟不是所有场景都需要用到大招级别的sdk，有的情况我们需要自己实现定位服务。

## 1.首先需要了解**PROVIDER**

看过前面介绍的3种定位方式之后，可以很容易理解PROVIDER是什么。其实它就对应着地理位置采集的几种方式：

- LocationManager.GPS_PROVIDER：通过gps来获取地理位置的经纬度信息，优点：获取地理位置信息精确度高，缺点：只能在户外使用，**耗时，耗电**。
- LocationManager.NETWORK_PROVIDER：通过移动网络的基站或者WiFi来获得地理位置，优点：只要有网络，获取速度快，耗电低，在室内室外都可以使用。
- LocationManager.PASSIVE_PROVIDER：被动的接收更新的地理位置信息，而不用自己主动请求地理位置。意思就是共享手机上其他App采集的位置信息，而不是自己主动去采集。

下图是3种Provider的特点和区别：

![3种Provider的特点](http://ww3.sinaimg.cn/large/65e4f1e6gw1f7f3p1sx4aj20kp0g5acw.jpg)

## 2.打开手机的设置

先看下原生系统中地理位置设置的界面截图：

![地理位置设置截图](http://ww1.sinaimg.cn/large/65e4f1e6gw1f7f47p124kj20rs0m376v.jpg)

以原生系统为例，需要采集地理位置时，需要：

- 打开通知栏的GPS开关
- 进入`设置->位置信息->模式`，打开开关。然后我们可以看到，这里有3类模式：
    - 高精确度：使用GPS、WLAN、蓝牙或者移动网络确定位置
    - 节电：使用WLAN、蓝牙或者移动网络确定位置
    - 仅限设备：使用GPS确定位置

**PS：**我发现小米手机上，即使你把通知栏里面地理位置开关关闭了，进入系统的设置界面，还是可以看到地理位置是开启的，默认选择的是`节电`模式。而原生系统你只要在通知栏关闭了开关，就无法使用定位服务了。这里感觉国内厂商在细节上可能会有一些不同的实现。

## 3.给你的App注册权限

当你在代码里面使用3种不同的Provider时，应该关注到两个权限：

- LocationManager.GPS_PROVIDER：android.permission.ACCESS_FINE_LOCATION
- LocationManager.NETWORK_PROVIDER：android.permission.ACCESS_COARSE_LOCATION 或者 android.permission.ACCESS_FINE_LOCATION。
    - 当声明ACCESS_FINE_LOCATION时，拿到的位置信息将更精确（几十米到几百米）
    - 当声明ACCESS_COARSE_LOCATION时，拿到的位置会粗略一点（几百米到几千米）
- LocationManager.PASSIVE_PROVIDER：android.permission.ACCESS_COARSE_LOCATION 

> 注意：如果声明了ACCESS_FINE_LOCATION时，就不用再声明ACCESS_COARSE_LOCATION了，因为ACCESS_FINE_LOCATION已经包含了使用NETWORK_PROVIDER的能力。此外从Android6.0开始，ACCESS_FINE_LOCATION和ACCESS_COARSE_LOCATION已经是***dangerous permission***，开发者需要注意这一点，当用户在运行你的App时，如果没有授权，仍然是无法获取到地理位置信息的。


## 4.根据需求的场景写代码（**记住要尽量省电**）

**一定要省电：**这是一个非常重要的用户体验，我们应该对自己做的App负责。什么时候开始使用地理位置服务，什么时候停止使用，我们一定要想清楚，尽量不要一直占用着这种高耗电的资源。

### 4.1基本代码

下面看代码，一段基本的获取地理位置的代码是这么写的，这段代码可以让你通过异步的方式获取到用户的地理位置。

```
// 获得Location Manager的实例
LocationManager locationManager = (LocationManager) this.getSystemService(Context.LOCATION_SERVICE);

// 定义一个监听器，实现onLocationChanged方法，在这个方法里面可以拿到更新后的地理位置
LocationListener locationListener = new LocationListener() {
    public void onLocationChanged(Location location) {
      // 新的Location值在这里返回，Location实例中包含着纬度、经度、海拔、精确度、更新时间等一系列信息。
      makeUseOfNewLocation(location);
    }

    public void onStatusChanged(String provider, int status, Bundle extras) {}

    public void onProviderEnabled(String provider) {}

    public void onProviderDisabled(String provider) {}
  };

// 注册监听器，当地理位置变化时，发出通知给Listener。这个方法很关键。4个参数需要了解清楚：
// 第1个参数：你所使用的provider名称，是个String
// 第2个参数minTime：地理位置更新时发出通知的最小时间间隔
// 第3个参数minDistance：地理位置更新发出通知的最小距离，第2和第3个参数的作用关系是“或”的关系，也就是满足任意一个条件都会发出通知。这里第2、3个参数都是0，意味着任何时间，只要位置有变化就会发出通知。
// 第4个参数：你的监听器
locationManager.requestLocationUpdates(LocationManager.NETWORK_PROVIDER, 0, 0, locationListener);
```


### 4.2如何优化

但是实战中一定要尽量去优化，虽然获取地理位置只能是异步的，但是仍然不建议一直不停地监听地理位置的变化。

谷歌官方也给出了一个采集地理位置的思路，非常值得我们来参考。思路的基本步骤如下：

- 启动应用。
- 当用户进入到应用中需要使用地理位置场景时，选择一个合适的Provider，开始监听地理位置的变化。
- 获取系统中缓存的上次的地理位置`LastKnownLocation`，保存到当前地理位置变量`currentLocation`中作为备选值，当拿到新的地理位置后，对比两者，选择最优的那个继续保存它。
- 停止监听地理位置的变化。
- 使用当前维护着的这个Location作为用户的位置。

谷歌还给出了这个方案的一个timeline图示。

![A timeline representing the window in which an application listens for location updates](http://ww1.sinaimg.cn/large/65e4f1e6gw1f7f4svf6zsj20mj064ab0.jpg)


### 4.3关键问题

我们比较关注下面4点：

- 1.如何选择一个最好的provider？
- 2.什么时候开始监听地理位置变化，什么时候结束？
- 3.如何比较两个地理位置，决定哪个更好？
- 4.LastknownPostion怎么获取，怎么使用？


下面介绍我的想法：

#### 第1点：如何选择一个最好的provider？

这需要看你的需求。系统中也提供了一些方法来帮我们选择，可以设定一个条件`Criteria`，指定帅选最符合条件的地理位置提供者，根据Cirteria指定的条件，设备会自动选择哪种location provider。

代码如下：

```
Criteria criteria = new Criteria();//
criteria.setAccuracy(Criteria.ACCURACY_FINE);//设置定位精准度
criteria.setAltitudeRequired(false);//是否要求海拔
criteria.setBearingRequired(true);//是否要求方向
criteria.setCostAllowed(true);//是否要求收费
criteria.setSpeedRequired(true);//是否要求速度
criteria.setPowerRequirement(Criteria.POWER_LOW);//设置相对省电
criteria.setBearingAccuracy(Criteria.ACCURACY_HIGH);//设置方向精确度
criteria.setSpeedAccuracy(Criteria.ACCURACY_HIGH);//设置速度精确度
criteria.setHorizontalAccuracy(Criteria.ACCURACY_HIGH);//设置水平方向精确度
criteria.setVerticalAccuracy(Criteria.ACCURACY_HIGH);//设置垂直方向精确度

// 返回满足条件的，当前设备可用的location provider
// 当第2个参数为false时，返回当前设备所有provider中最符合条件的那个（但是不一定可用）。
// 当第2个参数为true时，返回当前设备所有可用的provider中最符合条件的那个。
String rovider  = mLocationManager.getBestProvider(criteria,true);
```


总之，一共就3个provider，其实对于大部分开发者，选来选去就是`gps` or `network`。


#### 第2点，什么时候开始，什么时候结束？

我认为最好开启了监听器后，要尽可能早地结束它。也就是不要调用了`requestLocationUpdates(provider, minTime, minDistance, listener)`让位置服务开始工作后，很长时间都不去`removeUpdates(listener)`来停止服务。

虽然在`requestLocationUpdates`方法中，有**minTime**、**minDistance**参数可以设置。比如设置了60000ms的minTime，希望采更新完一次地理位置后休息60s。或者设置2000米的minDistance，希望位置变化不超过2公里，也休息。这样做**看起来好像**是可以省电。

但是实测中发现，如果调用`requestLocationUpdates(LocationManager.GPS_PROVIDER, 60000, 2000, listener)`注册监听器后，系统的状态栏上面的GPS那个小图标一直在显示。只要你不`removeUpdates(listener)`，他就一直在工作。其实我理解，即使你设置了minTime和minDistance，位置服务还是一直处于工作状态的，不然它怎么知道位置变化超过了你设定的minDistance呢？

![系统栏中的GPS图标](http://ww1.sinaimg.cn/large/65e4f1e6gw1f7f94tb263j20eu022t8r.jpg)

所以我的建议是，当你调用**requestLocationUpdates**后，还应该是设置一个定时器，比如30s。当30s时间到了之后，就**removeUpdate**，不再监听地理位置了，转而使用备选的LastKnownLocation。当下次需要使用地理位置时，再重新注册监听器，监听30s，然后就移除监听器。如果对实时性要求高，我们可以在用户进入App中某个需要定位服务的场景之前，采用这个方法获取一次地理位置，把它保存下来。

#### 第3点，如何比较两个Location，选出更好的那个？

谷歌也给出了代码示例，先看一下。

```
private static final int TWO_MINUTES = 1000 * 60 * 2;

/** Determines whether one Location reading is better than the current Location fix
  * @param location  The new Location that you want to evaluate
  * @param currentBestLocation  The current Location fix, to which you want to compare the new one
  */
protected boolean isBetterLocation(Location location, Location currentBestLocation) {
    if (currentBestLocation == null) {
        // A new location is always better than no location
        return true;
    }

    // Check whether the new location fix is newer or older
    long timeDelta = location.getTime() - currentBestLocation.getTime();
    boolean isSignificantlyNewer = timeDelta > TWO_MINUTES;
    boolean isSignificantlyOlder = timeDelta < -TWO_MINUTES;
    boolean isNewer = timeDelta > 0;

    // If it's been more than two minutes since the current location, use the new location
    // because the user has likely moved
    if (isSignificantlyNewer) {
        return true;
    // If the new location is more than two minutes older, it must be worse
    } else if (isSignificantlyOlder) {
        return false;
    }

    // Check whether the new location fix is more or less accurate
    int accuracyDelta = (int) (location.getAccuracy() - currentBestLocation.getAccuracy());
    boolean isLessAccurate = accuracyDelta > 0;
    boolean isMoreAccurate = accuracyDelta < 0;
    boolean isSignificantlyLessAccurate = accuracyDelta > 200;

    // Check if the old and new location are from the same provider
    boolean isFromSameProvider = isSameProvider(location.getProvider(),
            currentBestLocation.getProvider());

    // Determine location quality using a combination of timeliness and accuracy
    if (isMoreAccurate) {
        return true;
    } else if (isNewer && !isLessAccurate) {
        return true;
    } else if (isNewer && !isSignificantlyLessAccurate && isFromSameProvider) {
        return true;
    }
    return false;
}

/** Checks whether two providers are the same */
private boolean isSameProvider(String provider1, String provider2) {
    if (provider1 == null) {
      return provider2 == null;
    }
    return provider1.equals(provider2);
}

```

这段代码的策略是：

- 1.先看更新时间：设定一个时间范围，2分钟。
    - 如果新的Location比旧的Location获取时间更新，且超过2分钟，那么认为新的Location更好。
    - 如果新的Location比旧的Location获取时间更老，且超过2分钟，那么认为新的Location不够好。
    - 如果新的Location比旧的Location获取时间更新，但没有超过2分钟，那么看下它们的精确度。

- 2.再看精确度：设定一个精确度范围，200米。
    - 如果新的Location比旧的Location精确度更高，那么认为新的Location更好。
    - 如果新的Location和旧的Location精确度相等，且获取时间更新，那么认为新的Location更好。
    - 如果新的Location比旧的Location精确度低200m以内，且获取时间更新，来自同一个provider，那么为认为新的Location更好。
    - 其他情况都认为旧的Location更好。

这段代码是一个参考，我们实际开发中可以更具需要去定义自己的**Better Location**策略。

另外，从API>=17开始，Location类还增加了一个`getElapsedRealtimeNanos`方法（获取从系统启动后走过的时间），这是为了解决`getTime`方法（获取UTC时间）不够精确，容易产生误差的问题。这个方法在比较两个Location时将更加可靠。

#### 第4点，怎么获取LastknownPostion，怎么使用？

相信有了第3点，应该知道怎么选择**Better Location**。至于获取LastKnownLocation直接看代码。

```
Location gpsLocation = null;
Location networkLocation = null;

if (context.checkCallingOrSelfPermission(Manifest.permission.ACCESS_FINE_LOCATION) == PackageManager.PERMISSION_GRANTED) {
    gpsLocation = locationManager.getLastKnownLocation(LocationManager.GPS_PROVIDER);
}

if (context.checkCallingOrSelfPermission(Manifest.permission.ACCESS_COARSE_LOCATION) == PackageManager.PERMISSION_GRANTED) {
    networkLocation = locationManager.getLastKnownLocation(LocationManager.NETWORK_PROVIDER);
}

// 下面可以比较一下哪个更好...
Location currentLocation = gpsLocation;
if (isBetterLocation(currentLocation, networkLocation)){
    currentLocation = networkLocation;
}
```

# 总结一下

说了一大堆，我觉得平时开发的时候应该这么做：

- 1.确定自己的应用什么时候要开始监听地理位置变化，什么时候停止。
- 2.选择一个合适的provider，开始监听它提供的地理位置变化。
- 3.读取系统中GPS和NETWORK这两个Provide缓存的**LastKnownPostion**，选出Better Location保存到currentBestLocation变量中。
- 4.监听到地理位置更新后，把更新到的Location和保存的currentBestLocation比较，得出Better One，再保存到currentBestLocation变量中。
- 5.使用currentBestLocation作为用户的位置，并在合适时机移除监听器。







