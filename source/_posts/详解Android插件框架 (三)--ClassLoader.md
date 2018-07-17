---
title: 详解Android插件框架 (三)--ClassLoader
date: 2018-06-27 20:15:20
tags:
toc_number: true
---

## 前言
我们写的Java文件最后都会变成一个个Class文件，而然Java的JVM虚拟机会加载并且执行这些Class文件。在Android中也是如此，虽然Android的Dalvik/ART虚拟机和JVM有很多的不同，但是他们的工作原理都是类似的。但是可以说的是Android虚拟机加载Class的过程比JVM要稍微复杂些。

## ClassLoader实例
虚拟机管理加载Class文件的工具就是ClassLoader，在Android系统中一个App最少有两个ClassLoader实例，一个叫做BootClassLoader， 一个就做PathClassLoader，我们可以随便创建一个App工程，通过getClassLoader()函数来打印出来所有ClassLoader对象，这个函数在任意Activity中调用即可。
```
public void printClassLoader() {
    ClassLoader classLoader = getClassLoader();
    if (classLoader != null) 
	    Log.i(TAG, classLoader.toString());
        while (classLoader.getParent() != null) {
            classLoader = classLoader.getParent();
            Log.i(TAG, classLoader.toString());
        }
    }
}
```
> *  dalvik.system.PathClassLoader
> *  java.lang.BootClassLoader@14af4e32

可以看到结果确实是两个ClassLoader对象

BootClassLoader对象是系统创建的，用来加载Android Framwork的Class文件，也就是你所用的Activity，View 等等系统的类，这些类都是由BootClassLoader 来管理加载的

PathClassLoader对象是应用创建时候产生了，负责管理加载你APP中的类，也就是你写的所有代码生成的Class文件

但是我们仔细看下上面的代码会发现，BootClassLoader是由PathClassLoader对象调用getParent()得到的，所以我们猜测 BootClassLoader 是 PathClassLoader的父类？其实并不是，**这里是一种双亲代理模型（Parent-Delegation Model）的设计模式**，简单来说就是如果你想创建一个ClassLoader对象，你必须要以一个ClassLoader对象作为基础来创建，我们能不能自己创建ClassLoader呢？答案是能！我们看一下ClassLoader的构造函数
```
    protected ClassLoader() {
        this(getSystemClassLoader(), false);
    }
    protected ClassLoader(ClassLoader parentLoader) {
        this(parentLoader, false);
    }
```
很明显了，创建一个ClassLoader对象一定需要一个parent，如果你不传parent，那么系统就会使用PathClassLoader对象当做Parent,对这里确实是用PathClassLoader当parent不是BootClassLoader这个哦！

## 如何加载
ClassLoader有一个loadClass方法，就是用来加载Class文件的，我们简单看下源码，不难
```
protected Class<?> loadClass(String className, boolean resolve){
	// 在自己缓存中，看看是否加载过？
    Class<?> clazz = findLoadedClass(className);
    if (clazz == null) {
        ClassNotFoundException suppressed = null;
        try {
	        // 自己缓存没有，问问parent缓存有没有
            clazz = parent.loadClass(className, false);
        } catch (ClassNotFoundException e) {
            suppressed = e;
        }
        if (clazz == null) {
            try {
            // parent 缓存和未加载列表中都没有，在自己的未加载列表中找 
                clazz = findClass(className);
            } catch (ClassNotFoundException e) {
                e.addSuppressed(suppressed);
                throw e;
            }
        }
    }
    return clazz;
  }
```

源码不难，建议可以仔细看看，不过实在不愿意看就听我说吧
从源码中我们可以看出，ClassLoader查找类的时候按照如下顺序
1、先在自己的加载缓存中找Class 没找到就委托parent找
2、parent在自己缓存找，没找到
3、如果已经是最顶层Classloader 会开始在自己未加载列表里找
4、如果找到了 就返回，如果没找到 让子类自己在未加载列表里找
5、子类如果在未加载列表里找到了就返回，没有找到就 not find class 异常

总结来说 先往上找缓存，大家缓存都没有，就再由上到下找自己未加载列表里有没有
有就返回，大家都没有就报异常

这样做有个明显的特点，如果一个类被位于树根的ClassLoader加载过，那么在以后整个系统的生命周期内，这个类永远不会被重新加载。

## DexClassLoader
这个类是我们实际会用到的ClassLoader类，下面是ClassLoader的类图

![image.png](https://upload-images.jianshu.io/upload_images/1967257-adea86005990b21f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到由ClassLoader类派生出很多类，BootClassLoader和PathClassLoader这两个我们见过了。而最下面的 DexClassLoader 就是我们平常会用的。
简单说一下他们的区别吧：
> * BootClassLoader: 系统创建，我们一般无法直接使用
> * PathClassLoader:只能加载Apk中自身带的Dex文件（Class）
> * DexClassLoader：可以加载SD卡中（先拷贝到data）的 APK DEX JAR 文件 *

这里需要说一下 其实我们之前一直说加载Class文件，是为了方便大家理解，但是这样的表述是不准确的，因为其实Android的Dalvik/ART虚拟机认的是Dex文件，DexClassLoader 加载APK、DEX、JAR文件本质都是在加载DEX文件。

通过上面的区别，大家可以看到，虽然大家的腰椎都很突出，但是DexClassLoader同学的腰椎特别的突出，啥都能加载，还不分地方，所以DexClassLoader就成了我们最喜欢使用的ClassLoader啦

这部分的源码 我就不带大家看了，类图很清楚 代码也比较容易，有兴趣可以自己看看，只要大家知道原理就行了。

## ClassLoader需要注意得地方
先问一个问题，包名一样的两个对象，一定是同一个Class么？比如说
有个A类 我new了一个A类的对象 a，用这个a instanceof A 一定会返回 true么？
```
 A a = new A()
if(a instanceof A){
	// 一定会是true么？
}
```
其实不一定哦，原因如下
**同一个Class = 相同的 ClassName + 相同的PackageName + 相同的ClassLoader**
也就是说一个Class如果相同 必须确保这个Class名字 包名 还有加载它的ClassLoader都必须相同
也就是说如果我们自己new一个ClassLoader 加载了A类 然后和另一个ClassLoader加载的A类的对象做instanceof 判断，会返回false哦
那有人会疑惑，上面不是说双亲委派就是为了避免重复加载，而且所有ClassLoader由是树形关系，怎么会两个Classloader同时加载一个Class呢？

我们看看下面这个类关系图
![image.png](https://upload-images.jianshu.io/upload_images/1967257-aade99bda9cffd13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

MyclassLoader_1 和 MyClassLodaer_2 是我创建的两个Classloader对象
他们都是继承PathClassloader的 但是他们彼此间没有双亲委派关系
所以他们是能够同时加载Class A 的
但是他们两个的Class A 按照规则 就不是同一个Class

虽然我们在正常开发中，基本不会遇到这种问题，因为我们正常开发很少自己创建ClassLoader
但是在我们插件框架中，是会有几个Classloader存在的，在这种情况下，可能就会遇到这种坑。
这个后面我们会在讲插件框架时候再给大家举例

## 结语
本篇文章主要带大家熟悉一下 ClassLoader 以及它的基本原理，因为它是我们插件化技术的基石，
我们后面华丽的操作都要基于它。
好了下篇文章我们会和大家分享一下插件化第二位功臣 反射机制
[《详解Android插件框架 (四) -- 反射机制》