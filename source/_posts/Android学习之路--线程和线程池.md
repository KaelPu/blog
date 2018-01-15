---
title: Android学习之路 -- 线程和线程池
date: 2017-08-27 00:24:33
tags: Android学习之路
---

### Android的线程和线程池
在Android系统，线程主要分为主线程和子线程，主线程处理和界面相关的事情，而子线程一般用于执行耗时操作。AsyncTask底层是线程池；IntentService/HandlerThread底层是线程；

在Android中，线程的形态有很多种：
1. AsyncTask封装了线程池和Handler。
2. HandlerThread是具有消息循环的线程，内部可以使用handler
3. IntentService是一种Service，内部采用HandlerThread来执行任务，当任务执行完毕后IntentService会自动退出。由于它是一种Service，所以不容易被系统杀死

操作系统中，线程是操作系统调度的最小单元，同时线程又是一种受限的系统资源，其创建和销毁都会有相应的开销。同时当系统存在大量线程时，系统会通过时间片轮转的方式调度每个线程，因此线程不可能做到绝对的并发，除非线程数量小于等于CPU的核心数。

频繁创建销毁线程不明智，使用线程池是正确的做法。线程池会缓存一定数量的线程，通过线程池就可以避免因为频繁创建和销毁线程所带来的系统开销。

#### 主线程和子线程
主线程主要处理界面交互逻辑，由于用户随时会和界面交互，所以主线程在任何时候都需要有较高响应速度，则不能执行耗时的任务；

android3.0开始，网络访问将会失败并抛出NetworkOnMainThreadException这个异常，这样做是为了避免主线程由于被耗时操作所阻塞从而现ANR现象
#### Android中的线程形态

##### AsyncTask
AsyncTask是一种轻量级的异步任务类, 他可以在线程池中执行后台任务, 然后把执行的进度和最终的结果传递给主线程并在主线程更新UI. 从实现上来说. AsyncTask封装了Thread和Handler, 通过AsyncTask可以更加方便地执行后台任务,但是AsyncTask并不适合进行特别耗时的后台任务，对于特别耗时的任务来说, 建议使用线程池。

AsyncTask就是一个抽象的泛型类. 这三个泛型的意义
1. Params：参数的类型
2. Progress：后台任务的执行进度的类型
3. Result：后台任务的返回结果的类型

如果不需要传递具体的参数, 那么这三个泛型参数可以用Void来代替.

四个方法 ：
- onPreExecute()
在主线程执行, 在异步任务执行之前, 此方法会被调用, 一般可以用于做一些准备工作
- doInBackground()
 在线程池中执行, 此方法用于执行异步任务, 参数params表示异步任务的输入参数. 在此方法中可以通过publishProgress()方法来更新任务的进度, publishProgress()方法会调用onProgressUpdate()方法. 另外此方法需要返回计算结果给onPostExecute()
- onProgressUpdate()
 在主线程执行,在异步任务执行之后, 此方法会被调用, 其中result参数是后台任务的返回值, 即doInBackground()的返回值.
- onPostExecute()
 在主线程执行, 在异步任务执行之后, 此方法会被调用, 其中result参数是后台任务的返回值, 即doInBackground的返回值.

除了上述的四种方法,还有onCancelled(), 它同样在主线程执行, 当异步任务被取消时, onCancelled()方法会被调用, 这个时候onPostExecute()则不会被调用.

AsyncTask在使用过程中有一些条件限制
1. AsyncTask的类必须在主线程被加载, 这就意味着第一次访问AsyncTask必须发生在主线程, 这个问题不是绝对, 因为在Android 4.1及以上的版本已经被系统自动完成. 在Android 5.0的源码中, 可以看到ActivityThread#main()会调用AsyncTask#init()方法.
2. AsyncTask的对象必须在主线程中创建.
3. execute方法必须在UI线程调用.
4. 不要在程序中直接调用onPreExecute(), onPostExecute(), doInBackground和onProgressUpdate()
5. 一个AsyncTask对象只能执行一次, 即只能调用一次execute()方法, 否则会报运行时异常.
6. 在Android 1.6之前, AsyncTask是串行执行任务的; Android 1.6的时候AsyncTask开始采用线程池里处理并行任务; 但是Android 3.0开始, 为了避免AsyncTask带来的并发错误, AsyncTask又采用了一个线程来串行的执行任务. 尽管如此在3.0以后, 仍然可以通过AsyncTask#executeOnExecutor()方法来并行执行任务.

