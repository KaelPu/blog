---
title: Android学习之路 -- 四大组件(BroadcastReceiver)
date: 2017-07-03 01:14:43
tags: Android学习之路
---

### 四大组件的工作过程
#### BroadcastReceiver的工作过程
简单回顾一下广播的使用方法, 首先定义广播接收者, 只需要继承BroadcastReceiver并重写onReceive()方法即可. 定义好了广播接收者, 还需要注册广播接收者, 分为两种静态注册或者动态注册. 注册完成之后就可以发送广播了.
###9.4.1 广播的注册过程
![image.png](https://upload-images.jianshu.io/upload_images/1967257-2cf63d9d7d51173d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1. 动态注册的过程是从ContextWrapper#registerReceiver()开始的. 和Activity或者Service一样. ContextWrapper并没有做实际的工作, 而是将注册的过程直接交给了ContextImpl来完成.
2. ContextImpl#registerReceiver()方法调用了本类的registerReceiverInternal()方法.
3. 系统首先从mPackageInfo获取到IIntentReceiver对象, 然后再采用跨进程的方式向AMS发送广播注册的请求. 之所以采用IIntentReceiver而不是直接采用BroadcastReceiver, 这是因为上述注册过程中是一个进程间通信的过程. 而BroadcastReceiver作为Android中的一个组件是不能直接跨进程传递的. 所有需要通过IIntentReceiver来中转一下.
4. IIntentReceiver作为一个Binder接口, 它的具体实现是LoadedApk.ReceiverDispatcher.InnerReceiver, ReceiverDispatcher的内部同时保存了BroadcastReceiver和InnerReceiver, 这样当接收到广播的时候, ReceiverDispatcher可以很方便的调用BroadcastReceiver#onReceive()方法. 这里和Service很像有同样的类, 并且内部类中同样也是一个Binder接口.
5. 由于注册广播真正实现过程是在AMS中, 因此跟进AMS中, 首先看registerReceiver()方法, 这里只关心里面的核心部分. 这段代码最终会把远程的InnerReceiver对象以及IntentFilter对象存储起来, 这样整个广播的注册就完成了.

##### 广播的发送和接收过程
![image.png](https://upload-images.jianshu.io/upload_images/1967257-00d1a61663661a21.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
广播的发送有几种：普通广播、有序广播和粘性广播，他们的发送/接收流程是类似的，因此
只分析普通广播的实现。
1. 广播的发送和接收, 本质就是一个过程的两个阶段. 广播的发送仍然开始于ContextImpl#sendBroadcase()方法, 之所以不是Context, 那是因为Context#sendBroad()是一个抽象方法. 和广播的注册过程一样, ContextWrapper#sendBroadcast()仍然什么都不做, 只是把事情交给了ContextImpl去处理.
2. ContextImpl里面也几乎什么都没有做, 内部直接向AMS发起了一个异步请求用于发送广播.
3. 调用AMS#broadcastIntent()方法，继续调用broadcastIntentLocked()方法。
4. 在broadcastIntentLocked()内部, 会根据intent-filter查找出匹配的广播接收者并经过一系列的条件过滤. 最终会将满足条件的广播接收者添加到BroadcastQueue中, 接着BroadcastQueue就会将广播发送给相应广播接收者.
5. BroadcastQueue#scheduleBroadcastsLocked()方法内并没有立即发送广播, 而是发送了一个BROADCAST_INTENT_MSG类型的消息, BroadcastQueue收到消息后会调用processNextBroadcast()方法。
6. 无序广播存储在mParallelBroadcasts中, 系统会遍历这个集合并将其中的广播发送给他们所有的接收者, 具体的发送过程是通过deliverToRegisteredReceiverLocked()方法实现. deliverToRegisteredReceiverLocked()负责将一个广播发送给一个特定的接收者, 它的内部调用了performReceiverLocked方法来完成具体发送过程.
7. performReceiverLocked()方法调用的ApplicationThread#scheduleRegisteredReceiver()实现比较简单, 它通过InnerReceiver来实现广播的接收
8. scheduleRegisteredReceiver()方法中，receiver.performReceive()中的receiver对应着IIntentReceiver类型的接口. 而具体的实现就是ReceiverDispatcher$InnerReceiver. 这两个嵌套的内部类是所属在LoadedApk中的。
9. 又调用了LoadedApk$ReceiverDispatcher#performReceive()的方法.在performReceiver()这个方法中, 会创建一个Args对象并通过mActivityThread的post方法执行args中的逻辑. 而这些类的本质关系就是:
	- Args: 实现类Runnable
	- mActivityThread: 是一个Handler, 就是ActivityThread中的mH. mH就是ActivityThread$H. 这个内部类H以前说过.
10. 实现Runnable接口的Args中BroadcastReceiver#onReceive()方法被执行了, 也就是说应用已经接收到了广播, 同时onReceive()方法是在广播接收者的主线程中被调用的.

android 3.1开始就增添了两个标记为. 分别是FLAG_INCLUDE_STOPPED_PACKAGES, FLAG_EXCLUDE_STOPPED_PACKAGES. 用来控制广播是否要对处于停止的应用起作用.
- FLAG_INCLUDE_STOPPED_PACKAGES: 包含停止应用, 广播会发送给已停止的应用.
- FLAG_EXCLUDE_STOPPED_PACKAGES: 不包含已停止应用, 广播不会发送给已停止的应用

在android 3.1开始, 系统就为所有广播默认添加了FLAG_EXCLUDE_STOPPED_PACKAGES标识。 当这两个标记共存的时候以FLAG_INCLUDE_STOPPED_PACKAGES(非默认项为主).

应用处于停止分为两种
- 应用安装后未运行
- 被手动或者其他应用强停

开机广播同样受到了这个标志位的影响. 从Android 3.1开始处于停止状态的应用同样无法接受到开机广播, 而在android 3.1之前处于停止的状态也是可以接收到开机广播的.

