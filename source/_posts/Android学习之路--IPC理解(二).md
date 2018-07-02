---
title: Android学习之路 -- IPC理解（二）
date: 2017-03-21 16:32:14
tags: Android学习之路
---

### Android IPC
#### Android中的IPC方式
主要有以下方式：
1. Intent中附加extras
2. 共享文件
3. Binder
4. ContentProvider
5. Socket

##### 使用Bundle
四大组件中的三大组件（ Activity、Service、Receiver） 都支持在Intent中传递 Bundle 数据。
Bundle实现了Parcelable接口，因此可以方便的在不同进程间传输。当我们在一个进程中启动了另一个进程的Activity、Service、Receiver，可以再Bundle中附加我们需要传输给远程进程的消息并通过Intent发送出去。被传输的数据必须能够被序列化。

##### 使用文件共享
我们可以序列化一个对象到文件系统中的同时从另一个进程中恢复这个对象。
1. 通过 ObjectOutputStream / ObjectInputStream 序列化一个对象到文件中，或者在另一个进程从文件中反序列这个对象。注意：反序列化得到的对象只是内容上和序列化之前的对象一样，本质是两个对象。
2. 文件并发读写会导致读出的对象可能不是最新的，并发写的话那就更严重了 。所以文件共享方式适合对数据同步要求不高的进程之间进行通信，并且要妥善处理并发读写问题。
3. SharedPreferences 底层实现采用XML文件来存储键值对。系统对它的读/写有一定的缓存策略，即在内存中会有一份 SharedPreferences 文件的缓存，因此在多进程模式下，系统对它的读/写变得不可靠，面对高并发读/写时 SharedPreferences 有很大几率丢失数据，因此不建议在IPC中使用 SharedPreferences 。

