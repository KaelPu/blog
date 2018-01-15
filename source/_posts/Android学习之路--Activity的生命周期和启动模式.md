---
title: Android学习之路 -- Activity的生命周期和启动模式
date: 2017-03-08 00:24:13
tags: Android学习之路
---

### Activity 的生命周期和启动模式
#### Activity的生命周期
![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1967257-e11df06aa2630194.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)

 - onCreate - 构建Activity
 - onRestart - Activity在重启,一般来说Activity恢复时在onStop后执行
 - onStart - Activity开始启动，这时Activity已经可见，但是仍然处于后台状态，所以尽管处于可见状态，用户也看不见,事件也无法响应。
 - onResume -  Activity已经启动，并且处于前台状态.用户可见，事件可响应。
 - onPause - Activity正在执行停止 一般情况紧接着会有onStop。注意只有当前onPause函数执行，新Activity才会执行onResume，所以切记不可做耗时处理。
 - onStop - Activity即将停止 ，同样不能太耗时。
 - onDestroy - Activity最后一个生命函数，可以做资源回收。
 
 **总结一下**
  - onStart 和 onStop 是一对的他们代表了Activity是否可见
  - onResume 和 onPause 是一对他们表示了Activiry是否在前台/后台
 
 由上面两个总结我们可以得知两种特别的情况：
 1. 如果新的Activity主题是透明的，那么可以知道旧Activity仍然会显示，所以旧Activity的onStop是不会执行的。
 2. 只有旧Activity退到后台，新Activity才能到前台来，也就是旧的onPause执行完新的才会执行onResume

   **`异常情况`**
当系统配置发生改变或者内存不足时候,系统会异常杀死Activity,每当这时系统都会调用onSaveInstanceState 和 onRestoreInstanceState 这两个函数
   1. onSaveInsatanceState 函数会保存当前Activity的数据，它会在onStop之前调用
   2. onRestoreInstanceState在onStart之后调用

  这里简单说下onSaveInstanceState原理，首先Activity会委托Window去执行保存，Window会找到GroupView，大多数情况下GroupView就是DecorView ，然后DecorView会遍历所有的子View的onSaveInstanState方法去执行保存方法。所以想知道View都保存了那些数据，可以查看该View的onSaveInstanceState方法。数据恢复会调用onRestoreInsatnceState方法，方然恢复的过程也和上面保存过程类似，这里就不再赘述。

 接下来我们看看这两个函数的参数，两个参数都是Bundle类型，而其实保存的数据也都保存在这个bundle对象中。所我们也可以将我们Activity数据保存在bundle中。这里需要注意，在生命周期函数中onCreate函数参数也是bundle，它和onRestoreInsatnceState这个函数有什么关系呢。其实他俩都能做恢复操作，只是Activity在正常启动时候onCreate函数中的bundle可能为空需要判空，然而onRestoreInsatnceState并不需要。
  
  如果不想让屏幕因为系统配置的问题改变，我们可以给Activity设置configChanges属性。
  
---

#### Activity 启动模式
Activity有四种启动模式，这四种启动模式决定了Activity是否需要重新创建还是复用之前已有的对象。这些已存在的对象会被放进任务栈中被管理。系统可以有多个任务栈。
- standard - *标准模式* 该模式很简单，系统每启动一个Activity都会创建一个新的对象，谁启动了该Activity它就会被加入到启动者的任务栈中。**这里需要注意由于ApplicationContext是不包含任务栈的，所以启动标准模式的Activity时会报错。**解决这个问题的办法就是给Activity加上FLAG_ACTIVITY_NEW_TASK标记，其实这就是singleTask启动模式，**这里注意给XML添加singleTask模式也不能解决该错误，必须代码添加**后面我们会介绍。
-
- singleTop - 栈顶复用。在这种模式下如果当前任务栈中栈顶存在Activity对象，则会直接复用该对象，并且直接调用该对象的onNewIntent函数，然而方法的参数Intent对象中可取出当前请求的信息。**但是需要注意这个时候onCreate onStart生命周期不会被调用**因为Activity并没有被重新创建。
-
- singleTask - 栈内复用，其实就是一个**栈内单例**模式，就是只要栈内存在该对象就不会重新创建，只会把该对象调到栈顶，并且调用onNewIntent函数。**这里需要注意说是调到栈顶，其实不准确确切的说是依次出栈，直到栈顶是该对象**。也就是说原本在该对象之上的对象全部会被出栈，这就是该模式默认具有clearTop的效果。
-
- singleInstance - 单例模式，这个模式和singleTask区别在于,使用这个模式的Activity对象会独立存在一个栈中，**并且这个栈只给它用，别人不能用这个栈**，比如说它再启动一个Activity，那么这个Activity如果没有指定栈，那个这个新对象不会进入这个独有的栈，会进入启动singleInstance模式的那个栈。有点绕可以看看图在理解一下

**`实验`**
这里做了一个实验，我们有A、B、C 三个Activity其中B为singleInstance模式 A和C为standard模式，我用AppContext上下文启动B，报错说没有堆栈，我们将B改成singleTask模式，任然报错，只有我们代码添加FLAG_ACTIVITY_NEW_TASK才可
成功运行。
- 结论1 ：AppContext启动Activity 必须代码设置FLAG_ACTIVITY_NEW_TASK属性

那么是因为XML设置的没生效么？我们仍然将B设置成singleInstance模式然后按照如下顺序启动 A-》B-》C 如果xml中singleInstance这个没生效，这三个对象应该在同一个任务栈。可是结果AC在一个栈，而B在一个栈，说明XML中的配置也生效了。



![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1967257-e6cb65a5ce3ca16d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)


至于怎么创建新的任务栈我们可以在xml中配置TaskAffinity属性注意该属性只和singleTask模式搭配才有效。仔细结合上面的思考一下为什么？查看任务栈情况可以使用如下命令，红色框就是不同栈，蓝色框是Activity名。
``` adb shell dumpsys activity ```

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1967257-008e83394ce5cbb7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)

---

#### IntentFilter的匹配规则
启动Activity有两种方式一种显示调用，一种隐式调用。显示调用很简单，只需要指定包名和类名即可，而隐式调用需要明确的指定出组件的信息。原则上说一个Intent不应该既包含显示调用又包含隐式调用，如果两个都包含，则会以显示调用为主。这里我们主要介绍一下隐式调用。隐式调用需要Intent能够匹配所有目标组件的IntentFilter中所设置的过滤信息，如果不匹配就无法启动。IntentFilter中过滤信息有三种action、category、data。
- action - Intent中的action必须和组件过滤规则中的其中一个action相同，换句话说Intent可以有好几个action，只要组件配了其中一个action即可匹配
- category - Intent中的所有category必须在组件过滤规则中都存在
- data - Intent中的data规则和action相似

data由两部分组成 mineType 和 URL
- mineType 指媒体类型 image/jpeg video/* 等
- URL URL能包含的数据变化就多了具体定义用到时候可以上网查这里主要了解他们匹配规则即可