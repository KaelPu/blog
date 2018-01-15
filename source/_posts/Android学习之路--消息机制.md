---
title: Android学习之路 -- 消息机制
date: 2017-04-11 21:25:48
tags: Android学习之路
---

### Android的消息机制
Android的消息机制主要是指Handler的运行机制。从开发的角度来说，Handler是Android消息机制的上层接口。Handler的运行需要底层的 MessageQueue 和 Looper 的支撑。
- MessageQueue是一个消息队列，内部存储了一组消息，以队列的形式对外提供插入和删除的工作，内部采用单链表的数据结构来存储消息列表。
- Lopper会以无限循环的形式去查找是否有新消息，如果有就处理消息，否则就一直等待着。
- ThreadLocal并不是线程，它的作用是在每个线程中存储数据。Handler通过ThreadLocal可以获取每个线程中的Looper。
- 线程是默认没有Looper的，使用Handler就必须为线程创建Looper。我们经常提到的主线程，也叫UI线程，它就是ActivityThread，被创建时就会初始化Looper。

#### Android的消息机制概述
![](http://wujingchao.com/assets/Handler%E5%B7%A5%E4%BD%9C%E6%9C%BA%E5%88%B6.png)
- Handler的主要作用是将某个任务切换到Handler所在的线程中去执行。为什么Android要提供这个功能呢？这是因为Android规定访问UI只能通过主线程，如果子线程访问UI,程序可能会导致ANR。那我们耗时操作在子线程执行完毕后，我们需要将一些更新UI的操作切换到主线程当中去。所以系统就提供了Handler。
- 系统为什么不允许在子线程中去访问UI呢？ 因为Android的UI控件不是线程安全的，多线程并发访问可能会导致UI控件处于不可预期的状态，为什么不加锁？因为加锁机制会让UI访问逻辑变得复杂；其次锁机制会降低UI访问的效率，因为锁机制会阻塞某些线程的执行。所以Android采用了高效的单线程模型来处理UI操作。
- Handler创建时会采用当前线程的Looper来构建内部的消息循环系统，如果当前线程没有Looper就会报错。Handler可以通过post方法发送一个Runnable到消息队列中，也可以通过send方法发送一个消息到消息队列中，其实post方法最终也是通过send方法来完成。
- MessageQueue的enqueueMessage方法最终将这个消息放到消息队列中，当Looper发现有新消息到来时，处理这个消息，最终消息中的Runnable或者Handler的handleMessage方法就会被调用，注意**Looper是运行Handler所在的线程中的**，这样一来业务逻辑就切换到了Handler所在的线程中去执行了。

#### Android的消息机制分析

##### ThreadLocal的工作原理
ThreadLocal是一个线程内部的数据存储类，通过它可以在指定线程中存储数据，数据存储后，只有在指定线程中可以获取到存储的数据，对于其他线程来说无法获得数据。

在某些特殊的场景下，ThreadLocal可以轻松实现一些很复杂的功能。Looper、ActivityThread以及AMS都用到了ThreadLocal。当某些数据是以线程为作用域并且不同线程具有不同的数据副本的时候，就可以考虑采用ThreadLocal。

对于Handler来说，它需要获取当前线程的Looper,而Looper的作用于就是线程并且不同的线程具有不同的Looper，通过ThreadLocal可以轻松实现线程中的存取。 

ThreadLocal的另一个使用场景是可以让监听器作为线程内的全局对象而存在，在线程内部只要通过get方法就可以获取到监听器。如果不采用ThreadLocal，只能采用函数参数调用和静态变量的方式。而第一种方式在调用栈很深时很糟糕，第二种方式不具有扩展性，比如同时多个线程执行。

虽然在不同线程访问同一个ThreadLocal对象，但是获得的值却是不同的。不同线程访问同一个ThreadLoacl的get方法，ThreadLocal的get方法会从各自的线程中取出一个数组，然后再从数组中根据当前ThreadLocal的索引去查找对应的Value值。
ThreadLocal的set方法：

    public void set(T value) {
        Thread currentThread = Thread.currentThread();
        //通过values方法获取当前线程中的ThreadLoacl数据——localValues
        Values values = values(currentThread);
        if (values == null) {
            values = initializeValues(currentThread);
        } 
        values.put(this, value);
    }
1. 在 localValues 内部有一个数组： private Object[] table ，ThreadLocal的值就存在这个数组中。
2. ThreadLocal的值在table数组中的存储位置总是ThreadLocal的reference字段所标识的对象的下一个位置。

ThreadLocal的get方法：

    	public T get() {
        // Optimized for the fast path.
        Thread currentThread = Thread.currentThread();
        Values values = values(currentThread);//找到localValues对象
        if (values != null) {
            Object[] table = values.table;
            int index = hash & values.mask;
            if (this.reference == table[index]) {//找到ThreadLocal的reference对象在table数组中的位置
            	return (T) table[index + 1];//reference字段所标识的对象的下一个位置就是ThreadLocal的值
            }
        } else {
        	values = initializeValues(currentThread);
        } 
        return (T) values.getAfterMiss(this);
    }
从ThreadLocal的set/get方法可以看出，它们所操作的对象都是当前线程的localValues对象的table数组，因此在不同线程中访问同一个ThreadLocal的set/get方法，它们ThreadLocal的读/写操作仅限于各自线程的内部。理解ThreadLocal的实现方式有助于理解Looper的工作原理。

##### 消息队列的工作原理
消息队列指的是MessageQueue，主要包含两个操作：插入和读取。读取操作本身会伴随着删除操作。

MessageQueue内部通过一个单链表的数据结构来维护消息列表，这种数据结构在插入和删除上的性能比较有优势。

插入和读取对应的方法分别是：enqueueMessage和next方法。
enqueueMessage()的源码实现主要操作就是单链表的插入操作
next()的源码实现也是从单链表中取出一个元素的操作，next()方法是一个无线循环的方法，如果消息队列中没有消息，那么next方法会一直阻塞在这里。当有新消息到来时，next()方法会返回这条消息并将其从单链表中移除。

##### Looper的工作原理
Looper在Android的消息机制中扮演着消息循环的角色，具体来说就是它会不停地从MessageQueue中查看是否有新消息，如果有新消息就会立即处理，否则就一直阻塞在那里。

通过Looper.prepare()方法即可为当前线程创建一个Looper，再通过Looper.loop()开启消息循环。prepareMainLooper()方法主要给主线程也就是ActivityThread创建Looper使用的，本质也是通过prepare方法实现的。

Looper提供quit和quitSafely来退出一个Looper，区别在于quit会直接退出Looper，而quitSafely会把消息队列中已有的消息处理完毕后才安全地退出。 Looper退出后，这时候通过Handler发送的消息会失败，Handler的send方法会返回false。
在子线程中，如果手动为其创建了Looper，在所有事情做完后，应该调用Looper的quit方法来终止消息循环，否则这个子线程就会一直处于等待状态；而如果退出了Looper以后，这个线程就会立刻终止，因此建议不需要的时候终止Looper。
loop()方法会调用MessageQueue的next()方法来获取新消息，而next是是一个阻塞操作，但没有信息时，next方法会一直阻塞在那里，这也导致loop方法一直阻塞在那里。如果MessageQueue的next方法返回了新消息，Looper就会处理这条消息：msg.target.dispatchMessage(msg),这里的msg.target是发送这条消息的Handler对象，这样Handler发送的消息最终又交给Handler来处理了。

##### Handler的工作原理
Handler的工作主要包含消息的发送和接收过程。通过post的一系列方法和send的一系列方法来实现。

Handler发送过程仅仅是向消息队列中插入了一条消息。MessageQueue的next方法就会返回这条消息给Looper，Looper拿到这条消息就开始处理，最终消息会交给Handler的dispatchMessage()来处理，这时Handler就进入了处理消息的阶段。

handler处理消息的过程
![](http://upload-images.jianshu.io/upload_images/2534721-cde6d86b8220f4f3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Handler的构造方法
1. 派生Handler的子类
2. 通过Callback
`Handler handler = new Handler(callback)`
其中，callback接口定义如下
        public interface Callback{
            public boolean handleMessage(Message msg);
        }
3. 通过Looper
		public Handler(Looper looper){
        this(looper,null,false);
        }

#### 主线程的消息循环
Android的主线程就是ActivityThread，主线程的入口方法为main(String[] args)，在main方法中系统会通过Looper.prepareMainLooper()来创建主线程的Looper以及MessageQueue，并通过Looper.loop()来开启主线程的消息循环。

ActivityThread通过ApplicationThread和AMS进行进程间通信，AMS以进程间通信的方式完成ActivityThread的请求后会回调ApplicationThread中的Binder方法，然后ApplicationThread会向H发送消息，H收到消息后会将ApplicationThread中的逻辑切换到ActivityTread中去执行，即切换到主线程中去执行。四大组件的启动过程基本上都是这个流程。

>Looper.loop()，这里是一个死循环，如果主线程的Looper终止，则应用程序会抛出异常。那么问题来了，既然主线程卡在这里了，（1）那Activity为什么还能启动；（2）点击一个按钮仍然可以响应？ 
>
>问题1：startActivity的时候，会向AMS（ActivityManagerService）发一个跨进程请求（AMS运行在系统进程中），之后AMS启动对应的Activity；AMS也需要调用App中Activity的生命周期方法（不同进程不可直接调用），AMS会发送跨进程请求，然后由App的ActivityThread中的ApplicationThread会来处理，ApplicationThread会通过主线程线程的Handler将执行逻辑切换到主线程。重点来了，主线程的Handler把消息添加到了MessageQueue，Looper.loop会拿到该消息，并在主线程中执行。这就解释了为什么主线程的Looper是个死循环，而Activity还能启动，因为四大组件的生命周期都是以消息的形式通过UI线程的Handler发送，由UI线程的Looper执行的。
>
>  问题2：和问题1原理一样，点击一个按钮最终都是由系统发消息来进行的，都经过了Looper.loop()处理。 问题2详细分析请看原书作者的[Android中MotionEvent的来源和ViewRootImpl](http://blog.csdn.net/singwhatiwanna/article/details/50775201)。