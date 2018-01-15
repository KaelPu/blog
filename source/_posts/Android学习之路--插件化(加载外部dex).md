---
title: Android学习之路 -- 插件化(加载外部dex)
date: 2017-10-28 18:11:47
tags: Android学习之路
---

Java程序中，JVM虚拟机是通过类加载器ClassLoader加载.jar文件里面的类的。Android也类似，不过Android用的是Dalvik/ART虚拟机，不是JVM，也不能直接加载.jar文件，而是加载dex文件。

先要通过Android SDK提供的DX工具把.jar文件优化成.dex文件，然后Android的虚拟机才能加载。注意，有的Android应用能直接加载.jar文件，那是因为这个.jar文件已经经过优化，只不过后缀名没改（其实已经是.dex文件）。

#### jar文件优化成.dex文件
首先我们可以通过JDK的编译命令javac把Java代码编译成.class文件，再使用jar命令把.class文件封装成.jar文件，这与编译普通Java程序的时候完全一样。 
之后再用Android SDK的DX工具把.jar文件优化成.dex文件（在“android-sdk\build-tools\具体版本\”路径下）
```
dx --dex --output=target.dex origin.jar // target.dex就是我们要的了
```

此外，我们可以现把代码编译成APK文件，再把APK里面的.dex文件解压出来，或者直接把APK文件当成.dex使用（只是APK里面的静态资源文件我们暂时还用不到）。至此我们发现，无论加载.jar，还是.apk，其实都和加载.dex是等价的，Android能加载.jar和.apk，是因为它们都包含有.dex，直接加载.apk文件时，ClassLoader也会自动把.apk里的.dex解压出来。

#### 加载并调用.dex里面的方法
与JVM不同，Android的虚拟机不能用ClassCload直接加载.dex，而是要用DexClassLoader或者PathClassLoader,他们都是ClassLoader的子类，这两者的区别是 
1) DexClassLoader：可以加载jar/apk/dex，可以从SD卡中加载未安装的apk； 
2) PathClassLoader：要传入系统中apk的存放Path，所以只能加载已经安装的apk文件； 
使用前，先看看DexClassLoader的构造方法
```
 public DexClassLoader(String dexPath, String optimizedDirectory, String libraryPath, ClassLoader parent) {
    super((String) null, (File) null, (String) null, (ClassLoader) null);
    throw new RuntimeException("Stub!");
}

```
注意，我们之前提到的，DexClassLoader并不能直接加载外部存储的.dex文件，而是要先拷贝到内部存储里。这里的dexPath就是.dex的外部存储路径，而optimizedDirectory则是内部路径，libraryPath用null即可，parent则是要传入当前应用的ClassLoader，这与ClassLoader的“双亲代理模式”有关。
```
File optimizedDexOutputPath = new File(Environment.getExternalStorageDirectory().getAbsolutePath() + File.separator + "test_dexloader.jar");// 外部路径
File dexOutputDir = this.getDir("dex", 0);// 无法直接从外部路径加载.dex文件，需要指定APP内部路径作为缓存目录（.dex文件会被解压到此目录）
DexClassLoader dexClassLoader = new DexClassLoader(optimizedDexOutputPath.getAbsolutePath(), dexOutputDir.getAbsolutePath(), null, getClassLoader());
```
到这里，我们已经成功把.dex文件给加载进来了，接下来就是如何调用.dex里面的代码，主要有两种方式。

#### 如何调用.dex里面的代码
使用反射的方式

使用DexClassLoader加载进来的类，我们本地并没有这些类的源码，所以无法直接调用，不过可以通过反射的方法调用，简单粗暴。
```
DexClassLoader dexClassLoader = new DexClassLoader(optimizedDexOutputPath.getAbsolutePath(), dexOutputDir.getAbsolutePath(), null, getClassLoader());
Class libProviderClazz = null;
try{
    libProviderClazz=dexClassLoader.loadClass("com.dexclassloader.MyLoader");
    // 遍历类里所有方法
    Method[]methods=libProviderClazz.getDeclaredMethods();
    for(int i=0;i<methods.length;i++){
    Log.e(TAG,methods[i].toString());
    }
    Method start=libProviderClazz.getDeclaredMethod("func");// 获取方法
    start.setAccessible(true);// 把方法设为public，让外部可以调用
    String string=(String)start.invoke(libProviderClazz.newInstance());// 调用方法并获取返回值
    Toast.makeText(this,string,Toast.LENGTH_LONG).show();
    }catch(Exception exception){
    // Handle exception gracefully here.
    exception.printStackTrace();
    }
```
使用接口的方式 
毕竟.dex文件也是我们自己维护的，所以可以把方法抽象成公共接口，把这些接口也复制到主项目里面去，就可以通过这些接口调用动态加载得到的实例的方法了。
```
pulic interface IFunc{
    public String func();
}

// 调用
IFunc ifunc = (IFunc)libProviderClazz;
String string = ifunc.func();
Toast.makeText(this, string, Toast.LENGTH_LONG).show();
```
到这里，我们已经成功从外部路径动态加载一个.dex文件，并执行里面的代码逻辑了。通过从服务器下载最新的.dex文件并替换本地的旧文件，就能初步实现“APP的动态升级了”。

虽然我们已经能调用插件的方法了，但是还有如下问题

无法使用res目录下的资源，特别是使用XML布局，以及无法通过res资源到达自适应
无法动态加载新的Activity等组件，因为这些组件需要在Manifest中注册，动态加载无法更改当前APK的Manifest