##### 使用Messenger
Messenger可以在不同进程间传递Message对象。是一种轻量级的IPC方案，底层实现是AIDL。它对AIDL进行了封装，使得我们可以更简便的进行IPC。
![image.png](https://upload-images.jianshu.io/upload_images/1967257-86f4be84232b011a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
具体使用时，分为服务端和客户端：
1. 服务端：创建一个Service来处理客户端请求，同时创建一个Handler并通过它来创建一个
Messenger，然后再Service的onBind中返回Messenger对象底层的Binder即可。
        private final Messenger mMessenger = new Messenger (new xxxHandler());
2. 客户端：绑定服务端的Sevice，利用服务端返回的IBinder对象来创建一个Messenger，通过这个Messenger就可以向服务端发送消息了，消息类型是 Message 。如果需要服务端响应，则需要创建一个Handler并通过它来创建一个Messenger（ 和服务端一样） ，并通过 Message 的 replyTo 参数传递给服务端。服务端通过Message的 replyTo 参数就可以回应客户端了。

总而言之，就是客户端和服务端 拿到对方的Messenger来发送 Message 。只不过客户端通过bindService 而服务端通过 message.replyTo 来获得对方的Messenger。

Messenger中有一个 Hanlder 以串行的方式处理队列中的消息。不存在并发执行，因此我们不用考虑线程同步的问题。

##### 使用AIDL
如果有大量的并发请求，使用Messenger就不太适合，同时如果需要跨进程调用服务端的方法，Messenger就无法做到了。这时我们可以使用AIDL。

流程如下：
1. 服务端需要创建Service来监听客户端请求，然后创建一个AIDL文件，将暴露给客户端的接口在AIDL文件中声明，最后在Service中实现这个AIDL接口即可。
2. 客户端首先绑定服务端的Service，绑定成功后，将服务端返回的Binder对象转成AIDL接口所属的类型，接着就可以调用AIDL中的方法了。


**AIDL支持的数据类型：**
1. 基本数据类型、String、CharSequence
2. List：只支持ArrayList，里面的每个元素必须被AIDL支持
3. Map：只支持HashMap，里面的每个元素必须被AIDL支持
4. Parcelable
5. 所有的AIDL接口本身也可以在AIDL文件中使用

自定义的Parcelable对象和AIDL对象，不管它们与当前的AIDL文件是否位于同一个包，都必须显式import进来。

如果AIDL文件中使用了自定义的Parcelable对象，就必须新建一个和它同名的AIDL文件，并在其中声明它为Parcelable类型。

        package com.ryg.chapter_2.aidl;
        parcelable Book;
AIDL接口中的参数除了基本类型以外都必须表明方向in/out。AIDL接口文件中只支持方法，不支持声明静态常量。建议把所有和AIDL相关的类和文件放在同一个包中，方便管理。
        
        void addBook(in Book book);
 AIDL方法是在服务端的Binder线程池中执行的，因此当多个客户端同时连接时，管理数据的集合直接采用 CopyOnWriteArrayList 来进行自动线程同步。类似的还有 ConcurrentHashMap 。

因为客户端的listener和服务端的listener不是同一个对象，所以 RecmoteCallbackList 是系统专门提供用于删除跨进程listener的接口，支持管理任意的AIDL接口，因为所有AIDL接口都继承自 IInterface 接口。

        public class RemoteCallbackList<E extends IInterface>
它内部通过一个Map接口来保存所有的AIDL回调，这个Map的key是 IBinder 类型，value是 Callback 类型。当客户端解除注册时，遍历服务端所有listener，找到和客户端listener具有相同Binder对象的服务端listenr并把它删掉。

==客户端RPC的时候线程会被挂起，由于被调用的方法运行在服务端的Binder线程池中，可能很耗时，不能在主线程中去调用服务端的方法。==

**权限验证**
默认情况下，我们的远程服务任何人都可以连接，我们必须加入权限验证功能，权限验证失败则无法调用服务中的方法。通常有两种验证方法：
1. 在onBind中验证，验证不通过返回null
验证方式比如permission验证，在AndroidManifest声明：
		<permission
        android:name="com.rgy.chapter_2.permisson.ACCESS_BOOK_SERVICE"
        android:protectionLevel="normal"/>
		public IBinder onBind(Intent intent){
        int check = checkCallingOrSelefPermission("com.ryq.chapter_2.permission.ACCESS_BOOK_SERVICE");
        if(check == PackageManager.PERMISSION_DENIED){
        	return null;
        }
        return mBinder;
        }
这种方法也适用于Messager。
2. 在onTransact中验证，验证不通过返回false
可以permission验证，还可以采用Uid和Pid验证。

##### 使用ContentProvider
==ContentProvider是四大组件之一，天生就是用来进程间通信。和Messenger一样，其底层实现是用Binder。==

系统预置了许多ContentProvider，比如通讯录、日程表等。要RPC访问这些信息，只需要通过ContentResolver的query、update、insert和delete方法即可。

创建自定义的ContentProvider，只需继承ContentProvider类并实现 onCreate 、 query 、 update 、 insert 、 getType 六个抽象方法即可。getType用来返回一个Uri请求所对应的MIME类型，剩下四个方法对应于CRUD操作。这六个方法都运行在ContentProvider进程中，除了 onCreate 由系统回调并运行在主线程里，其他五个方法都由外界调用并运行在Binder线程池中。

ContentProvider是通过Uri来区分外界要访问的数据集合，例如外界访问ContentProvider中的表，我们需要为它们定义单独的Uri和Uri_Code。根据Uri_Code，我们就知道要访问哪个表了。

==query、update、insert、delete四大方法存在多线程并发访问，因此方法内部要做好线程同步。==若采用SQLite并且只有一个SQLiteDatabase，SQLiteDatabase内部已经做了同步处理。若是多个SQLiteDatabase或是采用List作为底层数据集，就必须做线程同步。

##### 使用Socket
Socket也称为“套接字”，分为流式套接字和用户数据报套接字两种，分别对应于TCP和UDP协议。Socket可以实现计算机网络中的两个进程间的通信，当然也可以在本地实现进程间的通信。我们以一个跨进程的聊天程序来演示。

在远程Service建立一个TCP服务，然后在主界面中连接TCP服务。服务端Service监听本地端口，客户端连接指定的端口，建立连接成功后，拿到 Socket 对象就可以向服务端发送消息或者接受服务端发送的消息。


除了采用TCP套接字，也可以用UDP套接字。实际上socket不仅能实现进程间的通信，还可以实现设备间的通信（只要设备之间的IP地址互相可见）。

#### Binder连接池
前面提到AIDL的流程是：首先创建一个service和AIDL接口，接着创建一个类继承自AIDL接口中的Stub类并实现Stub中的抽象方法，客户端在Service的onBind方法中拿到这个类的对象，然后绑定这个service，建立连接后就可以通过这个Stub对象进行RPC。

那么如果项目庞大，有多个业务模块都需要使用AIDL进行IPC，随着AIDL数量的增加，我们不能无限制地增加Service，我们需要把所有AIDL放在同一个Service中去管理。
![](http://upload-images.jianshu.io/upload_images/667368-b564d4bdd7af3141?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 服务端只有一个Service，把所有AIDL放在一个Service中，不同业务模块之间不能有耦合
- 服务端提供一个 queryBinder 接口，这个接口能够根据业务模块的特征来返回响应的Binder对象给客户端
- 不同的业务模块拿到所需的Binder对象就可以进行RPC了


#### 选用合适的IPC方式
![image.png](https://upload-images.jianshu.io/upload_images/1967257-db24e1ae767a080c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)