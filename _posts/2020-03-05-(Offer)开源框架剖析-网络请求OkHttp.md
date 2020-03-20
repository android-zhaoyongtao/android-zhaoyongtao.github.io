---
layout:     post
title:      开源框架剖析-网络请求OkHttp
subtitle:   okhttp
date:       2020-03-5
author:     ZYT
header-img: img/post-bg-offer.jpeg
catalog: true
tags:
    - Android
    - Offer
    - 网络请求
    - 开源框架剖析
    - okhttp
---

# 开源框架剖析-网络请求OkHttp

### 使用流程
```
    OkHttpClient client = new OkHttpClient.Builder().build();
    Request request = new Request.Builder().build();
    Call call = client.newCall(request);
    call.execute()/enqueue(callback)
```
        
1，创建OkhttpClient和Request对象
均使用了Builder模式

2，将Request封装成Call对象

3，同步调用Call.execute()，同步调用Call.enqueue()

![](https://tva1.sinaimg.cn/large/00831rSTly1gcvypgb993j30rs18gdhq.jpg)

### 核心类
#### Dispatcher
维护同步、异步请求状态，维护一个线程池

readyAsyncCalls就绪等待异步请求

runningAsyncCalls正在执行的任务请求（包含已经取消但没有执行完的)

runningSyncCalls同步请求队列

executorService线程池

异步请求为什么需要两个队列？生产消费模型

Dispatcher的finish方法在执行的finally中执行，主要做3件事
1）队列移除当前请求，2）调整请求任务队列，3）重新计算正在进行线程数量

#### Okhttp拦截器链getResponseWithInterceptorChain

可以实现网络监听、请求重写及相应重写、请求失败重试等功能

##### 实现步骤：
1，创建一些列拦截器,放入一个拦截器list中

2，创建一个拦截器链对象RealInterceptorChain，并执行拦截器的proceed方法(核心创建下一个拦截器链，内部会执行下一个拦截器的interceptor方法)

##### 内部提供拦截器有：

###### 1，RetryAndFollowUpInterceptor
    重试和失败重定向
    
步骤：
    
创建StreanAllocation对象

调用RealInterceptorChain.proceed()进行网络请求

根据异常结果或者相应结果判断是否需要进行重新请求

调用下一个拦截器，对response进行处理，返回给上一个拦截器

###### 2，BridgeInterceptor
    桥接适配。内容长度、编码方式、压缩
    
步骤：

负责将用户构建好的Request请求转化为能够进行网络访问的请求

将这个符合网络请求的Request进行网络请求

将网络请求返回的Response转化为用户库尔用的Response

###### 3，CacheInterceptor
    缓存策略使用的Cache类的put\get方法
    
步骤：

取缓存，然后做判断能不能用

如果不能用调用proceed方法进入下一个拦截器的intercept方法

intercept方法返回response经一系列判断后决定要不要存入缓存

###### 4，ConnectInterceptor
    建立可靠连接
    
步骤：

弄一个RealConnection对象，
ConnectInterceptor获取Interceptor传过来的**StreanAllocation**,streamAllocation.newStream()

选择不同的连接方式
streamAllocation.connection();

将上边这些参数传递给后边的拦截器

###### 5，CallServerInterceptor
    负责将请求写进网络io流，同时读取返回的io
    
![](https://tva1.sinaimg.cn/large/00831rSTly1gcvyq5553hj3132075ab1.jpg)




