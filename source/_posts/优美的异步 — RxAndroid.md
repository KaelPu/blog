---
title: 优美的异步 --- RxAndroid
date: 2017-08-27 10:59:48
tags: 
toc: true
---

这里和大家一起分享一下一个著名的Android异步库RxAndroid。它应该是2016年最流行的开源库之一。RxAndroid起源于RxJava，是一个专门针对Android版本的Rxjava库。[RxAndroid-Github](https://github.com/ReactiveX/RxAndroid) 目前最新的版本是**v2.0.x**我们今天的分享也基于2.0版本的API。

#### 响应式编程
什么是响应式编程？和平常经常听说的面向对象编程和函数式编程一样，响应式编程(Reactive Programming)就是一个编程范式，但是与其他编程范式不同的是它是基于数据流和变化传播的。我们经常在程序中这样写
``` java
A = B + C
```
A被赋值为B和C的值。这时，如果我们改变B的值，A的值并不会随之改变。而如果我们运用一种机制，当B或者C的值发现变化的时候，A的值也随之改变，这样就实现了**响应式**。
而响应式编程的提出，其目的就是简化类似的操作，因此它在用户界面编程领域以及基于实时系统的动画方面都有广泛的应用。另一方面，在处理嵌套回调的异步事件，复杂的列表过滤和变换的时候也都有良好的表现。
RxAndroid其实是一个响应式编程思想的实现库。也因为这样的思想，是它在一些方面表现的异常优秀。下面我将先用一个简单的例子，让大家直观的感受一下的样子。

##### 网络加载图片显示
``` java
Observable.just(getDrawableFromNet())
        .subscribeOn(Schedulers.newThread())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(new Consumer<Drawable>() {
            @Override
            public void accept(Drawable drawable) throws Exception {
                ((ImageView)findViewById(R.id.imageView)).setImageDrawable(drawable);
            }
        });
```

#### 环境搭建
RxAndroid环境只需求要引入如下项目即可，我们不但需要RxAndroid项目还需要RxJava项目。
``` 
compile 'io.reactivex.rxjava2:rxandroid:2.0.1'
compile 'io.reactivex.rxjava2:rxjava:2.1.5'
```
---

#### 基础知识
RxAndroid的核心就是“异步”两个字，其最关键的东西就是三个：
> Observable（被观察者）
> Observer（观察者）
> Subscriber （订阅）
Observable可以理解为事件的发送者，就好像快递的寄出者，而这些事件就好比快递 
Observer可以理解为事件的接收者，就好像快递的接收者

Subscriber  绑定两者

Observable可以发出一系列的 事件，这里的事件可以是任何东西，例如网络请求、复杂计算处理、数据库操作、文件操作等等，事件执行结束后交给 Observer回调处理。

那他们之间是如何进行联系的呢？答案就是通过subscribe()方法。
下面我们通过一个HelloDemo来看看Observable与Observer进行关联的典型方式，

``` java
  private void test_1() {
        Observable<String> oble = Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<String> e) throws Exception {
                e.onNext("hello");
                e.onComplete();
                e.onNext("hello2");

            }
        });

        Observer<String> oser = new Observer<String>() {
            @Override
            public void onSubscribe(@NonNull Disposable d) {
                Log.w("kaelpu","onSubscribe");
            }

            @Override
            public void onNext(@NonNull String s) {
                Log.w("kaelpu","onNext = "+s);
            }

            @Override
            public void onError(@NonNull Throwable e) {
                Log.w("kaelpu","onError" + e);
            }

            @Override
            public void onComplete() {
                Log.w("kaelpu","onComplete");
            }
        };

        Log.w("kaelpu","subscribe");
        oble.subscribe(oser);

    }
```

> 10-21 01:28:01.600 11386-11386/? W/kaelpu: subscribe
10-21 01:28:01.600 11386-11386/? W/kaelpu: onSubscribe
10-21 01:28:01.600 11386-11386/? W/kaelpu: onNext = hello
10-21 01:28:01.600 11386-11386/? W/kaelpu: onComplete


其实这段代码干了三件事：
1. 创建被观察者对象oble
2. 创建观察者oser
3. 连接观察者和被观察者

被观察者通过onNext函数给观察者通知结果
被贯彻者onComplete函数通知观察者执行结束
连接观察者和被观察者我们使用subscribe函数

* 通过打印的log我们可以看到观察者函数调用情况,调用subscribe函数去绑定观察者和被观察者时候，观察者的onSubscribe函数会被回调表示建立关联。
* 接着每当被观察者调用onNext给观察者发送数据时候，观察者的onNext 会收到回调，并且得到所发送的数据。
* 当被观察者调用onComplete函数时候,代表着完成，观察者的onComplete回调会被触发，并且断开了两者的关联，这时被观察者再发送数据，观察者也不会收到。

当然我们注意到观察者还有一个onError函数没有被触发过，那么该怎么触发呢，又代表着什么意思呢？我们来改变一下代码：
``` java
	private String error() throws Exception {
        throw new Exception();
    }
```
添加一个函数，名字随便但是返回值是String，这里我们叫做error函数。函数很简单就是抛出一个异常。然后我们继续修改被观察者的代码如下：
``` java
  Observable<String> oble = Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<String> e) throws Exception {
                e.onNext("hello");
                e.onNext(error());
                e.onNext("hello1");
                e.onComplete();
                e.onNext("hello2");

            }
        });
```
其实我们就添加了两行，添加了一个e.onNext(error()) 并且在之后还添加了一个e.onNext("hello1") 运行一下我们看看
> W/kaelpu: subscribe
 W/kaelpu: onSubscribe
 W/kaelpu: onNexthello
 W/kaelpu: onErrorjava.lang.Exception

折断log说明三个问题：
1. 被观察者onNext中是可以运行函数的
2. 如果运行的函数报错，则会调用我们观察者的onError函数
3. 当调用onError函数时候，也会断开关联，被观察者收不到后面的数据，但是观察者依然会继续发送。

最为关键的是onComplete和onError必须唯一并且互斥, 即不能发多个onComplete, 也不能发多个onError, 也不能先发一个onComplete, 然后再发一个onError, 反之亦然。
> 关于onComplete和onError唯一并且互斥这一点, 是需要自行在代码中进行控制, 如果你的代码逻辑中违背了这个规则, 并不一定会导致程序崩溃. 比如发送多个onComplete是可以正常运行的, 依然是收到第一个onComplete就不再接收了, 但若是发送多个onError, 则收到第二个onError事件会导致程序会崩溃.当我们写多个onComplete时，不会报错。

除了被观察者能断开关联，观察者也能主动断开连接，调用onSubscribe函数中传入的对象Disposable的dispose()函数即可完成断开连接，同样关联断开后，被观察者依然会继续发送数据


``**讲到这里第一感觉是不是？**``


![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1967257-07350b3d1d8d67d5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)


就输出个数字就这么麻烦，完全没看出哪里方便了！别着急我刚开始看RxAndroid文章也是这样的感觉，而且很多网上的文章都没有解释这个问题。所以看一会你就更晕了。别着急我一起深呼吸,来看看如何简化操作

你可能觉得，我就打印几个数，还要把Observable写的那么麻烦，能不能简便一点呢？答案是肯定的，RxAndroid内置了很多简化创建Observable对象的函数，比如Observable.just就是用来创建只发出一个事件就结束的Observable对象，上面创建Observable对象的代码可以简化为一行

``` java
		Observable<String> observable = Observable.just("hello");
```

同样对于Observer，这个例子中，我们其实并不关心OnComplete和OnError，我们只需要在onNext的时候做一些处理，这时候就可以使用Consumer类。

``` java
		Observable<String> observable = Observable.just("hello");
		Consumer<String> consumer = new Consumer<String>() {
	       @Override
	       public void accept(String s) throws Exception {
	           System.out.println(s);
	       }
		};
		observable.subscribe(consumer);
```

其实在RxAndroid中，我们可以为 Observer中的三种状态根据自身需要分别创建一个回调动作，通过Action 来替代onComplete():，通过Consumer来替代 onError(Throwable t)和onNext(T t)

``` java
Observable<String> observable = Observable.just("hello");
    Action onCompleteAction = new Action() {
        @Override
        public void run() throws Exception {
            Log.i("kaelpu", "complete");
        }
    };
    Consumer<String> onNextConsumer = new Consumer<String>() {
        @Override
        public void accept(String s) throws Exception {
            Log.i("kaelpu", s);
        }
    };
    Consumer<Throwable> onErrorConsumer = new Consumer<Throwable>() {
        @Override
        public void accept(Throwable throwable) throws Exception {
            Log.i("kaelpu", "error");
        }
    };
    observable.subscribe(onNextConsumer, onErrorConsumer, onCompleteAction);

}
```
subscribe()有多个重载的方法:
``` java
 public final Disposable subscribe() {}
 public final Disposable subscribe(Consumer<? super T> onNext) {}
 public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError) {} 
 public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError, Action onComplete) {}
 public final Disposable subscribe(Consumer<? super T> onNext, Consumer<? super Throwable> onError, Action onComplete, Consumer<? super Disposable> onSubscribe) {}
 public final void subscribe(Observer<? super T> observer) {}
```

不带任何参数的subscribe() 表示Observer不关心任何事件,Observable发送什么数据都随你 
带有一个Consumer参数的方法表示Observer只关心onNext事件, 其他的事件我假装没看见, 因此我们如果只需要onNext事件可以这么写

只要我们再本节中能明白观察者和被观察者之间是如何工作关联的就可以

---

#### 线程调度
关键的章节来了，看完上面的基础知识，很多人都会感觉就一个发送，一个接收，不就是个观察者模式嘛，感觉一点卵用都没有，还写这么多回调方法！完全没有看出什么优点。那么这一节就让你看到RxAndroid真正厉害的地方。

正常情况下, Observer和Observable是工作在同一个线程中的, 也就是说Observable在哪个线程发事件, Observer就在哪个线程接收事件. 
RxAndroid中, 当我们在主线程中去创建一个Observable来发送事件, 则这个Observable默认就在主线程发送事件. 
当我们在主线程去创建一个Observer来接收事件, 则这个Observer默认就在主线程中接收事件，但其实在现实工作中我们更多的是需要进行线程切换的，最常见的例子就是在子线程中请求网络数据，在主线程中进行展示

要达到这个目的, 我们需要先改变Observable发送事件的线程, 让它去子线程中发送事件, 然后再改变Observer的线程, 让它去主线程接收事件. 通过RxAndroid内置的线程调度器可以很轻松的做到这一点. 接下来看一段代码:
``` java
Observable<Integer> observable = Observable.create(new ObservableOnSubscribe<Integer>() {
        @Override
        public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
            Log.d("kaelpu", "Observable thread is : " + Thread.currentThread().getName());
            Log.d("kaelpu", "emitter 1");
            emitter.onNext(1);
        }
    });

    Consumer<Integer> consumer = new Consumer<Integer>() {
        @Override
        public void accept(Integer integer) throws Exception {
            Log.d("kaelpu", "Observer thread is :" + Thread.currentThread().getName());
            Log.d("kaelpu", "onNext: " + integer);
        }
    };

    observable.subscribeOn(Schedulers.newThread())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(consumer);
}
```

>  Observable thread is : RxNewThreadScheduler-1
 emitter 1
 Observer thread is :main
 onNext: 1

可以看到, observable发送事件的线程的确改变了, 是在一个叫 RxNewThreadScheduler-1的线程中发送的事件, 而consumer 仍然在主线程中接收事件, 这说明我们的目的达成了, 接下来看看是如何做到的.

这段代码只不过是增加了两行代码:

``` java
.subscribeOn(Schedulers.newThread())
.observeOn(AndroidSchedulers.mainThread())
```

简单的来说, subscribeOn() 指定的是Observable发送事件的线程, observeOn() 指定的是Observer接收事件的线程. 
多次指定Observable的线程只有第一次指定的有效, 也就是说多次调用subscribeOn() 只有第一次的有效, 其余的会被忽略. 
多次指定Observer的线程是可以的, 也就是说每调用一次observeOn() , Observer的线程就会切换一次.例如:

``` java
observable.subscribeOn(Schedulers.newThread())
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .observeOn(Schedulers.io())
        .subscribe(consumer);
```

>  Observable thread is : RxNewThreadScheduler-1
 emitter 1
 Observer thread is :RxCachedThreadScheduler-2
 onNext: 1

可以看到, Observable虽然指定了两次线程, 但只有第一次指定的有效, 依然是在RxNewThreadScheduler线程中, 而Observer则跑到了RxCachedThreadScheduler 中, 这个CacheThread其实就是IO线程池中的一个.
在 RxAndroid 中，提供了一个名为 Scheduler 的线程调度器，RxAndroid 内部提供了4个调度器，分别是：


* Schedulers.io(): I/O 操作（读写文件、数据库、网络请求等），与newThread()差不多，区别在于io() 的内部实现是是用一个无数量上限的线程池，可以重用空闲的线程，因此多数情况下 io() 效率比 newThread() 更高。值得注意的是，在 io() 下，不要进行大量的计算，以免产生不必要的线程；
* Schedulers.newThread(): 开启新线程操作；
* Schedulers.immediate(): 默认指定的线程，也就是当前线程；
* Schedulers.computation():计算所使用的调度器。这个计算指的是 CPU 密集型计算，即不会被 I/O等操作限制性能的操作，例如图形的计算。这个 Scheduler 使用的固定的线程池，大小为 CPU 核数。值得注意的是，不要把 I/O 操作放在 computation() 中，否则 I/O 操作的等待时间会浪费 CPU；
* AndroidSchedulers.mainThread(): Rxndroid 扩展的 Android 主线程； 

这些内置的Scheduler已经足够满足我们开发的需求, 因此我们应该使用内置的这些选项, 在RxAndroid内部使用的是线程池来维护这些线程, 所有效率也比较高。

对于线程还需要注意

*  create() , just() , from()   等                 --- 事件产生   
* map() , flapMap() , scan() , filter()  等    --  事件加工
* subscribe()                                          --  事件消费

事件产生：默认运行在当前线程，可以由 subscribeOn()  自定义线程
事件加工：默认跟事件产生的线程保持一致, 可由 observeOn() 自定义线程
事件消费：默认运行在当前线程，可以有observeOn() 自定义

好了说了这么多了,我们来写个简单的异步的例子，看看实际效果。我们这个例子就以加载网络图片并显示为例：
首先我们写一个耗时函数，用来模拟图片请求
``` java
    // 模拟网络请求图片
    private Drawable getDrawableFromUrl(String url){
        try {
            Thread.sleep(6000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return getResources().getDrawable(R.drawable.baidu);
    }
```
代码很简答，就是线程sleep 6秒，然后返回一张图片，如果运行在主线程那就会NAR，然后我么来用RxAndroid写一下这个异步拉去图片并显是的操作！

``` java
Observable.just(getDrawableFromNet("http://www.baidu.com/icon.png"))
        .subscribeOn(AndroidSchedulers.mainThread())
        .observeOn(Schedulers.newThread())
        .subscribe(new Consumer<Drawable>() {
            @Override
            public void accept(Drawable drawable) throws Exception {
                ((ImageView)findViewById(R.id.imageView)).setImageDrawable(drawable);
            }
        });
```
就这几行代码就搞定了！自己结合上面讲的理解一下~这里就不做解释了！因为我们还有更重要的一个环节，这个环节堪称RxAndroid的精髓！

#### 操作符的使用

在了解基本知识和线程调度后，我们来学习一下RxAndroid各种神奇的操作符

**Map** 
Map是RxAndroid中最简单的一个变换操作符了, 它的作用就是对Observable发送的每一个事件应用一个函数, 使得每一个事件都按照指定的函数去变化。
``` java
Observable.create(new ObservableOnSubscribe<Integer>() {
        @Override
        public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
            emitter.onNext(1);
            emitter.onNext(2);
            emitter.onNext(3);
        }
    }).map(new Function<Integer, String>() {
        @Override
        public String apply(Integer integer) throws Exception {
            return "This is result " + integer;
        }
    }).subscribe(new Consumer<String>() {
        @Override
        public void accept(String s) throws Exception {
            Log.d("kaelpu", s);
        }
    });
```

> This is result 1
This is result 2
This is result 3

通过Map, 可以将Observable发来的事件转换为任意的类型, 可以是一个Object, 也可以是一个集合，功能非常强大

例子：还是以图片加载的例子，我们传进来一个图片的路径，然后通过Map把drawble转换成bitmap再发送给观察者

``` java
Observable.just(getDrawableFromNet())
        .map(new Function<Drawable, Bitmap>() {
            @Override
            public Bitmap apply(@NonNull Drawable drawable) throws Exception {
                BitmapDrawable bt = (BitmapDrawable)drawable;
                return bt.getBitmap();
            }
        })
        .subscribeOn(AndroidSchedulers.mainThread())
        .observeOn(Schedulers.newThread())
        .subscribe(new Consumer<Bitmap>() {
            @Override
            public void accept(Bitmap bitmap) throws Exception {

            }
        });
```
Observable –> map变换 –> Observable
url -> drawable -> bitmap

不用到处调代码，直接一个链式操作... 是不是感觉很爽！

**ZIP** 
Zip通过一个函数将多个Observable发送的事件结合到一起，然后发送这些组合到一起的事件. 它按照严格的顺序应用这个函数。它只发射与发射数据项最少的那个Observable一样多的数据。
```java
Observable<Integer> observable1 = Observable.create(new ObservableOnSubscribe<Integer>() {
        @Override
        public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
            Log.d(TAG, "emitter 1");
            emitter.onNext(1);
            Log.d(TAG, "emitter 2");
            emitter.onNext(2);
            Log.d(TAG, "emitter 3");
            emitter.onNext(3);
            Log.d(TAG, "emitter 4");
            emitter.onNext(4);
            Log.d(TAG, "emit complete1");
            emitter.onComplete();
        }
    });

    Observable<String> observable2 = Observable.create(new ObservableOnSubscribe<String>() {
        @Override
        public void subscribe(ObservableEmitter<String> emitter) throws Exception {
            Log.d(TAG, "emitter A");
            emitter.onNext("A");
            Log.d(TAG, "emitter B");
            emitter.onNext("B");
            Log.d(TAG, "emitter C");
            emitter.onNext("C");
            Log.d(TAG, "emitter complete2");
            emitter.onComplete();
        }
    });

    Observable.zip(observable1, observable2, new BiFunction<Integer, String, String>() {
        @Override
        public String apply(Integer integer, String s) throws Exception {
            return integer + s;
        }
    }).subscribe(new Observer<String>() {
        @Override
        public void onSubscribe(Disposable d) {
            Log.d(TAG, "onSubscribe");
        }

        @Override
        public void onNext(String value) {
            Log.d(TAG, "onNext: " + value);
        }

        @Override
        public void onError(Throwable e) {
            Log.d(TAG, "onError");
        }

        @Override
        public void onComplete() {
            Log.d(TAG, "onComplete");
        }
    });
```
我们分别创建了observable, 一个发送1,2,3,4,Complete, 另一个发送A,B,C,Complete, 接着用Zip把发出的事件组合, 来看看运行结果吧: 
> onSubscribe
 emitter 1
 emitter 2
emitter 3
emitter 4
 emit complete1
 emitter A
 onNext: 1A
 emitter B
 onNext: 2B
 emitter C
 onNext: 3C
 emitter complete2
onComplete

观察发现observable1发送事件后，observable2才发送 
这是因为我们两个observable都是运行在同一个线程里, 同一个线程里执行代码肯定有先后顺序呀.


**from**
在Rxndroid的from操作符到2.0已经被拆分成了3个，fromArray, fromIterable, fromFuture接收一个集合作为输入，然后每次输出一个元素给subscriber。
``` java
Observable.fromArray(new Integer[]{1, 2, 3, 4, 5}).subscribe(new Consumer<Integer>() {
    @Override
    public void accept(Integer integer) throws Exception {
        Log.i(TAG, "number:" + integer);
    }
});
```

> number:1
 number:2
number:3
number:4
 number:5
 
 注意：如果from()里面执行了耗时操作，即使使用了subscribeOn(Schedulers.io())，仍然是在主线程执行，可能会造成界面卡顿甚至崩溃，所以耗时操作还是使用Observable.create(…);


**filter**
条件过滤，去除不符合某些条件的事件。举个栗子:
``` java
Observable.fromArray(new Integer[]{1, 2, 3, 4, 5})
       .filter(new Predicate<Integer>() {
           @Override
           public boolean test(Integer integer) throws Exception {
               // 偶数返回true，则表示剔除奇数，留下偶数
               return integer % 2 == 0;

           }
       }).subscribe(new Consumer<Integer>() {
    @Override
    public void accept(Integer integer) throws Exception {
        Log.i(TAG, "number:" + integer);
    }
});
```
> number:2
 number:4



**take** 
最多保留的事件数。
``` java
 Observable.just("1", "2", "6", "3", "4", "5").take(2).subscribe(new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {

            }

            @Override
            public void onNext(String value) {
                Log.d(TAG,value);
            }

            @Override
            public void onError(Throwable e) {

            }

            @Override
            public void onComplete() {

            }
        });
```
>  1
 2

可以发现我们发送了6个String，最后只打印了前两个，这就是take过滤掉的结果



**doOnNext** 
如果你想在处理下一个事件之前做某些事，就可以调用该方法

``` java
Observable.fromArray(new Integer[]{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12}).filter(new Predicate<Integer>() {
    @Override
    public boolean test(Integer integer) throws Exception {
        // 偶数返回true，则表示剔除奇数
        return integer % 2 == 0;
    }
})// 最多保留三个，也就是最后剩三个偶数
        .take(3).doOnNext(new Consumer<Integer>() {
    @Override
    public void accept(Integer integer) throws Exception {
        // 在输出偶数之前输出它的hashCode
        Log.i(TAG, "hahcode = " + integer.hashCode() + "");
    }
}).subscribe(new Observer<Integer>() {
    @Override
    public void onSubscribe(Disposable d) {

    }

    @Override
    public void onNext(Integer value) {
        Log.i(TAG, "number = " + value);
    }

    @Override
    public void onError(Throwable e) {

    }

    @Override
    public void onComplete() {

    }
});
```

>  hahcode = 2
number = 2
 hahcode = 4
number = 4
 hahcode = 6
 number = 6

---

#### 针对Android的一些扩展
RxAndroid是RxJava的一个针对Android平台的扩展。它包含了一些能够简化Android开发的工具。
首先，AndroidSchedulers提供了针对Android的线程系统的调度器。需要在UI线程中运行某些代码？很简单，只需要使用AndroidSchedulers.mainThread():

``` java
retrofitService.getImage(url)
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(bitmap -> myImageView.setImageBitmap(bitmap));
```

接着要介绍的就是AndroidObservable，它提供了跟多的功能来配合Android的生命周期。bindActivity()和bindFragment()方法默认使用AndroidSchedulers.mainThread()来执行观察者代码，这两个方法会在Activity或者Fragment结束的时候通知被观察者停止发出新的消息。

``` java
AndroidObservable.bindActivity(this, retrofitService.getImage(url))
    .subscribeOn(Schedulers.io())
    .subscribe(bitmap -> myImageView.setImageBitmap(bitmap);
```

我自己也很喜欢AndroidObservable.fromBroadcast()方法，它允许你创建一个类似BroadcastReceiver的Observable对象。下面的例子展示了如何在网络变化的时候被通知到：
``` java
IntentFilter filter = new IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION);
AndroidObservable.fromBroadcast(context, filter)
    .subscribe(intent -> handleConnectivityChange(intent));
```

最后要介绍的是ViewObservable,使用它可以给View添加了一些绑定。如果你想在每次点击view的时候都收到一个事件，可以使用ViewObservable.clicks()，或者你想监听TextView的内容变化，可以使用ViewObservable.text()

``` java
ViewObservable.clicks(mCardNameEditText, false)
    .subscribe(view -> handleClick(view));
```

---

#### RxAndroid的一些使用场景
这里总结了一些很合适使用RxAndroid的场景，供大家打开脑洞~分享时候有时间给大家看看demo


1. 界面需要等到多个接口并发取完数据，再更新

``` java
Observable<String> observable1 = Observable.create(new ObservableOnSubscribe<String>() {
        @Override
        public void subscribe(ObservableEmitter<String> e) throws Exception {
            e.onNext("haha");
        }
    }).subscribeOn(Schedulers.newThread());

    Observable<String> observable2 = Observable.create(new ObservableOnSubscribe<String>() {
        @Override
        public void subscribe(ObservableEmitter<String> e) throws Exception {
            e.onNext("hehe");
        }
    }).subscribeOn(Schedulers.newThread());


    Observable.merge(observable1, observable2)
            .subscribeOn(Schedulers.newThread())
            .subscribe(new Observer<String>() {
                @Override
                public void onSubscribe(Disposable d) {

                }

                @Override
                public void onNext(String value) {
                    Log.d(TAG,value);
                }

                @Override
                public void onError(Throwable e) {

                }

                @Override
                public void onComplete() {

                }
            });
```
3. 界面按钮需要防止连续点击的情况

``` java
RxView.clicks(button)
        .throttleFirst(1, TimeUnit.SECONDS)
        .subscribe(new Observer<Object>() {
            @Override
            public void onCompleted() {

            }

            @Override
            public void onError(Throwable e) {

            }

            @Override
            public void onNext(Object o) {
                Log.i(TAG, "do clicked!");
            }
        });
```

4. 响应式的界面 比如勾选了某个checkbox，自动更新对应的preference
``` java

SharedPreferences preferences = PreferenceManager.getDefaultSharedPreferences(context);
RxSharedPreferences rxPreferences = RxSharedPreferences.create(preferences);

Preference<String> username = rxPreferences.getString("username");
Preference<Boolean> showWhatsNew = rxPreferences.getBoolean("show-whats-new", true);

username.asObservable().subscribe(new Action1<String>() {
  @Override public void call(String username) {
    Log.d(TAG, "Username: " + username);  读取到当前值
  }
}

RxCompoundButton.checks(showWhatsNewView)
    .subscribe(showWhatsNew.asAction());

```

#### 最后的话
通过本篇文章，大家应该对RxAndroid有个大体的认识了，也应该体会到它在异步操作，代码链式书写等方面的优势了。需要注意的是由于RxJava存在理解的门槛，贸然引入项目要确保协同开发的人员也都对Rxjava有所了解~

---

[参考资料]

1. [响应式编程简介](http://blog.csdn.net/womendeaiwoming/article/details/46506017)
2. [深入浅出RxJava](http://blog.csdn.net/lzyzsd/article/details/45033611)
3. [RxPreferences 简单整理](http://blog.csdn.net/a332324956/article/details/51345235)
4.  [RxBinding安卓UI响应式编程](http://www.jianshu.com/p/34cf96b72102)
5.  [给 Android 开发者的 RxJava 详解](http://gank.io/post/560e15be2dca930e00da1083)