##### AsyncTask的工作原理
AsyncTask中有两个线程池(SerialExecutor和THREAD_POOL_EXECUTOR)和一个Handler(InternalHandler), 其中线程池SerialExecutor用于任务的排列, 而线程池THREAD_POOL_EXECUTOR用于真正的执行任务, 而InternalHandler用于将执行环境从线程切换到主线程, 其本质仍然是线程的调用过程.

AsyncTask的排队过程：首先系统会把AsyncTask#Params参数封装成FutureTask对象, FutureTask是一个并发类, 在这里充当了Runnable的作用. 接着这个FutureTask会交给SerialExecutor#execute()方法去处理. 这个方法首先会把FutureTask对象插入到任务队列mTasks中, 如果这个时候没有正在活动AsyncTask任务, 那么就会调用SerialExecutor#scheduleNext()方法来执行下一个AsyncTask任务. 同时当一个AsyncTask任务执行完后, AsyncTask会继续执行其他任务直到所有的任务都执行完毕为止, 从这一点可以看出, 在默认情况下, AsyncTask是串行执行的

##### HandlerThread
HandlerThread继承了Thread, 它是一种可以使用Handler的Thread, 它的实现也很简单, 就是run方法中通过Looper.prepare()来创建消息队列, 并通过Looper.loop()来开启消息循环, 这样在实际的使用中就允许在HandlerThread中创建Handler.

从HandlerThread的实现来看, 它和普通的Thread有显著的不同之处. 普通的Thread主要用于在run方法中执行一个耗时任务; 而HandlerThread在内部创建了消息队列, 外界需要通过Handler的消息方式来通知HandlerThread执行一个具体的任务. HandlerThread是一个很有用的类, 在Android中一个具体使用场景就是IntentService.

由于HandlerThread#run()是一个无线循环方法, 因此当明确不需要再使用HandlerThread时, 最好通过quit()或者quitSafely()方法来终止线程的执行.

##### IntentService
IntentSercie是一种特殊的Service，继承了Service并且是抽象类，任务执行完成后会自动停止，优先级远高于普通线程，适合执行一些高优先级的后台任务； IntentService封装了HandlerThread和Handler
1. onCreate方法自动创建一个HandlerThread
2. 用它的Looper构造了一个Handler对象mServiceHandler，这样通过mServiceHandler发送的消息都会在HandlerThread执行；
3. IntentServiced的onHandlerIntent方法是一个抽象方法，需要在子类实现，onHandlerIntent方法执行后，stopSelt(int startId)就会停止服务，如果存在多个后台任务，执行完最后一个stopSelf(int startId)才会停止服务。

#### Android线程池
优点：
1. 重用线程池的线程，减少线程创建和销毁带来的性能开销
2. 控制线程池的最大并发数，避免大量线程互相抢系统资源导致阻塞
3. 提供定时执行和间隔循环执行功能

Android中的线程池的概念来源于Java中的Executor, Executor是一个接口, 真正的线程池的实现为ThreadPoolExecutor.Android的线程池 大部分都是通 过Executor提供的工厂方法创建的。 ThreadPoolExecutor提供了一系列参数来配制线程池, 通过不同的参数可以创建不同的线程池. 而从功能的特性来分的话可以分成四类.

##### ThreadPoolExecutor
ThreadPoolExecutor是线程池的真正实现, 它的构造方法提供了一系列参数来配置线程池, 这些参数将会直接影响到线程池的功能特性.

    public ThreadPoolExecutor(int corePoolSize,
                     int maximumPoolSize,
                     long keepAliveTime,
                     TimeUnit unit,
                     BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
        Executors.defaultThreadFactory(), defaultHandler);
	}
