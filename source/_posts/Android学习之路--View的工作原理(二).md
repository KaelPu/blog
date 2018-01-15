---
title: Android学习之路 -- View的工作原理(二)
date: 2017-06-08 21:15:32
tags: Android学习之路
---
### View的工作原理
#### 自定义View

##### 自定义View的分类
**继承View 重写onDraw方法**
通过 onDraw 方法来实现一些不规则的效果，这种效果不方便通过布局的组合方式来达到。这种方式需要自己支持 wrap_content ，并且padding也要去进行处理。

**继承ViewGroup派生特殊的layout**
实现自定义的布局方式，需要合适地处理ViewGroup的测量、布局这两个过程，并同时处理子View的测量和布局过程。

**继承特定的View子类（ 如TextView、Button）**
扩展某种已有的控件的功能，比较简单，不需要自己去管理 wrap_content 和padding。

** 继承特定的ViewGroup子类（ 如LinearLayout）**
比较常见，实现几种view组合一起的效果。与方法二的差别是方法二更接近底层实现。

##### 自定义View须知
1. 直接继承View或ViewGroup的控件， 需要在onmeasure中对wrap_content做特殊处理。指定wrap_content模式下的默认宽/高。
2. 直接继承View的控件，如果不在draw方法中处理padding，那么padding属性就无法起作用。直接继承ViewGroup的控件也需要在onMeasure和onLayout中考虑padding和子元素margin的影响，不然padding和子元素的margin无效。
3. 尽量不要用在View中使用Handler，因为没必要。View内部提供了post系列的方法，完全可以替代Handler的作用。
4. View中有线程和动画，需要在View的onDetachedFromWindow中停止。当View不可见时，也需要停止线程和动画，否则可能造成内存泄漏。
5. View带有滑动嵌套情形时，需要处理好滑动冲突

##### 自定义View实例
- 继承View重写onDraw方法：[CircleView](https://github.com/singwhatiwanna/android-art-res/blob/master/Chapter_4/src/com/ryg/chapter_4/ui/CircleView.java)

>自定义属性设置方法：
1. 在values目录下创建自定义属性的XML，如attrs.xml。
		<?xml version="1.0" encoding="utf-8"?>
        <resources>
            <declare-styleable name="CircleView">
                <attr name="circle_color" format="color" />
            </declare-styleable>
        </resources>
2. 在View的构造方法中解析自定义属性的值并做相应处理，这里我们解析circle_color。
		public CircleView(Context context, AttributeSet attrs, int defStyleAttr) {
            super(context, attrs, defStyleAttr);
            TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.CircleView);
            mColor = a.getColor(R.styleable.CircleView_circle_color, Color.RED);
            a.recycle();
            init();
    	}
3. 在布局文件中使用自定义属性
        <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:app="http://schemas.android.com/apk/res-auto"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:background="#ffffff"
        android:orientation="vertical" >
        <com.ryg.chapter_4.ui.CircleView
            android:id="@+id/circleView1"
            android:layout_width="wrap_content"
            android:layout_height="100dp"
            android:layout_margin="20dp"
            android:background="#000000"
            android:padding="20dp"
            app:circle_color="@color/light_green" />
    	</LinearLayout>
[Android中的命名空间](http://blog.qiji.tech/archives/3744)

- 继承ViewGroup派生特殊的layout：[HorizontalScrollViewEx](https://github.com/singwhatiwanna/android-art-res/blob/master/Chapter_4/src/com/ryg/chapter_4/ui/HorizontalScrollViewEx.java)
onMeasure方法中，首先判断是否有子元素，没有的话根据LayoutParams中的宽高做相应处理。然后判断宽高是不是wrap_content，如果宽是，那么HorizontalScrollViewEx的宽就是所有所有子元素的宽度之和。如果高是wrap_content，HorizontalScrollViewEx的高度就是第一个子元素的高度。同时要处理padding和margin。
onLayout方法中，在放置子元素时候也要考虑padding和margin。

##### 自定义View的思想
- 掌握基本功，比如View的弹性滑动、滑动冲突、绘制原理等
- 面对新的自定义View时，对其分类并选择合适的实现思路。
