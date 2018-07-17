---
title: 详解Android插件框架 (五)--Hook与代理模式
date: 2018-06-29 18:25:40
tags:
toc_number: true
---


## 前言
Hook在中文意思就是"钩子",怎么理解什么是钩子呢？
这幅图应该比较形象，当我们执行一段函或者代码片段的过程中，我们可以在合适的位置设置一个钩子，让代码先从我们这里走，然后再回到它原来的轨道上。听上去有点拦截的意思，所以Hook天生就有点黑科技的味道。那么在Android中我们该如何实现Hook呢？
![image.png](https://upload-images.jianshu.io/upload_images/1967257-8fc081d5418fdf77.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 代理模式
Hook的关键就是运用了设计模式中的代理模式，下图是代理模式的类图，主要思想和上面的比喻一样，代理类和委托类都实现同一个接口，然后通过操作代理类，来间接操作委托类，为啥要这么麻烦呢，这就是我们刚说的，代理类其实在这里起到了拦截的作用，因为他和委托类实现同同一接口，在外部看来我操作代理类，还是操作委托类都是一样的。
![image.png](https://upload-images.jianshu.io/upload_images/1967257-66d52060504012e3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在Android中常见的代理分两种，**静态代理和动态代理**
这两种方法代理方式有不同的特点，适用于不同场景，但是思想都是类似的。下面我们分别来介绍。

### 无代理
首先我们先举个例子，在没有代理模式下的情况，由于比较简单，就直接上代码了，大家一看就明白

① 创建一个Human接口
```
public interface Human {
    void eat();
}
```
接口中只有一个eat方法，代表人吃东西

② 继承Human实现一个Human类
```
public class HumanImpl implements Human {
    @Override
    public void eat() {
        Log.d("HumanImpl", "Eat");
    }
}
```
实现eat方法 里面只打印一个Eat

③ 实例化对象，调用eat方法
```
public static void noProxy() {
        Human human = new HumanImpl();
        human.eat();
    }
```
一切正常，但是我们现在想拦截eat这个方法，然后再里面加一行打印信息该怎么做呢，有人说直接改一下HumanImpl 类中的eat方法不就行了，那么如果HumanImpl 文件是个库文件class，你不能修改呢？
这时候我们就可以考使用代理模式解决。

### 静态代理
还是上面的例子，我们在不修改HumanImpl 这个类的情况下，使用静态代理的方法，在eat函数加个打印信息。

① 实现Human 接口的一个代理类
```
public class HumanProxy implements Human {
    @Override
    public void eat(String something) {
        HumanImpl human = new HumanImpl();
       // 在调用eat之前，先打印一行log
		Log.d("HumanImpl", "Eat before");
        human.eat();
    }
}
```

②  实例化对象，调用eat方法
```
public static void proxy() {
        Human human = new HumanProxy();
        human.eat();
    }
```
这样我们在无法修改HumanImpl类的情况下，在eat方法执行前后打印了新的日志。
这只是个简单的例子。下面我们看看那什么是动态代理。

### 动态代理
动态代理和静态代理有什么区别呢？简单来说我们使用静态代理，需要现在代码中写好代理类，而写一个接口就需要写一个代理类，比如Human接口的代理类是HumanProxy，那么如果我们有多个接口需要使用代理模式呢？我们就需要写很多个代理类。有没有一种方法能作为所有接口的代理类呢？有！这就是动态代理。

动态代理是由JDK提供的方法，我们先来看看具体使用，然后再来分析下原理，还是上面的例子，我们使用动态代理的方式在eat函数之前打印一行日志

① 创建CommonProxy 类实现InvocationHandler接口
```
public class CommonProxy implements InvocationHandler {
    private Object object;

    public DynamicEatProxy(Object T) {
        object = T;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args){
		Log.d("HumanImpl", "Eat before");
        Object returnValue = method.invoke(object, args);
        return returnValue;
    }
}
```
InvocationHandler这个接口是啥 为啥要实现这个接口？其实这个接口就是JDK给我提供的动态代理方案的接口，实现了这个接口，我们就可以创建出一个什么都能代理的代类。

② 实例化对象 调用eat方法
```
 public static void proxy() {
        HumanImpl human = new HumanImpl();
        CommonProxy action = new CommonProxy(human);
        ClassLoader loader = human.getClass().getClassLoader();
        Class[] interfaces = human.getClass().getInterfaces();
        Object proxy = Proxy.newProxyInstance(loader, interfaces, action);
        Human humanProxy = (Human)proxy; //将proxy代理转为Human接口类进行Human接口内方法的调用
        humanProxy.eat();
    }
```
这段看上去有些复杂，其实很简单，来我们慢慢理解，其实关键的就三步

第一步，包装被代理对象，然后把被代理的传进去，注意这个参数是Object 类型哦，所以可以代理任何对象
```
	HumanImpl human = new HumanImpl();
	CommonProxy action = new CommonProxy(human);
```

第二步 创建公共代理对象
```
	ClassLoader loader = human.getClass().getClassLoader();
	Class[] interfaces = human.getClass().getInterfaces();
    Object proxy = Proxy.newProxyInstance(loader, interfaces, action);
```
通过Proxy.newProxyInstance()这个方法，给他传入被代理对象的 ClassLoader  interfaces  还有包装过的被代理对象action  ，这个方法就会返回一个公共代理对象 proxy 其实这玩意就等同于我们静态代理里面那个HumanProxy 对象了。

第三步 实例化对象，调用eat方法
```
    Human humanProxy = (Human)proxy; //将proxy代理转为Human接口类进行Human接口内方法的调用
    humanProxy.eat();
```
转回Human 类型，然后调用eat接口

转了一大圈，是不是有点懵逼？好看个图休息下，

![image.png](https://upload-images.jianshu.io/upload_images/1967257-ae1e51d78309acac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最后总结就是，通过动态代理的方案 我们可以给任意一个对象创建一个公共的代理类，当我们调用，通过这个公共代理类调用的任何方法，就会先走到上面的invoke，也就意味着被拦截了，我们插入一行打印log够，他会继续通过反射的方法去执行委托类中的原本函数。这就实现了拦截代理。

## Hook原理
Hook简单来说就是 代理模式和反射机制的结合，我们在开发中遇到某些API不能满足我们的要求，但是我们又无法修改源码，就可以使用Hook方式处理，Hook的基本方法就是通过找到要修改的API函数入口点，改变它地址指向我们自定义的函数。

一般Hook点我们都会选 静态变量 和 单例对象，因为这些对象一旦创建，他们不容易变化。Hook 过程就是寻找Hook点，尽量Hook public的对象和方法，选择适合的代理模式，结果是接口 动态代理 和 静态代理都可以，如果是对象，我们一般使用静态代理对象替换原来的对象，而替换对象的方式 就是通过反射替换。

## 结束语
本章我们主要理解了 代理模式 包括动态代理 ，静态代理，以及Hook的思想和一些Hook原则，到这里我们基本明白了 Android实现插件化的底层工具和原理，下一篇我们来通过Hook技术简单练个手，加强理解，也作为正式进入讲解插件框架的入门篇。
[《详解Android插件框架 (六) -- 小试牛刀Hook Activity》