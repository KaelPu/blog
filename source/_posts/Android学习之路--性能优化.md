---
title: Android学习之路 -- 性能优化
date: 2017-12-20 22:24:13
tags: Android学习之路
---

### Android的性能优化方法
#### 布局优化
布局优化的思想就是尽量减少布局文件的层级，这样绘制界面时工作量就少了，那么程序的性能自然就高了。

- 删除无用的控件和层级
- 有选择的使用性能较低的ViewGroup，如果布局中既可以使用Linearlayout也可以使用RelativeLayout，那就是用LinearLayout，因为RelativeLayout功能比较复杂，它的布局过程需要花费更多的CPU时间。
有时候通过LinearLayou无法实现产品效果，需要通过嵌套来完成，这种情况还是推荐使用RelativeLayout，因为ViewGroup的嵌套相当于增加了布局的层级，同样降低程序性能。
- 采用标签、标签和ViewStub
	- include标签
<include>标签用于布局重用，可以将一个指定的布局文件加载到当前布局文件中。<include>只支持android:layout开头的属性，当然android:id这个属性是个特例；如果指定了android:layout这种属性，那么要求android:layoutwidth和android:layout_height必须存在，否则android:layout属性无法生效。如果<include>指定了id属性，同时被包含的布局文件的根元素也指定了id属性，会以<include>指定的这个id属性为准。
	- merge标签
<merge>标签一般和<include>标签一起使用从而减少布局的层级。如果当前布局是一个竖直方向的LinearLayout，这个时候被包含的布局文件也采用竖直的LinearLayout，那么显然被包含的布局文件中的这个LinearLayout是多余的，通过<merge>标签就可以去掉多余的那一层LinearLayout。
	- ViewStub
ViewStub意义在于按需加载所需的布局文件，因为实际开发中，有很多布局文件在正常情况下是不会现实的，比如网络异常的界面，这个时候就没必要在整个界面初始化的时候将其加载进来，在需要使用的时候再加载会更好。在需要加载ViewStub布局时：
			 ((ViewStub)findViewById(R.id.stub_import)).setVisibility(View.VISIBLE);
            //或者
            View importPanel = ((ViewStub)findViewById(R.id.stub_import)).inflate();
当ViewStub通过setVisibility或者inflate方法加载后，ViewStub就会被它内部的布局替换掉，ViewStub也就不再是整个布局结构的一部分了。

##### 绘制优化
View的onDraw方法要避免执行大量的操作；

- onDraw中不要创建大量的局部对象，因为onDraw方法会被频繁调用，这样就会在一瞬间产生大量的临时对象，不仅会占用过多内存还会导致系统频繁GC，降低程序执行效率。
- onDraw也不要做耗时的任务，也不能执行成千上万的循环操作，尽管每次循环都很轻量级，但大量循环依然十分抢占CPU的时间片，这会造成View的绘制过程不流畅。根据Google官方给出的标准，View绘制保持在60fps是最佳的，这也就要求每帧的绘制时间不超过16ms(1000/60)；所以要尽量降低onDraw方法的复杂度。

##### 内存泄露优化
内存泄露是最容易犯的错误之一，内存泄露优化主要分两个方面；一方面是开发过程中避免写出有内存泄露的代码，另一方面是通过一些分析工具如LeakCanary或MAT来找出潜在的内存泄露继而解决。
- 静态变量导致的内存泄露
 比如Activity内，一静态Conext引用了当前Activity，所以当前Activity无法释放。或者一静态变量，内部持有了当前Activity，Activity在需要释放的时候依然无法释放。
- 单例模式导致的内存泄露
 比如单例模式持有了Activity，而且也没用解注册的操作。因为单例模式的生命周期和Application保存一致，生命周期比Activity要长，这样一来就导致Activity对象无法及时被释放。
- 属性动画导致的内存泄露 
属性动画中有一类无限循环的动画，如果在Activity播放了此类动画并且没有在onDestroy中去停止动画，那么动画会一直播放下去，并且这个时候Activity的View会被动画持有，而View又持有了Activity，最终导致Activity无法释放。解决办法是在Activity的onDrstroy中调用animator.cancel()来停止动画。

##### 响应速度优化和ANR日志分析
响应速度优化的核心思想就是避免在主线程中去做耗时操作，将耗时操作放在其他线程当中去执行。Activity如果5秒无法响应屏幕触摸事件或者键盘输入事件就会触发ANR，而BroadcastReceiver如果10秒还未执行完操作也会出现ANR。

当一个进程发生ANR以后系统会在/data/anr的目录下创建一个文件traces.txt，通过分析该文件就能定位出ANR的原因。

通过一个例子来了解如何去分析文件, 首先在onCreate()添加如下代码, 让主线程等待一个锁,然后点击返回5秒后会出现ANR。

    @Override
    protected void onCreate(Bundle savedInstanceState) {
       super.onCreate(savedInstanceState);
       setContentView(R.layout.activity_main);
       // 以下代码是为了模拟一个ANR的场景来分析日志
       new Thread(new Runnable() {
           @Override
           public void run() {
               testANR();
           }
       }).start();
       SystemClock.sleep(10);
       initView();
    }
    /**
    *  以下两个方法用来模拟出一个稍微不好发现的ANR
    */
    private synchronized void testANR(){
       SystemClock.sleep(3000 * 1000);
    }
    private synchronized void initView(){}

