---
title: Android学习之路 -- IPC理解（一）
date: 2017-03-14 22:11:15
tags: Android学习之路
---

### Android IPC
#### Android IPC 简介
- IPC即Inter-Process Communication，含义为进程间通信或者跨进程通信，是指两个进程之间进行数据交换的过程。
- 线程是CPU调度的最小单元，是一种有限的系统资源。进程一般指一个执行单元，在PC和移动设备上是指一个程序或者应用。进程与线程是包含与被包含的关系。一个进程可以包含多个线程。最简单的情况下一个进程只有一个线程，即主线程（ 例如Android的UI线程） 。
- 任何操作系统都需要有相应的IPC机制。如Windows上的剪贴板、管道和邮槽；Linux上命名管道、共享内容、信号量等。Android中最有特色的进程间通信方式就是binder，另外还支持socket。contentProvider是Android底层实现的进程间通信。
- 在Android中，IPC的使用场景大概有以下：
	- 有些模块由于特殊原因需要运行在单独的进程中。
	- 通过多进程来获取多份内存空间。
	- 当前应用需要向其他应用获取数据。

#### Android中的多进程模式

##### 开启多进程模式
在Android中使用多线程只有一种方法：给四大组件在Manifest中指定 android:process 属性。这个属性的值就是进程名。这意味着不能在运行时指定一个线程所在的进程。

tips：使用 adb shell ps 或 adb shell ps|grep 包名 查看当前所存在的进程信息。

两种进程命名方式的区别
1. “：remote”
“：”的含义是指在当前的进程名前面附加上当前的包名，完整的进程名为“com.example.c2.remote"。这种进程属于当前应用的私有进程，其他应用的组件不可以和它跑在同一个进程中。
2. "com.example.c2.remote"
这是一种完整的命名方式。这种进程属于全局进程，其他应用可以通过ShareUID方式和它跑在同一个进程中。


##### 多线程模式的运行机制
Android为每个进程都分配了一个独立的虚拟机，不同虚拟机在内存分配上有不同的地址空间，导致不同的虚拟机访问同一个类的对象会产生多份副本。例如不同进程的Activity对静态变量的修改，对其他进程不会造成任何影响。所有运行在不同进程的四大组件，只要它们之间需要通过内存在共享数据，都会共享失败。四大组件之间不可能不通过中间层来共享数据。

多进程会带来以下问题：
1. 静态成员和单例模式完全失效。
2. 线程同步锁机制完全失效。
这两点都是因为不同进程不在同一个内存空间下，锁的对象也不是同一个对象。
3. SharedPreferences的可靠性下降。
SharedPreferences底层是 通过读/写XML文件实现的，并发读/写会导致一定几率的数据丢失。
4. Application会多次创建。
由于系统创建新的进程的同时分配独立虚拟机，其实这就是启动一个应用的过程。在多进程模式中，不同进程的组件拥有独立的虚拟机、Application以及内存空间。

多进程相当于两个不同的应用采用了SharedUID的模式

实现跨进程的方式有很多：
1. Intent传递数据。
2. 共享文件和SharedPreferences。
3. 基于Binder的Messenger和AIDL。
4. Socket等

#### IPC基础概念介绍
主要介绍 Serializable 、 Parcelable 、 Binder 。Serializable和Parcelable接口可以完成对象的序列化过程，我们通过Intent和Binder传输数据时就需要Parcelabel和Serializable。还有的时候我们需要对象持久化到存储设备上或者通过网络传输到其他客户端，也需要Serializable完成对象持久化。

##### Serializable接口
Serializable 是Java提供的一个序列化接口（ 空接口） ，为对象提供标准的序列化和反序列化操作。只需要一个类去实现 Serializable 接口并声明一个 serialVersionUID 即可实现序列化。

	private static final long serialVersionUID = 8711368828010083044L

serialVersionUID也可以不声明。如果不手动指定 serialVersionUID 的值，反序列化时如果当前类有所改变（ 比如增删了某些成员变量） ，那么系统就会重新计算当前类的hash值并更新 serialVersionUID 。这个时候当前类的 serialVersionUID 就和序列化数据中的serialVersionUID 不一致，导致反序列化失败，程序就出现crash。

