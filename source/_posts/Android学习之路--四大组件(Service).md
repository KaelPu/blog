---
title: Android学习之路 -- 四大组件(Service)
date: 2017-08-10 00:28:21
tags: Android学习之路
---

### 四大组件的工作过程
#### Service的工作过程
1. 启动状态：执行后台计算
2. 绑定状态：用于其他组件与Service交互

两种状态是可以共存的

##### Service的启动过程
![image.png](https://upload-images.jianshu.io/upload_images/1967257-e58180e1c53e5ada.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1. Service的启动从 ContextWrapper 的 startService 开始
2. 在ContextWrapper中，大部分操作通过一个 ContextImpl 对象mBase实现
3. 在ContextImpl中， mBase.startService() 会调用 startServiceCommon 方法，而
startServiceCommon方法又会通过 ActivityManagerNative.getDefault() （ 实际上就是AMS） 这个对象来启动一个服务。
4. AMS会通过一个 ActiveService 对象（ 辅助AMS进行Service管理的类，包括Service的启动,绑定和停止等） mServices来完成启动Service： mServices.startServiceLocked() 。
5. 在mServices.startServiceLocked()最后会调用 startServiceInnerLocked() 方法：将Service的信息包装成一个 ServiceRecord 对象，ServiceRecord一直贯穿着整个Service的启动过程。通过 bringUpServiceLocked() 方法来处理，bringUpServiceLocked()又调用了 realStartServiceLocked() 方法，这才真正地去启动一个Service了。
6. realStartServiceLocked()方法的工作如下：
	1. app.thread.scheduleCreateService() 来创建Service并调用其onCreate()生命周期方法
	2. sendServiceArgsLocked() 调用其他生命周期方法，如onStartCommand()
	3. app.thread对象是 IApplicationThread 类型，实际上就是一个Binder，具体实现是ApplicationThread继承ApplictionThreadNative
7. 具体看Application对Service的启动过程app.thread.scheduleCreateService()：通过 sendMessage(H.CREATE_SERVICE , s) ，这个过程和Activity启动过程类似，同时通过发送消息给Handler H来完成的。
8. H会接受这个CREATE_SERVICE消息并通过ActivityThread的 handleCreateService() 来完成Service的最终启动。
9. handleCreateService()完成了以下工作：
	1. 通过ClassLoader创建Service对象
	2. 创建Service内部的Context对象
	3. 创建Application，并调用其onCreate()（ 只会有一次）
	4. 通过 service.attach() 方法建立Service与context的联系（ 与Activity类似）
	5. 调用service的 onCreate() 生命周期方法，至此，Service已经启动了
	6. 将Service对象存储到ActivityThread的一个ArrayMap中

##### Service的绑定过程
![image.png](https://upload-images.jianshu.io/upload_images/1967257-077ff376f076f959.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
和service的启动过程类似的：
1. Service的绑定是从 ContextWrapper 的 bindService 开始
2. 在ContextWrapper中，交给 ContextImpl 对象 mBase.bindService()
3. 最终会调用ContextImpl的 bindServiceCommon 方法，这个方法完成两件事：
	- 将客户端的ServiceConnection转化成 ServiceDispatcher.InnerConnection 对象。ServiceDispatcher连接ServiceConnection和InnerConnection。这个过程通过 LoadedApk 的 getServiceDispatcher 方法来实现，将客户端的ServiceConnection和ServiceDispatcher的映射关系存在一个ArrayMap中。
	- 通过AMS来完成Service的具体绑定过程 ActivityManagerNative.getDefault().bindService()
4. AMS中，bindService()方法再调用 bindServiceLocked() ，bindServiceLocked()再调用 bringUpServiceLocked() ，bringUpServiceLocked()又会调用 realStartServiceLocked() 。
5. AMS的realStartServiceLocked()会调用 ActiveServices 的requrestServiceBindingLocked() 方法，最终是调用了ServiceRecord对象r的 app.thread.scheduleBindService() 方法。
6. ApplicationThread的一系列以schedule开头的方法，内部都通过Handler H来中转：scheduleBindService()内部也是通过 sendMessage(H.BIND_SERVICE , s)
7. 在H内部接收到BIND_SERVICE这类消息时就交给 ActivityThread 的handleBindService() 方法处理：
	1. 根据Servcie的token取出Service对象
	2. 调用Service的 onBind() 方法，至此，Service就处于绑定状态了。
	3. 这时客户端还不知道已经成功连接Service，需要调用客户端的binder对象来调用客户端的ServiceConnection中的 onServiceConnected() 方法，这个通过 ActivityManagerNative.getDefault().publishService() 进行。ActivityManagerNative.getDefault()就是AMS。
8. AMS的publishService()交给ActivityService对象 mServices 的 publishServiceLocked() 来处理，核心代码就一句话 c.conn.connected(r.name,service) 。对象c的类型是 ConnectionRecord ，c.conn就是ServiceDispatcher.InnerConnection对象，service就是Service的onBind方法返回的Binder对象。
9. c.conn.connected(r.name,service)内部实现是交给了mActivityThread.post(new RunnConnection(name ,service,0)); 实现。ServiceDispatcher的mActivityThread是一个Handler，其实就是ActivityThread中的H。这样一来RunConnection就经由H的post方法从而运行在主线程中，因此客户端ServiceConnection中的方法是在主线程中被回调的。
10. RunConnection的定义如下：
	- 继承Runnable接口， run() 方法的实现也是简单调用了ServiceDispatcher的 doConnected 方法。
	- 由于ServiceDispatcher内部保存了客户端的ServiceConntion对象，可以很方便地调用ServiceConntion对象的 onServiceConnected 方法。
	- 客户端的onServiceConnected方法执行后，Service的绑定过程也就完成了。
	- 根据步骤8、9、10service绑定后通过ServiceDispatcher通知客户端的过程可以说明ServiceDispatcher起着连接ServiceConnection和InnerConnection的作用。 至于Service的停止和解除绑定的过程，系统流程都是类似的。