这样会出现ANR, 然后导出/data/anr/straces.txt文件. 因为内容比较多只贴出关键部分

    DALVIK THREADS (15):
    "main" prio=5 tid=1 Blocked
      | group="main" sCount=1 dsCount=0 obj=0x73db0970 self=0xf4306800
      | sysTid=19949 nice=0 cgrp=apps sched=0/0 handle=0xf778d160
      | state=S schedstat=( 151056979 25055334 199 ) utm=5 stm=9 core=1 HZ=100
      | stack=0xff5b2000-0xff5b4000 stackSize=8MB
      | held mutexes=
      at com.szysky.note.androiddevseek_15.MainActivity.initView(MainActivity.java:0)
      - waiting to lock <0x2fbcb3de> (a com.szysky.note.androiddevseek_15.MainActivity) 
      - held by thread 15
      at com.szysky.note.androiddevseek_15.MainActivity.onCreate(MainActivity.java:42)
这段可以看出最后指明了ANR发生的位置在ManiActivity的42行. 并且通过上面看出initView方法正在等待一个锁<0x2fbcb3de>锁的类型是一个MainActivity对象. 并且这个锁已经被线程id为15(tid=15)的线程持有了. 接下来找一下线程15

    "Thread-404" prio=5 tid=15 Sleeping
      | group="main" sCount=1 dsCount=0 obj=0x12c00f80 self=0xeb95bc00
      | sysTid=19985 nice=0 cgrp=apps sched=0/0 handle=0xef34be80
      | state=S schedstat=( 391248 0 1 ) utm=0 stm=0 core=2 HZ=100
      | stack=0xe2bfe000-0xe2c00000 stackSize=1036KB
      | held mutexes=
      at java.lang.Thread.sleep!(Native method)
      - sleeping on <0x2e3896a7> (a java.lang.Object)
      at java.lang.Thread.sleep(Thread.java:1031)
      - locked <0x2e3896a7> (a java.lang.Object)
      at java.lang.Thread.sleep(Thread.java:985)
      at android.os.SystemClock.sleep(SystemClock.java:120)
      at com.szysky.note.androiddevseek_15.MainActivity.testANR(MainActivity.java:50)
      - locked <0x2fbcb3de> (a com.szysky.note.androiddevseek_15.MainActivity)
tid = 15 就是相关信息如上, 首行已经标出线程的状态为Sleeping, 原因在50行, 就是SystemClock.sleep(3000 * 1000);这句话. 也就是testANR(). 而最后一行也表明了持有的locked<0x2fbcb3de>就是主线程在等待的那个锁对象.

##### ListView优化和Bitmap优化
ListView/GridView优化：
1. 采用ViewHolder避免在getView中执行耗时操作
2. 其次通过列表的滑动状态来控制任务的执行频率，比如快速滑动时不是和开启大量异步任务
3. 最后可以尝试开启硬件加速使得ListView的滑动更加流畅。

Bitmap优化：主要是想是根据需要对图片进行采样显示，详细请参考12章。

###15.1.6 线程优化
主要思想就是采用线程池, 避免程序中存在大量的Thread. 线程池可以重用内部的线程, 避免了线程创建和销毁的性能开销. 同时线程池还能有效的控制线程的最大并发数, 避免了大量线程因互相抢占系统资源从而导致阻塞现象的发生.详细参考第11章的内容。

##### 一些性能优化的小建议
1. 避免创建过多的对象，尤其在循环、onDraw这类方法中，谨慎创建对象；
2. 不要过多的使用枚举，枚举占用的内存空间比整形大。[Android 中如何使用annotion替代Enum](http://szysky.com/2016/05/20/Android-%E4%B8%AD%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8annotion%E6%9B%BF%E4%BB%A3Enum/)
3. 常量使用static final来修饰；
4. 使用一些Android特有的数据结构，比如 SparseArray 和 Pair 等，他们都具有更好的性能；
5. 适当的使用软引用和弱引用；
6. 采用内存缓存和磁盘缓存；
7. 尽量采用静态内部类，这样可以避免非静态内部类隐式持有外部类所导致的内存泄露问题。

#### 内存泄漏分析工具MAT
MAT全程Eclipse Memory Analyzer, 是一个内存泄漏分析工具. 下载后解压即可. 下载地址http://www.eclipse.org/mat/downloads.php. 这里仅简单说一下. 这个我没有手动去实践, 就当个记录, 因为现在Android Studio可以直接分析hprof文件.

可以手动写一个会造成内存泄漏的代码, 然后打开DDMS, 然后选中要分析的进程, 然后单击Dump HPROF file这个按钮. 等一小段会生成一个文件. 这个文件不能被MAT直接识别. 需要使用Android SDK中的工具进行格式转换一下.这个工具在platform-conv文件夹下

`hprof-conv 要转换的文件名 输出的文件名 `文件名的签名有包名.

然后打开MAT通过菜单打开转换后的这个文件. 这里常用的就有两个

- Histogram: 可以直观的看出内存中不同类型的buffer的数量和占用内存大小
- Dominator Tree: 把内存中的对象按照从大到小的顺序进行排序, 并且可以分析对象之间的引用关系, 内存泄漏分析就是通过这个完成的.

分析内存泄漏的时候需要分析Dominator Tree里面的内存信息, 一般会不直接显示出来, 可以按照从大到小的顺序去排查一遍. 如果发生了了泄漏, 那么在泄漏对象处右键单击Path To GC Roots->exclude wake/soft references. 可以看到最终是什么对象导致的无法释放. 刚才的操作之所以排除软引用和弱引用是因为,大部分情况下这两种类型都可以被gc回收掉,所以基本也就不会造成内存泄漏.

同样这里也可以使用搜索功能, 假如我们手动模拟了内存泄漏, 泄漏的对象就是Activity那么我们back退出重进循环几次, 会发现其实很多个Activit对象.
