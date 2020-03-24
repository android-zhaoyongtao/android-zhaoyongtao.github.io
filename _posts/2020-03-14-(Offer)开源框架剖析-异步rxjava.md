---
layout:     post
title:      开源框架剖析-异步rxjava
subtitle:   
date:       2020-03-14
author:     ZYT
header-img: img/post-bg-offer.jpeg
catalog: true
tags:
    - Android
    - Offer
    - 异步框架
    - 开源框架剖析
    - rxjava
---

# 开源框架剖析-异步rxjava
rxjava核心思想是观察者模式

RxJava提供了丰富的操作符链式编程，使代码简洁，易于维护，线程切换功能极其强大，可任意指定观察者发生的线程以及被观察者的线程

观察者模式使用场景

1，一个方面的操作依赖于另一个方面状态变化

2，如果在更改一个对象的时候，需要同时连带更改其他对象

3，当一个对象需要通知其他对象，但又希望其他被通知对象是松散耦合的


### rxjava四个基本元素

1，被观察者Observeable

2，观察者Observe

3，订阅subscribe  将 Observable 与 Observer 关联起来

4，事件 （包括 onNext,onComplete,onError 等事件）


### 使用代码
```
//第一步：创建被观察者：create
Observable<String> observable = Observable.create(new ObservableOnSubscribe<String>() {
    @Override
    public void subscribe(ObservableEmitter<String> emitter) throws Exception {
        Thread.sleep(3000);
        emitter.onNext("hello world");
        emitter.onComplete();
    }
});

//第二部：创建观察者
Consumer<String> nextConsumer = new Consumer<String>() {
    @Override
    public void accept(String s) throws Exception {
        textView.setText(s);
    }
};

//第三部：订阅
observable.subscribeOn(Schedulers.newThread())//子线程执行被观察着
        .observeOn(AndroidSchedulers.mainThread())//主线程回调
        .subscribe(nextConsumer);
```
### 线程切换

被观察者执行线程设置-subscribeOn

        observable.subscribeOn(Schedulers.computation());//cpu密集型计算
        observable.subscribeOn(Schedulers.newThread());//为每个任务创建一个新线程
        observable.subscribeOn(Schedulers.io());//无上限线程池
        observable.subscribeOn(Schedulers.single());//该调度器的线程池只能同时执行一个线程。
        observable.subscribeOn(Schedulers.trampoline());//其它排队的任务完成后，在当前线程排队开始执行。
        observable.subscribeOn(AndroidSchedulers.mainThread());//主线程

观察者执行线程设置-subscribeOn
    
        observable.observeOn(AndroidSchedulers.mainThread());

observeOn()指定的是它之后的操作所在的线程通过observeOn()的**多次**调用，程序实现了线程的多次切换。

subscribeOn()的位置放在那里都行，但它只能调用**一次**，原因就是subscribeOn()是通过新建Observable的方式。


RxJava的线程控制通过Scheduler来实现，通过Scheduler的createWorker()来获取Worker，由worker决定执行线程，真正由Worker的schedule()来运行Runnable。

