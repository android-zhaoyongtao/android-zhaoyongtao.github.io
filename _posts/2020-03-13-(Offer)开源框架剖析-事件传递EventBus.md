---
layout:     post
title:      开源框架剖析-事件传递EventBus
subtitle:   Android事件发布、订阅框架
date:       2020-03-13
author:     ZYT
header-img: img/post-bg-offer.jpeg
catalog: true
tags:
    - Android
    - Offer
    - 事件传递
    - 开源框架剖析
    - EventBus
---

通过EventBus可以简化事件传递，它的优点是代码简洁，使用简单，解耦事件的发布和订阅。


### EventBus事件流程
![](https://tva1.sinaimg.cn/large/00831rSTly1gd0emu4epzj31qc0reth9.jpg)
### EventBus三要素：
1，Event：事件

2，Subscriber：事件的订阅者 
    在EventBus3.0之后，事件处理的方法可以随意取名，但是需要添加一个注解@Subscribe，并指定线程模型（默认为POSTING）。

3，Publisher：事件发布者 
    发送消息，传递数据。

### EventBus使用

1，添加依赖

    implemention 'org.greenrobot:eventbus:3.1.1'
    
2，定义消息事件MessageEvent

    public class MessageEvent {
    
    }
    
3，在Activity1中注册事件、处理事件，解注册事件
```
Activity1 {
        void onCreat(...){
            EventBus.getDefault().register(this);
        }
        
        @Subscribe(threadMode = ThreadMode.MAIN, sticky = false, priority = 0)
        public void handleData(MessageEvent mMessageEvent) {
        }
        
        void onDestory(...){
            EventBus.getDefault().unregister(this);
        }
}
```
3，在Activity2中发送事件

    EventBus.getDefault().post(messageEvent);
### EventBus实现原理
#### EventBus.getDefault()

通过单例模式来创建。通过EventBuilder的build模式来构建
```
EventBus
    public static EventBus getDefault() {
            if (defaultInstance == null) {
                synchronized (EventBus.class) {
                    if (defaultInstance == null) {
                        defaultInstance = new EventBus();
                    }
                }
            }
            return defaultInstance;
    }
    
    EventBus(EventBusBuilder builder) {
            //
            subscriptionsByEventType = new HashMap<>();
            typesBySubscriber = new HashMap<>();
            stickyEvents = new ConcurrentHashMap<>();
            mainThreadPoster = new HandlerPoster(this, Looper.getMainLooper(), 10);
            backgroundPoster = new BackgroundPoster(this);
            asyncPoster = new AsyncPoster(this);
            indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;
            subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
                    builder.strictMethodVerification, builder.ignoreGeneratedIndex);
            logSubscriberExceptions = builder.logSubscriberExceptions;
            logNoSubscriberMessages = builder.logNoSubscriberMessages;
            sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
            sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
            throwSubscriberException = builder.throwSubscriberException;
            eventInheritance = builder.eventInheritance;
            executorService = builder.executorService;
    }
```
它的构造方法并没有设置成私有的，因此我们可以生成多个不同功能的EventBus。getDefault是默认的帮我们造好的一个
#### EventBus的主要事件集合filed
    //以event(即事件类)为key，以订阅列表(Subscription)为value，事件发送之后，在这里寻找订阅者,而Subscription又是一个CopyOnWriteArrayList，这是一个线程安全的容器。Subscription是一个封装类，封装了订阅者、订阅方法这两个类。
    private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
    
    //以订阅者类为key，以event事件类为value，在进行register或unregister操作的时候，会操作这个map。订阅者与该订阅者所订阅事件的集合关联。
    private final Map<Object, List<Class<?>>> typesBySubscriber;
    
    //保存的是粘性事件
    private final Map<Class<?>, Object> stickyEvents;
    
1，subscriptionsByEventType：
#### EventBus的@Subscribe注解
```
public @interface Subscribe {
        ThreadMode threadMode() default ThreadMode.POSTING;
        //粘性时间，在发布之后才注册的也能接受到的特殊事件
        boolean sticky() default false;
        int priority() default 0;
}

public enum ThreadMode {
        //默认线程模式回调，直接调用订阅方法不管是否在不再主线程。需要发布消息的线程在主线程
        POSTING,
    
        //主线程回调
        MAIN,
    
        //后台线程
        BACKGROUND,
    
        //不论发布线程是否在主线程，都会弄个空线程处理。线程都是独立的，不会卡顿。所以当处理事件的Method是耗时的，需要使用此模式
        ASYNC
}
```

#### EventBus的regiser方法

利用反射经过去重、校验、封装等操作将register的类的信息和订阅事件添加到EventBus的对应事件集合中。

```
public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
        
        private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
                ////获取订阅方法的参数类型(订阅事件)
                Class<?> eventType = subscriberMethod.eventType;
                ////订阅者和订阅方法 封装成Subscription
                Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
                ////获取当前订阅事件中Subscription的List集合
                CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
                ////如果为null，说明该subscriber尚未注册该事件
                if (subscriptions == null) {
                    subscriptions = new CopyOnWriteArrayList<>();
                    subscriptionsByEventType.put(eventType, subscriptions);
                } else {
                    ////订阅者已经注册则抛出Exception
                    if (subscriptions.contains(newSubscription)) {
                        throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                                + eventType);
                    }
                }
        
                ////根据优先级放进subscriptions，优先级高的会先被通知
                int size = subscriptions.size();
                for (int i = 0; i <= size; i++) {
                    if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                        subscriptions.add(i, newSubscription);
                        break;
                    }
                }
                
                ////根据subscriber(订阅者)来获取它的所有订阅事件
                List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
                if (subscribedEvents == null) {
                    subscribedEvents = new ArrayList<>();
                    typesBySubscriber.put(subscriber, subscribedEvents);
                }
                subscribedEvents.add(eventType);
        
                //粘性事件的处理
                if (subscriberMethod.sticky) {
                    if (eventInheritance) {
                        Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                        for (Map.Entry<Class<?>, Object> entry : entries) {
                            Class<?> candidateEventType = entry.getKey();
                            if (eventType.isAssignableFrom(candidateEventType)) {
                                Object stickyEvent = entry.getValue();
                                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                            }
                        }
                    } else {
                        Object stickyEvent = stickyEvents.get(eventType);
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
        }
}
```

#### EventBus的post方法
```
public void post(Object event) {
        //currentPostingThreadState存储在ThreadLocal中，即线程本地变量。
        PostingThreadState postingState = currentPostingThreadState.get();
        List<Object> eventQueue = postingState.eventQueue;
        //将当前事件event加入到队列中
        eventQueue.add(event);

        if (!postingState.isPosting) {////是否有事件正在分发
                postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();
                postingState.isPosting = true;
                if (postingState.canceled) {
                    throw new EventBusException("Internal error. Abort state was not reset");
                }
                try {
                    while (!eventQueue.isEmpty()) {
                        //调用postSingleEvent进行分发
                        postSingleEvent(eventQueue.remove(0), postingState);
                    }
                } finally {
                    postingState.isPosting = false;
                    postingState.isMainThread = false;
                }
        }
}
```
最终会调用到EventBus#postToSubscription方法，在这里进行了线程的调度
```
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {
            case POSTING:
                invokeSubscriber(subscription, event);
                break;
            case MAIN:
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            case BACKGROUND:
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            case ASYNC:
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
}
```
invokeSubscriber是直接利用反制执行事件方法。

mainThreadPoster.enqueue(subscription, event);是利用handler调度去主线程执行方法

backgroundPoster.enqueue(subscription, event);是利用线程池execute一个Runnable，依次执行queue的事件

asyncPoster.enqueue(subscription, event);是利用同一个线程池为每一个task开辟一个线程执行事件

#### EventBus的unregiser方法

    public synchronized void unregister(Object subscriber) {
            List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
            if (subscribedTypes != null) {
                for (Class<?> eventType : subscribedTypes) {
                    unsubscribeByEventType(subscriber, eventType);
                }
                typesBySubscriber.remove(subscriber);
            } else {
                Log.w(TAG, "Subscriber to unregister was not registered before: " + subscriber.getClass());
            }
    }
    
#### 线程调度

1， ```final class HandlerPoster extends Handler ``` 实现主线程的调度

2， ```final class BackgroundPoster implements Runnable ``` 实现子线程的调度

3， ```class AsyncPoster implements Runnable ``` 实现一个子线程单独执行一个事件的调度