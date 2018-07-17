---
title: 详解Android插件框架 (四)--反射机制
date: 2018-06-28 18:15:30
tags:
toc_number: true
---

## 前言
反射机制是Java的重要特性之一，Android的黑科技源泉，所以熟练掌握了解反射技术的原理是很有必要的。上篇我们说过Java虚拟机通过加载 .jar 文件里的类来执行程序，而我们Android虚拟机Dalvik/ART 确是直接加载的 .dex文件来执行里面的类，dex文件是Android对.jar文件进行优化来的
目的是更加适合Android平台的Dalvik/ART使用

## .dex文件和Apk

一般来说我们可以通过 Java JDK  java代码编译成为.class 文件
然后再使用jar把 .class 文件打包成.jar
然后我们可以使用Android SDK中给我们提供了工具来讲.jar文件优化成.dex文件
```
dx --dex --output=A.dex B.jar 
```
大家可以在“android-sdk\build-tools\版本\”文件中找到
其实我们应用的APK文件主要就是Dex文件 + 资源 + Asset文件 + AndroidManifest文件

![image.png](https://upload-images.jianshu.io/upload_images/1967257-dc8ce34800e9573e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

classes.dex 这个就是Apk的dex文件
所以对于虚拟机来说Apk也等同于一个Dex文件

## 加载dex
这个我们上一篇文章说过了，就是使用我们的Classloader去加载一个dex文件
我们一般会使用DexClassLoader去加载，需要注意的是，我们DexClassLoader先将dex文件拷贝到
应用所在的data/packagename/的私有文件中，这样DexClassLoader才能够有权限加载，
至于原因，主要是问了安全，前面文章也介绍过，这里看一下DexClassLoader的构造就懂了
```
DexClassLoader(String dexPath, String optimizedDirectory, String librarySearchPath, ClassLoader parent);
```
> * dexPath：dex文件所在SD卡路径
> * optimizedDirectory：dex加压路径 一般是App私有目录 data/packagename/xxx路径
> * librarySearchPath：指目标类中所使用的C/C++库存放的路径
> * classload：是指该装载器的父装载器,一般为当前执行类的装载器

## 反射
![image.png](https://upload-images.jianshu.io/upload_images/1967257-ec25ece8b3796953.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
说实话，本来这一篇想好好把我的反射技巧都秀一秀，但是仔细想了一下，要秀一秀估计就要再开三篇文章来具体讲反射，害怕把大家带跑偏了，这个系列文章，我们还是关注插件技术吧，反射作为基础工具，我相信很多人也使用过，所以我这里就偷个懒吧。而且我们后面讲插件框架时候，可从源码中学到很多优秀的反射技巧。我们简单来看看，刚才加载的dex我们如何使用里面的函数
```
// 先用Classloader加载dex
DexClassLoader dexClassLoader = new DexClassLoader(optimizedDexOutputPath.getAbsolutePath(), dexOutputDir.getAbsolutePath(), null, getClassLoader());

Class libProviderClazz = null;
try{
	
	// 得到Dex中某个类
    libProviderClazz=dexClassLoader.loadClass("com.karlpu.MyClass");
    
    // 遍历类里所有方法
    Method[]methods=libProviderClazz.getDeclaredMethods();
    for(int i=0;i<methods.length;i++){
    Log.e(TAG,methods[i].toString());
    }
    
    // 获取方法
    Method start=libProviderClazz.getDeclaredMethod("func");

	// 把方法设为public，让外部可以调用
    start.setAccessible(true);

	// 调用方法并获取返回值
    String string=(String)start.invoke(libProviderClazz.newInstance());

    }catch(Exception exception){
	    exception.printStackTrace();
    }
```

这样我们就调用了dex文件中的一个方法。

## 结语
到这里我们应该知道了，其实Android可以动态执行代码，只要我们在合适的时候讲Dex加载进Classloader然后就可以通过反射去调用相应的方法。理论我们可以将一个APK加载进我们自己的APK中然后通过反射调用它的方法。这其实就是个插件了。
但是问题依然存在：
> * 如果插件的dex文件中的代码用了资源，我们依然无法调用该代码
> * 如果插件的dex文件中使用了四大组件，我们也依然无法合法启动组件

具体原因我在之前文章说过，就不在赘述，其实这两个问题，就是我们插件框架主要要解决的问题，后面我们来一个一个搞定，下篇文章，我们需要温习一下动态代理的知识，因为动态代理模式
和反射的配合使得我们能够使用Hook技术来解决上面两个问题。

《详解Android插件框架 (五) --- 动态代理》

Ps：才疏学浅有问题请指出来~如需转载文章请注明 by karlpu 谢谢!