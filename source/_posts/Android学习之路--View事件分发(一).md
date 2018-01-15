---
title: Android学习之路 -- View事件分发(一)
date: 2017-05-08 18:32:43
tags: Android学习之路
---

### View的事件体系
本章介绍View的事件分发和滑动冲突问题的解决方案。
#### view的基础知识
View的位置参数、MotionEvent和TouchSlop对象、VelocityTracker、GestureDetector和Scroller对象。
##### 什么是view
View是Android中所有控件的基类，View的本身可以是单个空间，也可以是多个控件组成的一组控件，即ViewGroup，ViewGroup继承自View，其内部可以有子View，这样就形成了View树的结构。

##### View的位置参数
View的位置主要由它的四个顶点来决定，即它的四个属性：top、left、right、bottom，分别表示View左上角的坐标点（ top，left） 以及右下角的坐标点（ right，bottom） 。
![](http://img2016.itdadao.com/d/file/tech/2016/11/02/it289208021201281.png)
同时，我们可以得到View的大小：
        
    width = right - left
    height = bottom - top
而这四个参数可以由以下方式获取：
    
    Left = getLeft();
    Right = getRight();
    Top = getTop();
    Bottom = getBottom();
Android3.0后，View增加了x、y、translationX和translationY这几个参数。其中x和y是View左上角的坐标，而translationX和translationY是View左上角相对于容器的偏移量。他们之间的换算关系如下：
    
    x = left + translationX;
    y = top + translationY;

top,left表示原始左上角坐标，而x,y表示变化后的左上角坐标。在View没有平移时，x=left,y=top。==View平移的过程中，top和left不会改变，改变的是x、y、translationX和translationY。==
##### MotionEvent和TouchSlop
**MotionEvent**
事件类型
- ACTION_DOWN 手指刚接触屏幕
- ACTION_MOVE 手指在屏幕上移动
- ACTION_UP 手指从屏幕上松开

点击事件类型
- 点击屏幕后离开松开，事件序列为DOWN->UP
- 点击屏幕滑动一会再松开，事件序列为DOWN->MOVE->...->MOVE->UP 

通过MotionEven对象我们可以得到事件发生的x和y坐标，我们可以通过getX/getY和getRawX/getRawY得到。它们的区别是：getX/getY返回的是相对于当前View左上角的x和y坐标，getRawX/getRawY返回的是相对于手机屏幕左上角的x和y坐标。

**TouchSloup**
TouchSloup是系统所能识别出的被认为是滑动的最小距离，这是一个常量，与设备有关，可通过以下方法获得：

	ViewConfiguration.get(getContext()).getScaledTouchSloup().
当我们处理滑动时，比如滑动距离小于这个值，我们就可以过滤这个事件（系统会默认过滤），从而有更好的用户体验。
##### VelocityTracker、GestureDetector和Scroller
**VelocityTracker**
速度追踪，用于追踪手指在滑动过程中的速度，包括水平放向速度和竖直方向速度。使用方法：
- 在View的onTouchEvent方法中追踪当前单击事件的速度
        VelocityRracker velocityTracker = VelocityTracker.obtain();
        velocityTracker.addMovement(event);
- 计算速度，获得水平速度和竖直速度
        velocityTracker.computeCurrentVelocity(1000);
        int xVelocity = (int)velocityTracker.getXVelocity();
        int yVelocity = (int)velocityTracker.getYVelocity();
注意，获取速度之前必须先计算速度，即调用computeCurrentVelocity方法，这里指的速度是指一段时间内手指滑过的像素数，1000指的是1000毫秒，得到的是1000毫秒内滑过的像素数。速度可正可负：速度 = （ 终点位置 - 起点位置） / 时间段
- 最后，当不需要使用的时候，需要调用clear()方法重置并回收内存：
		velocityTracker.clear();
		velocityTracker.recycle();

**GestureDetector**
手势检测，用于辅助检测用户的单击、滑动、长按、双击等行为。使用方法：
- 创建一个GestureDetector对象并实现OnGestureListener接口，根据需要，也可实现OnDoubleTapListener接口从而监听双击行为：
        GestureDetector mGestureDetector = new GestureDetector(this);
        //解决长按屏幕后无法拖动的现象
        mGestureDetector.setIsLongpressEnabled(false);
- 在目标View的OnTouchEvent方法中添加以下实现：
        boolean consume = mGestureDetector.onTouchEvent(event);
        return consume;
- 实现OnGestureListener和OnDoubleTapListener接口中的方法
![](http://img.blog.csdn.net/20160322205838021)
其中常用的方法有：onSingleTapUp(单击)、onFling(快速滑动)、onScroll(拖动)、onLongPress(长按)和onDoubleTap（ 双击）。建议：如果只是监听滑动相关的，可以自己在onTouchEvent中实现，如果要监听双击这种行为，那么就使用GestureDetector。

**Scroller**
弹性滑动对象，用于实现View的弹性滑动。其本身无法让View弹性滑动，需要和View的computeScroll方法配合使用才能完成这个功能。使用方法：

    Scroller scroller = new Scroller(mContext);
    //缓慢移动到指定位置
    private void smoothScrollTo(int destX,int destY){
        int scrollX = getScrollX();
        int delta = destX - scrollX;
        //1000ms内滑向destX,效果就是慢慢滑动
        mScroller.startScroll(scrollX,0,delta,0,1000);
        invalidata();
    } 
    @Override
    public void computeScroll(){
        if(mScroller.computeScrollOffset()){
        scrollTo(mScroller.getCurrX,mScroller.getCurrY());
        postInvalidate();
        }
    }


#### View的滑动
三种方式实现View滑动
##### 使用scrollTo/scrollBy
scrollBy实际调用了scrollTo，它实现了基于当前位置的相对滑动，而scrollTo则实现了绝对滑动。

==scrollTo和scrollBy只能改变View的内容位置而不能改变View在布局中的位置。滑动偏移量mScrollX和mScrollY的正负与实际滑动方向相反，即从左向右滑动，mScrollX为负值，从上往下滑动mScrollY为负值。==

##### 使用动画
使用动画移动View，主要是操作View的translationX和translationY属性，既可以采用传统的View动画，也可以采用属性动画，如果使用属性动画，为了能够兼容3.0以下的版本，需要采用开源动画库nineolddandroids。 如使用属性动画：(View在100ms内向右移动100像素)

		ObjectAnimator.ofFloat(targetView,"translationX",0,100).setDuration(100).start();
	
##### 改变布局属性
通过改变布局属性来移动View，即改变LayoutParams。

###3.2.4 各种滑动方式的对比
- scrollTo/scrollBy：操作简单，适合对View内容的滑动；
- 动画：操作简单，主要适用于没有交互的View和实现复杂的动画效果；
- 改变布局参数：操作稍微复杂，适用于有交互的View。

#### 弹性滑动

##### 使用Scroller
使用Scroller实现弹性滑动的典型使用方法如下：

    Scroller scroller = new Scroller(mContext);
    //缓慢移动到指定位置
    private void smoothScrollTo(int destX,int dextY){
        int scrollX = getScrollX();
        int deltaX = destX - scrollX;
        //1000ms内滑向destX，效果就是缓慢滑动
        mScroller.startSscroll(scrollX,0,deltaX,0,1000);
        invalidate();
    } 
    @override
    public void computeScroll(){
        if(mScroller.computeScrollOffset()){
        scrollTo(mScroller.getCurrX(),mScroller.getCurrY());
        postInvalidate();
        }
    }
从上面代码可以知道，我们首先会构造一个Scroller对象，并调用他的startScroll方法，该方法并没有让view实现滑动，只是把参数保存下来，我们来看看startScroll方法的实现就知道了：

    public void startScroll(int startX,int startY,int dx,int dy,int duration){
        mMode = SCROLL_MODE;
        mFinished = false;
        mDuration = duration;
        mStartTime = AnimationUtils.currentAminationTimeMills();
        mStartX = startX;
        mStartY = startY;
        mFinalX = startX + dx;
        mFinalY = startY + dy;
        mDeltaX = dx;
        mDeltaY = dy;
        mDurationReciprocal = 1.0f / (float)mDuration;
    }
可以知道，startScroll方法的几个参数的含义，startX和startY表示滑动的起点，dx和dy表示的是滑动的距离，而duration表示的是滑动时间，注意，这里的滑动指的是View内容的滑动，在startScroll方法被调用后，马上调用invalidate方法，这是滑动的开始，invalidate方法会导致View的重绘，在View的draw方法中调用computeScroll方法，computeScroll又会去向Scroller获取当前的scrollX和scrollY；然后通过scrollTo方法实现滑动，接着又调用postInvalidate方法进行第二次重绘，一直循环，直到computeScrollOffset()方法返回值为false才结束整个滑动过程。 我们可以看看computeScrollOffset方法是如何获得当前的scrollX和scrollY的：

    public boolean computeScrollOffset(){
        ...
        int timePassed = (int)(AnimationUtils.currentAnimationTimeMills() - mStartTime);
        if(timePassed < mDuration){
            switch(mMode){
            case SCROLL_MODE:
            final float x = mInterpolator.getInterpolation(timePassed * mDuratio
            nReciprocal);
            mCurrX = mStartX + Math.round(x * mDeltaX);
            mCurrY = mStartY + Math.round(y * mDeltaY);
            break;
            ...
            }
        } 
        return true;
    }
到这里我们就基本明白了，computeScroll向Scroller获取当前的scrollX和scrollY其实是通过计算时间流逝的百分比来获得的，每一次重绘距滑动起始时间会有一个时间间距，通过这个时间间距Scroller就可以得到View当前的滑动位置，然后就可以通过scrollTo方法来完成View的滑动了。

##### 通过动画
动画本身就是一种渐近的过程，因此通过动画来实现的滑动本身就具有弹性。实现也很简单：

    ObjectAnimator.ofFloat(targetView,"translationX",0,100).setDuration(100).start()
    ;    
    //当然，我们也可以利用动画来模仿Scroller实现View弹性滑动的过程：
    final int startX = 0;
    final int deltaX = 100;
    ValueAnimator animator = ValueAnimator.ofInt(0,1).setDuration(1000);
    animator.addUpdateListener(new AnimatorUpdateListener(){
        @override
        public void onAnimationUpdate(ValueAnimator animator){
        float fraction = animator.getAnimatedFraction();
        mButton1.scrollTo(startX + (int) (deltaX * fraction) , 0);
        }
    });
    animator.start();
上面的动画本质上是没有作用于任何对象上的，他只是在1000ms内完成了整个动画过程，利用这个特性，我们就可以在动画的每一帧到来时获取动画完成的比例，根据比例计算出View所滑动的距离。采用这种方法也可以实现其他动画效果，我们可以在onAnimationUpdate方法中加入自定义操作。

##### 使用延时策略
延时策略的核心思想是通过发送一系列延时信息从而达到一种渐近式的效果，具体可以通过Hander和View的postDelayed方法，也可以使用线程的sleep方法。 下面以Handler为例：

    private static final int MESSAGE_SCROLL_TO = 1;
    private static final int FRAME_COUNT = 30;
    private static final int DELATED_TIME = 33;
    private int mCount = 0;
    @suppressLint("HandlerLeak")
    private Handler handler = new handler(){
        public void handleMessage(Message msg){
        switch(msg.what){
            case MESSAGE_SCROLL_TO:
            mCount ++ ;
            if (mCount <= FRAME_COUNT){
                float fraction = mCount / (float) FRAME_COUNT;
                int scrollX = (int) (fraction * 100);
                mButton1.scrollTo(scrollX,0);
                mHandelr.sendEmptyMessageDelayed(MESSAGE_SCROLL_TO , DELAYED_TIME);
                } 
            break;
            default : break;
            }
        }
    }