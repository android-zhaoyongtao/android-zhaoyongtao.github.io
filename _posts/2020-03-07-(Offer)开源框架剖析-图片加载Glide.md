---
layout:     post
title:      开源框架剖析-图片加载Glide
subtitle:   Glide，google官方推荐框架
date:       2020-03-7
author:     ZYT
header-img: img/post-bg-offer.jpeg
catalog: true
tags:
    - Android
    - Offer
    - 图片加载
    - 开源框架剖析
    - Glide
---

# 开源框架剖析-图片加载Glide

### Glide的几个基本概念
1，Model 模型概念，表示数据来源
可以是：图片url、资源ID、文件类型

2，Data
原始数据

3，Resourse 
解码之后的资源

4，TransformedResource
裁剪等转换后的资源

5，TranscededResource
转码完成后的资源

6，Target
需要显示的图片封装为Target

![](https://tva1.sinaimg.cn/large/00831rSTly1gcxwbmad3fj31xi0cw116.jpg)

### 使用流程
```
Glide.with(this).load(url).into(imageView);

Glide.with(getApplicationContext())
                .load("url")
                .placeholder(R.mipmap.ic_launcher)
                .error(R.mipmap.ic_launcher)
                .override(300, 300)//指定图片尺寸
                .fitCenter()//指定图片缩放类型
                .centerCrop()//指定图片缩放类型
                .skipMemoryCache(true)//跳过内存缓存
                .crossFade(1000)//设置渐变式显示的时间
                .diskCacheStrategy(DiskCacheStrategy.NONE)//磁盘缓存策略-跳过磁盘
                .diskCacheStrategy(DiskCacheStrategy.SOURCE)//只缓存原来全分辨率
                .diskCacheStrategy(DiskCacheStrategy.RESULT)//只缓存最终
                .diskCacheStrategy(DiskCacheStrategy.ALL)//缓存所有版本
                .priority(Priority.HIGH)//指定优先级
                .into(imageView);

```

## Glide核心类/方法
### Glide.with()方法分析
是Glide类中的一组静态方法，with()方法的重载很多，可以传入Activit、Fragment或者是Context。每一个with()方法都是先调用RequestManagerRetriever的静态get()方法得到一个
RequestManagerRetriever对象，然后再调用RequestManagerRetriever的实例get()方法，去获取RequestManager对象。
```
public class Glide {
    public static RequestManager with(Context context) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(context);
    }
}

public class RequestManagerRetriever implements Handler.Callback {
    public RequestManager get(Context context) {
        if (context == null) {
            throw new IllegalArgumentException("You cannot start a load on a null Context");
        } else if (Util.isOnMainThread() && !(context instanceof Application)) {
            if (context instanceof FragmentActivity) {
                return get((FragmentActivity) context);
            } else if (context instanceof Activity) {
                return get((Activity) context);
            } else if (context instanceof ContextWrapper) {
                return get(((ContextWrapper) context).getBaseContext());
            }
        }
        return getApplicationManager(context);
    }
    
     public RequestManager get(FragmentActivity activity) {
        if (Util.isOnBackgroundThread()) {
            return get(activity.getApplicationContext());
        } else {
            assertNotDestroyed(activity);
            FragmentManager fm = activity.getSupportFragmentManager();
            return supportFragmentGet(activity, fm);
        }
    }
}

```

get方法会判断传入的Context的类型，如果是Application或者是在子线程加载，这会通过getApplicationManager方法单例形式创建一个RequestManager对象。

否则会调用fragmentGet方法，传入Context对象和FragmentManager对象，最终会向当前的Activity当中添加一个隐藏的Fragment从而实现生命周期的监听

Application类型的将不包含生命周期的监听

##### RequestManager

通过Glide.with(this)得到，它的主要作用是管理我们的图片加载请求，可以根据我们传进来的参数，获取组件生命周期监听

一个RequestManager对应一个Fragment
##### RequestManagerRetirever

通过往get()方法中输入不同Context、Fragment从而输出RequestManager

当输入activity、fragment时，创建一个空的无界面的RequestManagerFragment，从而监听生命周期
通过Glide.with(this)得到，它的主要作用是管理我们的图片请求，可以根据我们传进来的参数，组件生命周期
##### RequestManagerFragment
通过lifecycle监听生命周期，让Glide捕捉到，从而进行有效图片加载

### RequestManager.load(xxx)方法分析
有多个重载的方法，多种参数传入类型

![](https://tva1.sinaimg.cn/large/00831rSTly1gcy5tw7skgj30lo0d40ur.jpg)

load内部调用了fromString方法，fromString方法调用了loadGeneric方法，
loadGeneric()方法也没几行代码，这里分别调用了Glide.buildStreamModelLoader()方法和Glide.buildFileDescriptorModelLoader()方法来获得ModelLoader对象。
ModelLoader对象是用于加载图片的，而我们给load()方法传入不同类型的参数，这里也会得到不同的ModelLoader对象，最后返回一个DrawableTypeRequest对象。

DrawableTypeRequest继承DrawableRequestBuilder，继承GenericRequestBuilder，
DrawableRequestBuilder中有很多个方法，这些方法其实就是Glide绝大多数的API了。比如说placeholder()方法、error()方法、diskCacheStrategy()方法、override()方法等，都是在DrawableRequestBuilder 类里面


### GenericRequestBuilder.into()方法分析

其中先构建一个Target

->requestTracker.runRequest(request);执行图片请求

->GenericRequest.begin();

->GenericRequest.onSizeReady(overrideWidth, overrideHeight);

->Engine.load() 任务创建，发起，回调，管理存活和缓存的资源
    读取内存和正在使用的缓存

### Glide特点：

1， 生命周期：图片的加载、GIF图片的播放，可和页面的生命周期一致。可接受Activity、Fragment、FragmentActivity、ApplicationContext。

实现原理：

Glide对每个页面维护了一个单独的RequestManager。

对于每一个Activity或Fragment，在其中添加一个RequestManagerFragment作为子Fragment，其生命周期和父组件Activity或Fragment的生命周期一致，在RequestManagerFragment中onStart、onStop、onDestroy中调用相应方法。

对于ApplicationContext，只调用了onStart方法。

优点： 可自动根据页面生命周期，开始/暂停加载图片、展示动态图片。

缺点： 会消耗更多资源。使用时如果不了解相关特性，容易出错。

2， 相比Fresco，没有使用JNI

优点： 不容易出现JNI相关的错误，配置更容易

缺点： 相比Fresco，性能可能稍差，OOM的概率可能多一点

3， Bitmap解码格式：默认优先使用RGB_565，比ARGB_8888内存占用减少一半，性能好。可全局配置优先使用RGB_565或ARGB_8888，也可对某个请求单独配置。Fresco也可以支持两种编码，而Picasso只支持ARGB_8888。

4， 磁盘缓存策略：默认使用了内存LRU缓存和磁盘LRU缓存。磁盘缓存支持配置缓存全尺寸、转换过的尺寸、两者都保存。可全局配置，或对某个请求单独配置。

Picasso内部只实现了内存LRU缓存，磁盘缓存直接使用了OKHTTP的缓存，只能缓存下载的原始图片，每次从磁盘加载都要转换。

5， 内存缓存策略：使用了两级内存缓存，MemoryCache和ActiveResource，前者默认为一个LruResourceCache，后者是一个Map弱引用，引用了从MemoryCache中读取过的资源和从网络、硬盘下载和转换出的资源。

加载图片时先使用MemoryCache，如果没有找到则尝试从ActiveResource中获取资源。如果还是没有再从磁盘、网络获取资源。

6， BitmapPool：进行Bitmap相关的操作时，对Bitmap进行缓存和复用。默认实现的是LruBitmapPool(仅支持Android 3.0及以上版本)。

7， 网络图片下载：网络图片默认使用HttpURLConnection加载（HttpUrlFetcher），可以通过注册模块的形式，设置成Volley或OkHttp等。

8， 相比Fresco，不需要特定的View，直接使用ImageView即可，通用性好

9， 可以暂停、继续、清除某个页面的RequestManager中所有请求。和Picasso相似（Picasso通过Tag来对Request分组进行操作）。

10， 尺寸适配：默认自动根据图片尺寸加载对应的图片。Picasso则需要显示调用fit()方法。

11， 图片转换：配合glide-transformations，可对图片实现裁剪，着色，模糊，滤镜等效果。

12， 预加载：提供了一个ListPreloader类，可用于AbsListView的预加载

原理：ListPreloader中实现了OnScrollListener，滚动时自动计算并预加载，所加载的Target为PreloadTarget。

13， 加载动态图：支持GIF和本地视频加载，并根据页面生命周期播放/暂停

14， 可自定义ModelLoader，从而指定网络加载库、实现指定格式的文件加载（例如SVG）、实现CDN图片根据URL参数缩放等。

15， 功能强大，因此配置和使用相对复杂，需要先进行充分了解，进行封装。每次发送请求时的流程比较多，性能上有少量损失。

16， 相对Picasso，方法数较多，包的尺寸较大，应使用Proguard进行优化