- corePoolSize: 线程池的核心线程数, 默认情况下, 核心线程会在线程池中一直存活, 即使都处于闲置状态. 如果将ThreadPoolExecutor#allowCoreThreadTimeOut属性设置为true, 那么闲置的核心线程在等待新任务到来时会有超时的策略, 这个时间间隔由keepAliveTime属性来决定. 当等待时间超过了keepAliveTime设定的值那么核心线程将会终止.
- maximumPoolSize: 线程池所能容纳的最大线程数, 当活动线程数达到这个数值之后, 后续的任务将会被阻塞.
- keepAliveTime: 非核心线程闲置的超时时长, 超过这个时长, 非核心线程就会被回收. allowCoreThreadTimeOut这个属性为true的时候, 这个属性同样会作用于核心线程.
- unit: 用于指定keepAliveTime参数的时间单位, 这是一个枚举, 常用的有TimeUtil.MILLISECONDS(毫秒), TimeUtil.SECONDS(秒)以及TimeUtil.MINUTES(分)
- workQueue: 线程池中的任务队列, 通过线程池的execute方法提交的Runnable对象会存储在这个参数中.
- threadFactory: 线程工厂, 为线程池提供创建新线程的功能. ThreadFactory是一个接口.

ThreadPoolExecutor执行任务大致遵循如下规则:
1. 如果线程池中的线程数量未达到核心线程的数量, 那么会直接启动一个核心线程来执行任务.
2. 如果线程池中的线程数量已经达到或者超过核心线程的数量, 那么任务会被插入到任务队列中排队等待执行.
3. 如果在步骤2中无法将任务插入到任务队列中, 这通常是因为任务队列已满, 这个时候如果线程数量未达到线程池的规定的最大值, 那么会立刻启动一个非核心线程来执行任务.
4. 如果步骤3中的线程数量已经达到最大值的时候, 那么会拒绝执行此任务,ThreadPoolExecutor会调用RejectedExecution方法来通知调用者.

AsyncTask的THREAD_POOL_EXECUTOR线程池配置:
1. 核心线程数等于CPU核心数+1
2. 线程池最大线程数为CPU核心数的2倍+1
3. 核心线程无超时机制，非核心线程的闲置超时时间为1秒
4. 任务队列容量是128

##### 线程池的分类
**FixedThreadPool**
通过Executor#newFixedThreadPool()方法来创建. 它是一种线程数量固定的线程池, 当线程处于空闲状态时, 它们并不会被回收, 除非线程池关闭了. 当所有的线程都处于活动状态时, 新任务都会处于等待状态, 直到有线程空闲出来. 由于FixedThreadPool只有核心线程并且这些核心线程不会被回收, 这意味着它能够更加快速地响应外界的请求.

** CachedThreadPool**
通过Executor#newCachedThreadPool()方法来创建. 它是一种线程数量不定的线程池, 它只有非核心线程, 并且其最大值线程数为Integer.MAX_VALUE. 这就可以认为这个最大线程数为任意大了. 当线程池中的线程都处于活动的时候, 线程池会创建新的线程来处理新任务, 否则就会利用空闲的线程来处理新任务. 线程池中的空闲线程都有超时机制, 这个超时时长为60S, 超过这个时间那么空闲线程就会被回收.

和FixedThreadPool不同的是, CachedThreadPool的任务队列其实相当于一个空集合, 这将导致任何任务都会立即被执行, 因为在这种场景下SynchronousQueue是无法插入任务的. SynchronousQueue是一个非常特殊的队列, 在很多情况下可以把它简单理解为一个无法存储元素的队列. 在实际使用中很少使用.这类线程比较适合执行大量的耗时较少的任务

**ScheduledThreadPool**
通过Executor#newScheduledThreadPool()方法来创建. 它的核心线程数量是固定的, 而非核心线程数是没有限制的, 并且当非核心线程闲置时会立刻被回收掉. 这类线程池用于执行定时任务和具有固定周期的重复任务

**SingleThreadExecutor**
通过Executor#newSingleThreadPool()方法来创建. 这类线程池内部只有一个核心线程, 它确保所有的任务都在同一个线程中按顺序执行. 这类线程池意义在于统一所有的外界任务到一个线程中, 这使得在这些任务之间不需要处理线程同步的问题
