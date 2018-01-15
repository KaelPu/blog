---
title: Android学习之路 -- 插件化(资源处理)
date: 2017-11-15 21:51:17
tags: Android学习之路
---
res里的每一个资源都会在R.java里生成一个对应的Integer类型的id，APP启动时会先把R.java注册到当前的上下文环境，我们在代码里以R文件的方式使用资源时正是通过使用这些id访问res资源，然而插件的R.java并没有注册到当前的上下文环境，所以插件的res资源也就无法通过id使用了。
#### 如何使用插件中的R资源
一种解决方式是插件里需要用到的新资源都通过纯Java代码的方式创建（包括XML布局、动画、点九图等），蛋疼但有效。

但这种方式太麻烦了，于是有了以下的解决方式： 
记得我们平时怎么使用res资源的吗，就是“getResources().getXXX(resid)”，看看“getResources()”
```
 @Override
    public Resources getResources() {
        if (mResources != null) {
            return mResources;
        }
        if (mOverrideConfiguration == null) {
            mResources = super.getResources();
            return mResources;
        } else {
            Context resc = createConfigurationContext(mOverrideConfiguration);
            mResources = resc.getResources();
            return mResources;
        }
    }
```
看起来像是通过mResources实例获取res资源的，在找找mResources实例是怎么初始化的，看看上面的代码发现是使用了super类ContextThemeWrapper里的“getResources()”方法，看进去
```
 Context mBase;

    public ContextWrapper(Context base) {
        mBase = base;
    }

    @Override
    public Resources getResources() {
        return mBase.getResources();
    }
```

看样子又调用了Context的“getResources()”方法，看到这里，我们知道Context只是个抽象类，其实际工作都是在ContextImpl完成的，赶紧去ContextImpl里看看“getResources()”方法吧
```
    @Override
    public Resources getResources() {
        return mResources;
    }
```
到这里并没有mResources的创建过程啊！mResources是ContextImpl的成员变量，可能是在构造方法中创建的，于是我们看一下构造方法（这里只给出关键代码）。
```
resources=mResourcesManager.getTopLevelResources(packageInfo.getResDir(),
        packageInfo.getSplitResDirs(),packageInfo.getOverlayDirs(),
        packageInfo.getApplicationInfo().sharedLibraryFiles,displayId,
        overrideConfiguration,compatInfo);
        mResources=resources;
```
看样子是在ResourcesManager的“getTopLevelResources”方法中创建的，看进去
```
 Resources getTopLevelResources(String resDir, String[] splitResDirs,
                                   String[] overlayDirs, String[] libDirs, int displayId,
                                   Configuration overrideConfiguration, CompatibilityInfo compatInfo) {
        Resources r;
        AssetManager assets = new AssetManager();
        if (libDirs != null) {
            for (String libDir : libDirs) {
                if (libDir.endsWith(".apk")) {
                    if (assets.addAssetPath(libDir) == 0) {
                        Log.w(TAG, "Asset path '" + libDir +
                                "' does not exist or contains no resources.");
                    }
                }
            }
        }
        DisplayMetrics dm = getDisplayMetricsLocked(displayId);
        Configuration config;
        ……
        r = new Resources(assets, dm, config, compatInfo);
        return r;
    }
```
看来这里是关键了，看样子就是通过这些代码从一个APK文件加载res资源并创建Resources实例，经过这些逻辑后就可以使用R文件访问资源了。具体过程是，获取一个AssetManager实例，使用其“addAssetPath”方法加载APK（里的资源），再使用DisplayMetrics、Configuration、CompatibilityInfo实例一起创建我们想要的Resources实例。

于是，我们可以通过以下代码加载插件APK里res资源
```
 try {  
        AssetManager assetManager = AssetManager.class.newInstance();  
        Method addAssetPath = assetManager.getClass().getMethod("addAssetPath", String.class);  
        addAssetPath.invoke(assetManager, mDexPath);  
        mAssetManager = assetManager;  
    } catch (Exception e) {  
        e.printStackTrace();  
    }  
    Resources superRes = super.getResources();  
    mResources = new Resources(mAssetManager, superRes.getDisplayMetrics(),  
            superRes.getConfiguration());  
```
注意，有的人担心从插件APK加载进来的res资源的ID可能与主项目里现有的资源ID冲突，其实这种方式加载进来的res资源并不是融入到主项目里面来，主项目里的res资源是保存在ContextImpl里面的Resources实例，整个项目共有，而新加进来的res资源是保存在新创建的Resources实例的，也就是说ProxyActivity其实有两套res资源，并不是把新的res资源和原有的res资源合并了（所以不怕R.id重复），对两个res资源的访问都需要用对应的Resources实例，这也是开发时要处理的问题。（其实应该有3套，Android系统会加载一套framework-res.apk资源，里面存放系统默认Theme等资源）

