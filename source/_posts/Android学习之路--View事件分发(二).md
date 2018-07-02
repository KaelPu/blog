---
title: Android学习之路 -- View事件分发(二)
date: 2017-05-015 19:22:13
tags: Android学习之路
---

### View的事件分发机制

#### 点击事件的传递规则
点击事件是MotionEvent。首先我们先看看下面一段伪代码，通过它我们可以理解到点击事件的传递规则：

    public boolean dispatchTouchEvent (MotionEvent ev){
    boolean consume = false;
    if (onInterceptTouchEvnet(ev){
    	consume = onTouchEvent(ev);
    } else {
    	consume = child.dispatchTouchEnvet(ev);
    } 
    return consume;
    }
上面代码主要涉及到以下三个方法：
- public boolean dispatchTouchEvent(MotionEvent ev); 
这个方法用来进行事件的分发。如果事件传递给当前view，则调用此方法。返回结果表示是否消耗此事件，受onTouchEvent和下级View的dispatchTouchEvent方法影响。
- public boolean onInterceptTouchEvent(MotionEvent ev); 
这个方法用来判断是否拦截事件。在dispatchTouchEvent方法中调用。返回结果表示是否拦截。
- public boolean onTouchEvent(MotionEvent ev); 
这个方法用来处理点击事件。在dispatchTouchEvent方法中调用，返回结果表示是否消耗事件。如果不消耗，则在同一个事件序列中，当前View无法再次接收到事件。

![image.png](https://upload-images.jianshu.io/upload_images/1967257-06264ace4007d4b6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
点击事件的传递规则：对于一个根ViewGroup，点击事件产生后，首先会传递给他，这时候就会调用他的dispatchTouchEvent方法，如果Viewgroup的onInterceptTouchEvent方法返回true表示他要拦截事件，接下来事件就会交给ViewGroup处理，调用ViewGroup的onTouchEvent方法；如果ViewGroup的onInteceptTouchEvent方法返回值为false，表示ViewGroup不拦截该事件，这时事件就传递给他的子View，接下来子View的dispatchTouchEvent方法，如此反复直到事件被最终处理。

当一个View需要处理事件时，如果它设置了OnTouchListener，那么onTouch方法会被调用，如果onTouch返回false，则当前View的onTouchEvent方法会被调用，返回true则不会被调用，同时，在onTouchEvent方法中如果设置了OnClickListener，那么他的onClick方法会被调用。==由此可见处理事件时的优先级关系： onTouchListener > onTouchEvent >onClickListener==

关于事件传递的机制，这里给出一些结论：
1. 一个事件系列以down事件开始，中间包含数量不定的move事件，最终以up事件结束。
2. 正常情况下，一个事件序列只能由一个View拦截并消耗。
3. 某个View拦截了事件后，该事件序列只能由它去处理，并且它的onInterceptTouchEvent
不会再被调用。
4. 某个View一旦开始处理事件，如果它不消耗ACTION_DOWN事件（ onTouchEvnet返回false） ，那么同一事件序列中的其他事件都不会交给他处理，并且事件将重新交由他的父元素去处理，即父元素的onTouchEvent被调用。
5. 如果View不消耗ACTION_DOWN以外的其他事件，那么这个事件将会消失，此时父元素的onTouchEvent并不会被调用，并且当前View可以持续收到后续的事件，最终消失的点击事件会传递给Activity去处理。
6. ViewGroup默认不拦截任何事件。
7. View没有onInterceptTouchEvent方法，一旦事件传递给它，它的onTouchEvent方法会被调用。
8. View的onTouchEvent默认消耗事件，除非他是不可点击的（ clickable和longClickable同时为false） 。View的longClickable属性默认false，clickable默认属性分情况（如TextView为false，button为true）。
9. View的enable属性不影响onTouchEvent的默认返回值。
10. onClick会发生的前提是当前View是可点击的，并且收到了down和up事件。
11. 事件传递过程总是由外向内的，即事件总是先传递给父元素，然后由父元素分发给子View，通过requestDisallowInterceptTouchEvent方法可以在子元素中干预父元素的分发过程，但是ACTION_DOWN事件除外。

##### 事件分发的源码解析
略
#### 滑动冲突
在界面中，只要内外两层同时可以滑动，这个时候就会产生滑动冲突。滑动冲突的解决有固定的方法。

##### 常见的滑动冲突场景
![image.png](https://upload-images.jianshu.io/upload_images/1967257-12ace9e2f05b5fb9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1. 外部滑动和内部滑动方向不一致；
比如viewpager和listview嵌套，但这种情况下viewpager自身已经对滑动冲突进行了处理。
2. 外部滑动方向和内部滑动方向一致；
3. 上面两种情况的嵌套。
只要解决1和2即可。

##### 滑动冲突的处理规则
对于场景一，处理的规则是：当用户左右（ 上下） 滑动时，需要让外部的View拦截点击事件，当用户上下（ 左右） 滑动的时候，需要让内部的View拦截点击事件。根据滑动的方向判断谁来拦截事件。

对于场景二，由于滑动方向一致，这时候只能在业务上找到突破点，根据业务需求，规定什么时候让外部View拦截事件，什么时候由内部View拦截事件。

场景三的情况相对比较复杂，同样根据需求在业务上找到突破点。

##### 滑动冲突的解决方式
**外部拦截法**
所谓外部拦截法是指点击事件都先经过父容器的拦截处理，如果父容器需要此事件就拦截，否则就不拦截。下面是伪代码：

    public boolean onInterceptTouchEvent (MotionEvent event){
    boolean intercepted = false;
    int x = (int) event.getX();
    int y = (int) event.getY();
    switch (event.getAction()) {
    case MotionEvent.ACTION_DOWN:
        intercepted = false;
        break;
    case MotionEvent.ACTION_MOVE:
        if (父容器需要当前事件） {
        intercepted = true;
        } else {
        intercepted = flase;
        } 
        break;
        } 
    case MotionEvent.ACTION_UP:
        intercepted = false;
        break;
    default : 
    	break;
    } 
    mLastXIntercept = x;
    mLastYIntercept = y;
    return intercepted;
针对不同冲突，只需修改父容器需要当前事件的条件即可。其他不需修改也不能修改。
- ACTION_DOWN:必须返回false。因为如果返回true，后续事件都会被拦截，无法传递给子View。
- ACTION_MOVE：根据需要决定是否拦截
- ACTION_UP：必须返回false。如果拦截，那么子View无法接受up事件，无法完成click操作。而如果是父容器需要该事件，那么在ACTION_MOVE时已经进行了拦截，根据上一节的结论3，ACTION_UP不会经过onInterceptTouchEvent方法，直接交给父容器处理。

**内部拦截法** 
内部拦截法是指父容器不拦截任何事件，所有的事件都传递给子元素，如果子元素需要此事件就直接消耗，否则就交由父容器进行处理。这种方法与Android事件分发机制不一致，需要配合requestDisallowInterceptTouchEvent方法才能正常工作。下面是伪代码：

    public boolean dispatchTouchEvent ( MotionEvent event ) {
    int x = (int) event.getX();
    int y = (int) event.getY();
    switch (event.getAction) {
    case MotionEvent.ACTION_DOWN:
        parent.requestDisallowInterceptTouchEvent(true);
        break;
    case MotionEvent.ACTION_MOVE:
        int deltaX = x - mLastX;
        int deltaY = y - mLastY;
        if (父容器需要此类点击事件) {
        	parent.requestDisallowInterceptTouchEvent(false);
        } 
        break;
    case MotionEvent.ACTION_UP:
        break;
    default : 
    	break;
    } 
    mLastX = x;
    mLastY = y;
    return super.dispatchTouchEvent(event);
    }
==除了子元素需要做处理外，父元素也要默认拦截除了ACTION_DOWN以外的其他事件，这样当子元素调用parent.requestDisallowInterceptTouchEvent(false)方法时，父元素才能继续拦截所需的事件。==因此，父元素要做以下修改：

    public boolean onInterceptTouchEvent (MotionEvent event) {
        int action = event.getAction();
        if(action == MotionEvent.ACTION_DOWN) {
            return false;
        } else {
            return true;
        }
    }

优化滑动体验：

	mScroller.abortAnimation();
外部拦截法实例：[HorizontalScrollViewEx](https://github.com/singwhatiwanna/android-art-res/blob/master/Chapter_3/src/com/ryg/chapter_3/ui/HorizontalScrollViewEx.java)