静态成员变量属于类不属于对象，不参与序列化过程，其次 transient 关键字标记的成员变量也不参与序列化过程。

通过重写writeObject和readObject方法可以改变系统默认的序列化过程。

##### Parcelable接口
Parcel内部包装了可序列化的数据，可以在Binder中自由传输。序列化过程中需要实现的功能有序列化、反序列化和内容描述。

序列化功能由 writeToParcel 方法完成,最终是通过 Parcel 的一系列writer方法来完成。
   
       @Override
        public void writeToParcel(Parcel out, int flags) {
        out.writeInt(code);
        out.writeString(name);
        }
反序列化功能由 CREATOR 来完成，其内部表明了如何创建序列化对象和数组，通过 Parcel 的一系列read方法来完成。

    public static final Creator<Book> CREATOR = new Creator<Book>() {
    @Override
    public Book createFromParcel(Parcel in) {
    return new Book(in);
    } 
	@Override
    public Book[] newArray(int size) {
    return new Book[size];
    }
    };
    protected Book(Parcel in) {
    code = in.readInt();
    name = in.readString();
    }

在Book(Parcel in)方法中，如果有一个成员变量是另一个可序列化对象，在反序列化过程中需要传递当前线程的上下文类加载器，否则会报无法找到类的错误。
       
       book = in.readParcelable(Thread.currentThread().getContextClassLoader());
内容描述功能由 describeContents 方法完成，几乎所有情况下都应该返回0，仅当当前对象中存在文件描述符时返回1。

    public int describeContents() {
    return 0;
    }
Serializable 是Java的序列化接口，使用简单但开销大，序列化和反序列化过程需要大量I/O操作。而 Parcelable 是Android中的序列化方式，适合在Android平台使用，效率高但是使用麻烦。 Parcelable 主要在内存序列化上，Parcelable 也可以将对象序列化到存储设备中或者将对象序列化后通过网络传输，但是稍显复杂，推荐使用 Serializable 。

##### Binder
Binder是Android中的一个类，实现了 IBinder 接口。从IPC角度说，Binder是Andoird的一种跨进程通讯方式，Binder还可以理解为一种虚拟物理设备，它的设备驱动是/dev/binder。从Android Framework角度来说，Binder是 ServiceManager 连接各种Manager( ActivityManager· 、 WindowManager )和相应 ManagerService 的桥梁。从Android应用层来说，Binder是客户端和服务端进行通信的媒介，当bindService时，服务端返回一个包含服务端业务调用的Binder对象，通过这个Binder对象，客户端就可以获取服务器端提供的服务或者数据（ 包括普通服务和基于AIDL的服务）。

