---
title: Android学习之路 -- View的工作原理(一)
date: 2017-05-24 20:24:42
tags: Android学习之路
---

### View的工作原理
主要内容
- View的工作原理
- 自定义View的实现方式
- 自定义View的底层工作原理，比如View的测量流程、布局流程、绘制流程
- View常见的回调方法，比如构造方法、onAttach.onVisibilityChanged/onDetach等

#### 初识ViewRoot和DecorView
ViewRoot的实现是 ViewRootImpl 类，是连接WindowManager和DecorView的纽带，View的三大流程（ mearsure、layout、draw） 均是通过ViewRoot来完成。当Activity对象被创建完毕后，会将DecorView添加到Window中，同时创建 ViewRootImpl 对象，并将ViewRootImpl 对象和DecorView建立连接，源码如下：

	root = new ViewRootImpl(view.getContext(),display);
	root.setView(view,wparams, panelParentView);
View的绘制流程是从ViewRoot的performTraversals开始的
![image.png](https://upload-images.jianshu.io/upload_images/1967257-f245ffbefe056fb6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1. measure用来测量View的宽高
2. layout来确定View在父容器中的位置
3. draw负责将View绘制在屏幕上

performTraversals会依次调用 performMeasure 、 performLayout 和performDraw 三个方法，这三个方法分别完成顶级View的measure、layout和draw这三大流程。其中 performMeasure 中会调用 measure 方法，在 measure 方法中又会调用 onMeasure 方法，在 onMeasure 方法中则会对所有子元素进行measure过程，这样就完成了一次measure过程；子元素会重复父容器的measure过程，如此反复完成了整个View数的遍历。另外两个过程同理。

- Measure完成后, 可以通过getMeasuredWidth 、getMeasureHeight 方法来获取View测量后的宽/高。特殊情况下，测量的宽高不等于最终的宽高，详见后面。
- Layout过程决定了View的四个顶点的坐标和实际View的宽高，完成后可通过 getTop 、 getBotton 、 getLeft 和 getRight 拿到View的四个定点坐标。

DecorView作为顶级View，其实是一个 FrameLayout ，它包含一个竖直方向的 LinearLayout ，这个 LinearLayout 分为标题栏和内容栏两个部分。
![image.png](https://upload-images.jianshu.io/upload_images/1967257-a52532aa98246bae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在Activity通过setContextView所设置的布局文件其实就是被加载到内容栏之中的。这个内容栏的id是 R.android.id.content ，通过 ``` ViewGroup content = findViewById(R.android.id.content);
可以得到这个contentView。View层的事件都是先经过DecorView，然后才传递到子View。


#### 理解MeasureSpec
MeasureSpec决定了一个View的尺寸规格。但是父容器会影响View的MeasureSpec的创建过程。系统将View的 LayoutParams 根据父容器所施加的规则转换成对应的MeasureSpec，然后根据这个MeasureSpec来测量出View的宽高。

##### MeasureSpec
MeasureSpec代表一个32位int值，高2位代表SpecMode（ 测量模式） ，低30位代表SpecSize（ 在某个测量模式下的规格大小） 。
SpecMode有三种：
- UNSPECIFIED ：父容器不对View进行任何限制，要多大给多大，一般用于系统内部
- EXACTLY：父容器检测到View所需要的精确大小，这时候View的最终大小就是SpecSize所指定的值，对应LayoutParams中的 match_parent 和具体数值这两种模式
- AT_MOST ：对应View的默认大小，不同View实现不同，View的大小不能大于父容器的SpecSize，对应 LayoutParams 中的 wrap_content

##### MeasureSpec和LayoutParams的对应关系
对于DecorView，其MeasureSpec由窗口的尺寸和其自身的LayoutParams共同确定。而View的MeasureSpec由父容器的MeasureSpec和自身的LayoutParams共同决定。

View的measure过程由ViewGroup传递而来，参考ViewGroup的 measureChildWithMargins 方法，通过调用子元素的 getChildMeasureSpec 方法来得到子元素的MeasureSpec，再调用子元素的 measure 方法。
![image.png](https://upload-images.jianshu.io/upload_images/1967257-c50ccd20a316e977.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
parentSize是指父容器中目前可使用的大小。

1. 当View采用固定宽/高时（ 即设置固定的dp/px） ,不管父容器的MeasureSpec是什么，View的MeasureSpec都是EXACTLY模式，并且大小遵循我们设置的值。
2. 当View的宽/高是 match_parent 时，View的MeasureSpec都是EXACTLY模式并且其大小等于父容器的剩余空间。
3. 当View的宽/高是 wrap_content 时，View的MeasureSpec都是AT_MOST模式并且其大小不能超过父容器的剩余空间。
4. 父容器的UNSPECIFIED模式，一般用于系统内部多次Measure时，表示一种测量的状态，一般来说我们不需要关注此模式。




#### View的工作流程

##### measure过程
**View的measure过程**

直接继承View的自定义控件需要重写 onMeasure 方法并设置 wrap_content （ 即specMode是 AT_MOST 模式） 时的自身大小，否则在布局中使用 wrap_content 相当于使用 match_parent 。对于非 wrap_content 的情形，我们沿用系统的测量值即可。

	  @Override
	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
        int widthSpecSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);
        int heightSpecSize = MeasureSpec.getSize(heightMeasureSpec);
          // 在 MeasureSpec.AT_MOST 模式下，给定一个默认值mWidth,mHeight。默认宽高灵活指定
          //参考TextView、ImageView的处理方式
          //其他情况下沿用系统测量规则即可
        if (widthSpecMode == MeasureSpec.AT_MOST
                && heightSpecMode == MeasureSpec.AT_MOST) {
            setMeasuredDimension(mWith, mHeight);
        } else if (widthSpecMode == MeasureSpec.AT_MOST) {
            setMeasuredDimension(mWith, heightSpecSize);
        } else if (heightSpecMode == MeasureSpec.AT_MOST) {
            setMeasuredDimension(widthSpecSize, mHeight);
        }
	}
**ViewGroup的measure过程**

ViewGroup是一个抽象类，没有重写View的 onMeasure 方法，但是它提供了一个 measureChildren 方法。这是因为不同的ViewGroup子类有不同的布局特性，导致他们的测量细节各不相同，比如 LinearLayout 和 RelativeLayout ,因此ViewGroup没办法同一实现 onMeasure方法。

measureChildren方法的流程：
1. 取出子View的 LayoutParams
2. 通过 getChildMeasureSpec 方法来创建子元素的 MeasureSpec
3. 将 MeasureSpec 直接传递给View的measure方法来进行测量

通过LinearLayout的onMeasure方法里来分析ViewGroup的measure过程：
1. LinearLayout在布局中如果使用match_parent或者具体数值，测量过程就和View一致，即高度为specSize
2. LinearLayout在布局中如果使用wrap_content，那么它的高度就是所有子元素所占用的高度总和，但不超过它的父容器的剩余空间
3. LinearLayout的的最终高度同时也把竖直方向的padding考虑在内

View的measure过程是三大流程中最复杂的一个，measure完成以后，通过 getMeasuredWidth/Height 方法就可以正确获取到View的测量后宽/高。在某些情况下，系统可能需要多次measure才能确定最终的测量宽/高，所以在onMeasure中拿到的宽/高很可能不是准确的。

==如果我们想要在Activity启动的时候就获取一个View的宽高，怎么操作呢？==因为View的measure过程和Activity的生命周期并不是同步执行，无法保证在Activity的 onCreate、onStart、onResume 时某个View就已经测量完毕。所以有以下四种方式来获取View的宽高：
1. Activity/View#onWindowFocusChanged
onWindowFocusChanged这个方法的含义是：VieW已经初始化完毕了，宽高已经准备好了，需要注意：它会被调用多次，当Activity的窗口得到焦点和失去焦点均会被调用。
2. view.post(runnable)
通过post将一个runnable投递到消息队列的尾部，当Looper调用此runnable的时候，View也初始化好了。
3. ViewTreeObserver
使用 ViewTreeObserver 的众多回调可以完成这个功能，比如OnGlobalLayoutListener 这个接口，当View树的状态发送改变或View树内部的View的可见性发生改变时，onGlobalLayout 方法会被回调，这是获取View宽高的好时机。需要注意的是，伴随着View树状态的改变， onGlobalLayout 会被回调多次。
4. view.measure(int widthMeasureSpec,int heightMeasureSpec)
手动对view进行measure。需要根据View的layoutParams分情况处理：
	- match_parent：
无法measure出具体的宽高，因为不知道父容器的剩余空间，无法测量出View的大小
	- 具体的数值（ dp/px）:
            int widthMeasureSpec = MeasureSpec.makeMeasureSpec(100,MeasureSpec.EXACTLY);
            int heightMeasureSpec = MeasureSpec.makeMeasureSpec(100,MeasureSpec.EXACTLY);
            view.measure(widthMeasureSpec,heightMeasureSpec);
	- wrap_content：
            int widthMeasureSpec = MeasureSpec.makeMeasureSpec((1<<30)-1,MeasureSpec.AT_MOST);
            // View的尺寸使用30位二进制表示，最大值30个1，在AT_MOST模式下，我们用View理论上能支持的最大值去构造MeasureSpec是合理的
            int heightMeasureSpec = MeasureSpec.makeMeasureSpec((1<<30)-1,MeasureSpec.AT_MOST);
            view.measure(widthMeasureSpec,heightMeasureSpec);

##### layout过程
layout的作用是ViewGroup用来确定子View的位置，当ViewGroup的位置被确定后，它会在onLayout中遍历所有的子View并调用其layout方法，在 layout 方法中， onLayout 方法又会被调用。

View的 layout 方法确定本身的位置，源码流程如下： 
1. setFrame 确定View的四个顶点位置，即确定了View在父容器中的位置
2. 调用 onLayout 方法，确定所有子View的位置，和onMeasure一样，onLayout的具体实现和布局有关，因此View和ViewGroup均没有真正实现 onLayout 方法。

以LinearLayout的 onLayout 方法为例：
1. 遍历所有子View并调用 setChildFrame 方法来为子元素指定对应的位置
2. setChildFrame 方法实际上调用了子View的 layout 方法，形成了递归

==View的测量宽高和最终宽高的区别：==
在View的默认实现中，View的测量宽高和最终宽高相等，只不过测量宽高形成于measure过程，最终宽高形成于layout过程。但重写view的layout方法可以使他们不相等。

##### draw过程
View的绘制过程遵循如下几步：
1. 绘制背景 drawBackground(canvas)
2. 绘制自己 onDraw
3. 绘制children dispatchDraw 遍历所有子View的 draw 方法
4. 绘制装饰 onDrawScrollBars

ViewGroup会默认启用 setWillNotDraw 为ture，导致系统不会去执行 onDraw ，所以自定义ViewGroup需要通过onDraw来绘制内容时，必须显式的关闭 WILL_NOT_DRAW 这个优化标记位，即调用 setWillNotDraw(false);