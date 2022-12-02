---
title: Activity的生命周期和启动模式
date: 2016-10-11
tags: Android
---
-------

# 一 、Activity的生命周期
## 1.典型情况下的生命周期分析如下：
![Activity生命周期](http://autism-tl.cn/picture/activity%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F.png )
`onCreate()` Activity正在被创建，可以做些初始化工作。
<!-- more -->

`onRestart()` Activity正在重新启动，一般，当当前Activity从不可见重新可见时调用。

`onStart()` Activity正在启动，Activity可见但是还没出现在前台，还无法与用户交互。

`onResume()` Activity可见，并且出现在前台开始活动。


`onPause` 表示Activity正在停止，正常情况下，紧接着`onStop` 会被调用，但如果在这个时候快速回到原Activity，`onResume()` 将被调用。 此处可以做一些存储数据、停止动画等工作，但不能进行太耗时的操作，不然将会影响新Activity的显示（`onPause()`必须先执行完，新Activity的`onResume()`才会执行）。 

`0nStop()` Activity即将停止，可以做一些稍微重量级的回收工作，同样不能太耗时。

`onDestroy()` Activity即将被销毁，可以进行最终的回收工作与资源释放。

**注意**
+ 新Activity是透明主题时，旧Activity不会走onStop。
+ Activity切换时，旧Activity的onPause会先执行，然后才会启动新的         Activity。
+ `onStart()`和`onStop()`、`onPause()`和`onResume()`的区别：这两个配对分别代表不同的意义，onStart与onStop是从Activity是否可见这个角度调用的，onResume和onPause是从Activity是否显示在前台这个角度来回调的，在实际使用没其他明显区别。

## 2.异常情况下的生命周期分析
除了典型生命周期外，还有一些异常情况，比如当资源相关的系统配置发送改变以及系统内存不足的时候，Activity就可能被杀死。
### 1. 资源相关的系统配置发送改变导致Activity被杀死并重新创建
根据安卓系统的资源加载机制，我们为了兼容不同的设备，一张图片需要准备不同大小精度的图片，系统会根据当前设备的情况去加载适合的Resources资源，这就导致横屏和竖屏会加载到两张不同的图片，如果突然旋转屏幕，系统配置发送了改变，在默认情况下，Activity会被销毁重建，当然我们也可以阻止系统重建我们的Activity。

默认情况下，当系统配置发送改变后，Activity会调用到`onDestroy`，同时由于Activity在异常情况下终止，系统会在`onStop`之前调用`onSaveInstanceState`来保存当前Activity的状态。当Activity被重建后，系统会在`onStart`之后调用`onRestoreInstanceState`,并且把`onSaveInstanceState`方法保存的Bundle对象作为参数传递给onRestoreInstanceState和onCreate方法，用来取出之前的数据并恢复。

系统默认为我们做了一定的恢复工作，比如恢复Activity的视图结构（文本框输入的数据，ListView滚动的位置等，每个View也有`onSaveInstanceState`和`onRestoreInstanceState`,看看不同View的具体实现就能知道系统自动为它们恢复了哪些数据）。保存和恢复View层次结构用的是典型的委托思想，顶层容器一般是DecorView。
### 2. 资源内存不足导致低优先级的Activity被杀死
**优先级**
+ 前台Activity--正在交互的Activity，优先级最高，
+ 可见但非前台Activity
+ 后台Activity--执行了onStop的Activity，优先级最低

当系统内存不足时，会按照上述优先级去杀死目标Activity所在的进程，并在后续通过`onSaveInstanceState`和`onRestoreInstanceState`存储和恢复数据。一些后台工作不适合脱离四大组件而独自运行在后台中，因为一个进程中如果没有四大组件在运行很快会被系统杀死，比较好的方法是将后台工作放入Service中从而保证进程有一定的优先级。


**给Activity指定configChanges属性可以防止系统默认销毁重建（系统会调用Activity的onConfigurationChanged方法，我们能在这个方法做一些自己的特殊处理）。**
# 二、Activity的启动模式
1. standard:标准模式，默认模式。每次启动一个Activity就会创建一个新的实例。（使用ApplicationContext去启动standard模式Activity就会报错。因为sandard模式的Activity会默认进入启动它所属的任务栈，但是由于非Activity类型的Context（如ApplicationContext）没有所谓的任务栈 就出错了。）
2. singleTop：栈顶复用模式。如果新Activity已经位于任务栈的栈顶，就不会重新创建，同时回调它的onNewIntent方法。
3. singleTask：栈内复用模式。 单实例模式，只要Activity在一个任务栈中存在，那么就不会重新创建，系统将回调onNewIntent方法。如果不存在A想要的任务栈就会重新创建一个任务栈 并将A实例化后放入栈中（默认具有clearTop的效果）。
4. singleInstance：单实例模式。 具有此模式的Activity只能单独位于一个任务栈中。

**在Intent中设置标记位同样也能设置启动模式，但会有些许的差别。**
# 三、IntentFilter的匹配规则
```
activity
 android:name="com.gaop.MainActivity"
 android:configChanges="screenLayout"//设置在某些情况下不重新创建Activity
 android:lable="@string/app_name"
 android:launchMode="singleTask"
 android:taskAffinity="com.gaop.task1"//设置任务栈
 <intent-filter>
  <action android:name="com.tl.c"/>
  <action android:name="com.tl.d"/>
  <category android:name="com.tl.category.c"/>
  <category android:name="com.tl.category.d"/>
  <category android:name="android.intent.category.DEFAULT"/>
  <data android:mimeType="text/plain"/>
 </intent-filter>
</activity>
```
1. action匹配规则 要Intent中的action能够与过滤规则中的任何一个相同即可匹配成功，如果没有指定action，匹配失败。
2. category匹配规则 Intent中一旦出现category就必须每个category与过滤规则中的其中一个相同，Intent中没有category也会匹配成功（系统会自动添加一个默认category）。
3. data的匹配规则 data由mimeType（媒体类型，如 image/jpeg ）与URL组成。 规则中只有mimeType时，Intent必须是指定的mimeType。 有多组data规则时，匹配其中一组即可。

**采用隐式方式启动Activity时，可以用PackageManager的resolveActivity方法或者Intent的resolveActivity方法判断是否有Activity匹配我们的隐式Intent。**