这里你可能注意到了我们采用了反射的方法调用AssetManager的“addAssetPath”方法，而在上面ResourcesManager中调用AssetManager的“addAssetPath”方法是直接调用的，不用反射啊，而且看看SDK里AssetManager的“addAssetPath”方法的源码（这里也能看到具体APK资源的提取过程是在Native里完成的），发现它也是public类型的，外部可以直接调用，为什么还要用反射呢？
```
 /**
     * Add an additional set of assets to the asset manager.  This can be
     * either a directory or ZIP file.  Not for use by applications.  Returns
     * the cookie of the added asset, or 0 on failure.
     * {@hide}
     */
    public final int addAssetPath(String path) {
        synchronized (this) {
            int res = addAssetPathNative(path);
            makeStringBlocks(mStringBlocks);
            return res;
        }
    }
```
这里有个误区，SDK的源码只是给我们参考用的，APP实际上运行的代码逻辑在android.jar里面（位于android-sdk\platforms\android-XX），反编译android.jar并找到ResourcesManager类就可以发现这些接口都是对应用层隐藏的。
```
public final class AssetManager {
    AssetManager() {
        throw new RuntimeException("Stub!");
    }

    public void close() {
        throw new RuntimeException("Stub!");
    }

    public final InputStream open(String fileName) throws IOException {
        throw new RuntimeException("Stub!");
    }

    public final InputStream open(String fileName, int accessMode) throws IOException {
        throw new RuntimeException("Stub!");
    }

    public final AssetFileDescriptor openFd(String fileName) throws IOException {
        throw new RuntimeException("Stub!");
    }

    public final native String[] list(String paramString) throws IOException;

    public final AssetFileDescriptor openNonAssetFd(String fileName) throws IOException {
        throw new RuntimeException("Stub!");
    }

    public final AssetFileDescriptor openNonAssetFd(int cookie, String fileName) throws IOException {
        throw new RuntimeException("Stub!");
    }

    public final XmlResourceParser openXmlResourceParser(String fileName) throws IOException {
        throw new RuntimeException("Stub!");
    }

    public final XmlResourceParser openXmlResourceParser(int cookie, String fileName) throws IOException {
        throw new RuntimeException("Stub!");
    }

    protected void finalize() throws Throwable {
        throw new RuntimeException("Stub!");
    }

    public final native String[] getLocales();
}
```
#### 加载插件中的layout资源
我们使用LayoutInflate对象，一般使用方法如下：
```
View view = LayoutInflater.from(context).inflate(R.layout.main_fragment, null);
```

其中，R.layout.main_fragment我们可以通过上述方法获取其ID，那么关键的一步就是如何生成一个context？直接传入当前的context是不行的。 解决方案有2个： 
• 1.创建一个自己的ContextImpl，Override其方法。 
• 2.通过反射，直接替换当前context的mResources私有成员变量。 
当然，我们是使用第二种方案
```
@Override
    protected void attachBaseContext(Context context) {
        replaceContextResources(context);
        super.attachBaseContext(context);
    }

    /**
     * 使用反射的方式，使用Bundle的Resource对象，替换Context的mResources对象
     * @param context
     */
    public void replaceContextResources(Context context){
        try {
            Field field = context.getClass().getDeclaredField("mResources");
            field.setAccessible(true);
            field.set(context, mBundleResources);
            System.out.println("debug:repalceResources succ");
        } catch (Exception e) {
            System.out.println("debug:repalceResources error");
            e.printStackTrace();
        }
    }
```
我们在Activity的attachBaseContext方法中，对Context的mResources进行替换，这样，我们就可以加载离线apk中的布局了。
这样，加载插件的R资源就解决了。