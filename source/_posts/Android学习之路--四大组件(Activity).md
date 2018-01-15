---
title: Android学习之路 -- 四大组件(Activity)
date: 2017-06-25 18:54:33
tags: Android学习之路
---

### 四大组件的工作过程
#### Activity的工作过程
![](https://hujiaweibujidao.github.io/images/androidart_activity.png)
1. Activity的所有 startActivity 重载方法最终都会调用 startActivityForResult 。
2. 调用 mInstrumentation.execStartActivity.execStartActivity() 方法。
3. 代码中启动Activity的真正实现是由ActivityManagerNative.getDefault().startActivity()方法完成的. ActivityManagerService简称AMS. AMS继承自ActivityManagerNative(), 而ActivityManagerNative()继承自Binder并实现了IActivityManager这个Binder接口, 因此AMS也是一个Binder, 它是IActivityManager的具体实现.ActivityManagerNative.getDefault()本质是一个IActivityManager类型的Binder对象, 因此具体实现是AMS.
4. 在ActivityManagerNative中, AMS这个Binder对象采用单例模式对外提供, Singleton是一个单例封装类. 第一次调用它的get()方法时会通过create方法来初始化AMS这个Binder对象, 在后续调用中会返回这个对象. 
5. AMS的startActivity()过程
	1. checkStartActivityResult () 方法检查启动Activity的结果（ 包括检查有无在
manifest注册）
	2. Activity启动过程经过两次转移, 最后又转移到了mStackSupervisor.startActivityMayWait()这个方法, 所属类为ActivityStackSupervisor. 在startActivityMayWait()内部又调用了startActivityLocked()这里会返回结果码就是之前checkStartActivityResult()用到的。
	3. 方法最后会调用startActivityUncheckedLocked(), 然后又调用了ActivityStack#resumeTopActivityLocked(). 这个时候启动过程已经从ActivityStackSupervisor转移到了ActivityStack类中.
![](http://img.blog.csdn.net/20160428014427476?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
6. 在最后的 ActivityStackSupervisor. realStartActivityLocked() 中，调用了 app.thread.scheduleLaunchActivity() 方法。 这个app.thread是ApplicationThread 类型，继承于 IApplicationThread 是一个Binder类，内部是各种启动/停止 Service/Activity的接口。
7. 在ApplicationThread中， scheduleLaunchActivity() 用来启动Activity，里面的实现就是发送一个Activity的消息（ 封装成 从ActivityClientRecord 对象） 交给Handler处理。这个Handler有一个简洁的名字 H 。
8. 在H的 handleMessage() 方法里，通过 handleLaunchActivity() 方法完成Activity对象的创建和启动,并且ActivityThread通过handleResumeActivity()方法来调用被启动的onResume()这一生命周期方法。PerformLaunchActivity()主要完成了如下几件事：
	1. 从ActivityClientRecord对象中获取待启动的Activity组件信息
	2. 通过 Instrumentation 的 newActivity 方法使用类加载器创建Activity对象
	3. 通过 LoadedApk 的makeApplication方法尝试创建Application对象，通过类加载器实现（ 如果Application已经创建过了就不会再创建）
	4. 创建 ContextImpl 对象并通过Activity的 attach 方法完成一些重要数据的初始化（ContextImpl是一个很重要的数据结构, 它是Context的具体实现, Context中的大部分逻辑都是由ContentImpl来完成的. ContextImpl是通过Activity的attach()方法来和Activity建立关联的,除此之外, 在attach()中Activity还会完成Window的创建并建立自己和Window的关联, 这样当Window接收到外部输入事件收就可以将事件传递给Activity.）
	5. 通过 mInstrumentation.callActivityOnCreate(activity, r.state) 方法调用Activity的 onCreate 方法