![](http://gityuan.com/images/binder/prepare/IPC-Binder.jpg)
> Binder通信采用C/S架构，从组件视角来说，包含Client、Server、ServiceManager以及binder驱动，其中ServiceManager用于管理系统中的各种服务。

>图中的Client,Server,Service Manager之间交互都是虚线表示，是由于它们彼此之间不是直接交互的，而是都通过与Binder驱动进行交互的，从而实现IPC通信方式。其中Binder驱动位于内核空间，Client,Server,Service Manager位于用户空间。Binder驱动和Service Manager可以看做是Android平台的基础架构，而Client和Server是Android的应用层，开发人员只需自定义实现client、Server端，借助Android的基本平台架构便可以直接进行IPC通信。
>http://gityuan.com/2015/10/31/binder-prepare/

Android中Binder主要用于 Service ，包括AIDL和Messenger。普通Service的Binder不涉及进程间通信，Messenger的底层其实是AIDL，所以下面通过AIDL分析Binder的工作机制。

**由系统根据AIDL文件自动生成.java文件**
1. Book.java
表示图书信息的实体类，实现了Parcelable接口。
2. Book.aidl
Book类在AIDL中的声明。
3. IBookManager.aidl
定义的管理Book实体的一个接口，包含 getBookList 和 addBook 两个方法。尽管Book类和IBookManager位于相同的包中，但是在IBookManager仍然要导入Book类。
4. IBookManager.java
系统为IBookManager.aidl生产的Binder类，在 gen 目录下。
IBookManager继承了 IInterface 接口，所有在Binder中传输的接口都需要继IInterface接口。结构如下：
	- 声明了 getBookList 和 addBook 方法，还声明了两个整型id分别标识这两个方法，用于标识在 transact 过程中客户端请求的到底是哪个方法。
	- 声明了一个内部类 Stub ，这个 Stub 就是一个Binder类，当客户端和服务端位于同一进程时，方法调用不会走跨进程的 transact 。当二者位于不同进程时，方法调用需要走 transact 过程，这个逻辑有 Stub 的内部代理类 Proxy 来完成。
    - 这个接口的核心实现就是它的内部类 Stub 和 Stub 的内部代理类 Proxy 。

**Stub和Proxy类的内部方法和定义**
![](http://upload-images.jianshu.io/upload_images/1944615-3c92d9d160957e78.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1. DESCRIPTOR
Binder的唯一标识，一般用Binder的类名表示。
2. asInterface(android.os.IBinder obj)
将服务端的Binder对象转换为客户端所需的AIDL接口类型的对象，如果C/S位于同一进
程，此方法返回就是服务端的Stub对象本身，否则返回的就是系统封装后的Stub.proxy对
象。
3. asBinder
返回当前Binder对象。
4. onTransact
这个方法运行在服务端的Binder线程池中，由客户端发起跨进程请求时，远程请求会通过
系统底层封装后交由此方法来处理。该方法的原型是
        java public Boolean onTransact(int code,Parcelable data,Parcelable reply,int flags)
	1. 服务端通过code确定客户端请求的目标方法是什么，
	2. 接着从data取出目标方法所需的参数，然后执行目标方法。
	3. 执行完毕后向reply写入返回值（ 如果有返回值） 。
	4. 如果这个方法返回值为false，那么服务端的请求会失败，利用这个特性我们可以来做权限验证。
5. Proxy#getBookList 和Proxy#addBook
这两个方法运行在客户端，内部实现过程如下：
	1. 首先创建该方法所需要的输入型对象Parcel对象_data，输出型Parcel对象_reply和返回值对象List。
	2. 然后把该方法的参数信息写入_data（ 如果有参数）
	3. 接着调用transact方法发起RPC（ 远程过程调用） ，同时当前线程挂起
	4. 然后服务端的onTransact方法会被调用知道RPC过程返回后，当前线程继续执行，并从_reply中取出RPC过程的返回结果，最后返回_reply中的数据。

AIDL文件不是必须的，之所以提供AIDL文件，是为了方便系统为我们生成IBookManager.java，但我们完全可以自己写一个。

**linkToDeath和unlinkToDeath**
如果服务端进程异常终止，我们到服务端的Binder连接断裂。但是，如果我们不知道Binder连接已经断裂，那么客户端功能会受影响。通过linkTODeath我们可以给Binder设置一个死亡代理，当Binder死亡时，我们就会收到通知。
1. 声明一个 DeathRecipient 对象。 DeathRecipient 是一个接口，只有一个方法 binderDied ，当Binder死亡的时候，系统就会回调 binderDied 方法，然后我们就可以重新绑定远程服务。
        private IBinder.DeathRecipient mDeathRecipient = new IBinder.DeathRecipient(){
        @Override
        public void binderDied(){
        if(mBookManager == null){
        return;
        } 
        mBookManager.asBinder().unlinkToDeath(mDeathRecipient,0);
        mBookManager = null;
        // TODO：这里重新绑定远程Service
        }
        }
2. 在客户端绑定远程服务成功后，给binder设置死亡代理：
        mService = IBookManager.Stub.asInterface(binder);
        binder.linkToDeath(mDeathRecipient,0);
3. 另外，可以通过Binder的 isBinderAlive 判断Binder是否死亡。