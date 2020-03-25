---
layout:     post
title:      开源框架剖析-图片加载Picasso
subtitle:   
date:       2020-03-8
author:     ZYT
header-img: img/post-bg-offer.jpeg
catalog: true
tags:
    - Android
    - Offer
    - 图片加载
    - 开源框架剖析
    - Picasso
---

# 开源框架剖析-图片加载Picasso

Picasso是一个强大的图片加载缓存框架

### 使用流程

    Picasso.get()
                .load(path)
                .placeholder(R.drawable.block_canary_icon)
                .error(R.drawable.block_canary_icon)
                .resize(480,480)//
                .centerCrop()
                .rotate(360)
                .priority(Picasso.Priority.HIGH)
                .tag("tage1")//可以用了暂停图片请求
                .memoryPolicy(MemoryPolicy.NO_CACHE)//内存缓存
                .networkPolicy(NetworkPolicy.NO_CACHE)//磁盘缓存
                .into(imageView);
                

## Picasso核心类/方法
![](https://tva1.sinaimg.cn/large/00831rSTly1gd56zk2ajdj30hs0edt9v.jpg)
### Picasso.get()方法分析

主要是基础配置
```
public static Picasso get() {
        if (singleton == null) {
              synchronized (Picasso.class) {
                    if (singleton == null) {
                          if (PicassoProvider.context == null) {
                            throw new IllegalStateException("context == null");
                          }
                          singleton = new Builder(PicassoProvider.context).build();
                    }
              }
        }
        return singleton;
}

class Builder {
        public Picasso build() {
              Context context = this.context;
        
              if (downloader == null) {
                downloader = new OkHttp3Downloader(context);
              }
              if (cache == null) {
                cache = new LruCache(context);
              }
              if (service == null) {
                service = new PicassoExecutorService();
              }
              if (transformer == null) {
                transformer = RequestTransformer.IDENTITY;
              }
        
              Stats stats = new Stats(cache);
        
              Dispatcher dispatcher = new Dispatcher(context, service, HANDLER, downloader, cache, stats);
        
              return new Picasso(context, dispatcher, cache, listener, transformer, requestHandlers, stats,
                  defaultBitmapConfig, indicatorsEnabled, loggingEnabled);
        }
} 

class Dispatcher {//重要的分发器类
    Dispatcher(Context context, ExecutorService service, Handler mainThreadHandler, Downloader downloader, Cache cache, Stats stats) {
        this.dispatcherThread = new DispatcherThread();//一个HandlerThread
        this.dispatcherThread.start();
        Utils.flushStackLocalLeaks(dispatcherThread.getLooper());
        this.context = context;
        this.service = service;//线程池，支持自定义
        this.hunterMap = new LinkedHashMap<>();//BitmapHunter实现了Runnable接口，这个才是网络请求核心类，实现了下载，解码，对bitmap进行
        this.failedActions = new WeakHashMap<>();//保存了失败的action
        this.pausedActions = new WeakHashMap<>();//保存了暂停的action
        this.pausedTags = new LinkedHashSet<>();////暂停tag
        ////自己内部的handler，分发请求线程的时候，就是通过这个自己线程内部的handler进行分发
        this.handler = new DispatcherHandler(dispatcherThread.getLooper(), this);//子线程handler
        this.downloader = downloader;//下载接口
        this.mainThreadHandler = mainThreadHandler;//主线程handler
        this.cache = cache;
        this.stats = stats;
        this.batch = new ArrayList<>(4);
        this.airplaneMode = Utils.isAirplaneModeOn(this.context);
        this.scansNetworkChanges = hasPermission(context, Manifest.permission.ACCESS_NETWORK_STATE);
        this.receiver = new NetworkBroadcastReceiver(this);//网络变化广播
        receiver.register();
  }
}
```
在此，插入一个HandlerThread（DispatcherThread）的简介

![](https://tva1.sinaimg.cn/large/00831rSTly1gd58hyj7oaj31au0l4n1g.jpg)
```
//Picasso的构造方法
Picasso(Context context, Dispatcher dispatcher, Cache cache, Listener listener,
      RequestTransformer requestTransformer, List<RequestHandler> extraRequestHandlers, Stats stats,
      Bitmap.Config defaultBitmapConfig, boolean indicatorsEnabled, boolean loggingEnabled) {
    this.context = context;
    this.dispatcher = dispatcher;//dispatcher是负责分发请求的
    this.cache = cache;//picasso的缓存机制
    this.listener = listener;
    this.requestTransformer = requestTransformer;
    this.defaultBitmapConfig = defaultBitmapConfig;

    int builtInHandlers = 7; // Adjust this as internal handlers are added or removed.
    int extraCount = (extraRequestHandlers != null ? extraRequestHandlers.size() : 0);
    //通过不同的requesthandler来处理不同类型的图片
    List<RequestHandler> allRequestHandlers = new ArrayList<>(builtInHandlers + extraCount);

    // ResourceRequestHandler needs to be the first in the list to avoid
    // forcing other RequestHandlers to perform null checks on request.uri
    // to cover the (request.resourceId != 0) case.
    allRequestHandlers.add(new ResourceRequestHandler(context));
    if (extraRequestHandlers != null) {
      allRequestHandlers.addAll(extraRequestHandlers);
    }
    allRequestHandlers.add(new ContactsPhotoRequestHandler(context));
    allRequestHandlers.add(new MediaStoreRequestHandler(context));
    allRequestHandlers.add(new ContentStreamRequestHandler(context));
    allRequestHandlers.add(new AssetRequestHandler(context));
    allRequestHandlers.add(new FileRequestHandler(context));
    allRequestHandlers.add(new NetworkRequestHandler(dispatcher.downloader, stats));//
    requestHandlers = Collections.unmodifiableList(allRequestHandlers);

    this.stats = stats;
    this.targetToAction = new WeakHashMap<>();
    this.targetToDeferredRequestCreator = new WeakHashMap<>();
    this.indicatorsEnabled = indicatorsEnabled;
    this.loggingEnabled = loggingEnabled;
    this.referenceQueue = new ReferenceQueue<>();
    this.cleanupThread = new CleanupThread(referenceQueue, HANDLER);
    this.cleanupThread.start();
  }
```
get()方法就是一个DoubleCheck的单例模式。但他的单例创建是通过Builder方式做的。

与前期版本不同的是Picasso需要的Context是通过PicassoProvider的context获取的，省掉了手动初始化。

### Picasso.load()方法分析

最主要的就是创建了RequestCreator对象。Request.Builder能配置更多的参数.
在into之前都是对请求的描述，设置各种属性。然后生成了Request。
```
Picasso
    public RequestCreator load(@Nullable Uri uri) {
        return new RequestCreator(this, uri, 0);
    }

RequestCreator
    RequestCreator(Picasso picasso, Uri uri, int resourceId) {
    this.picasso = picasso;
    this.data = new Request.Builder(uri, resourceId, picasso.defaultBitmapConfig);
}

```

### Picasso.into()方法分析
```
class RequestCreator
        public void into(ImageView target, Callback callback) {
            ...
            Action action =
                new ImageViewAction(picasso, target, request, memoryPolicy, networkPolicy, errorResId,
                    errorDrawable, requestKey, tag, callback, noFade);
        
            picasso.enqueueAndSubmit(action);
        }
```
最后提交的是一个Action，Action也是一个抽象类，会根据我们不同的请求生成不同的action子类，他包含了请求信息，回调接口，picasso的实例。如果是一次要加载多个图片，那么会产生多个request和多个Aciton。
```
class Picasso
        void enqueueAndSubmit(Action action) {
            submit(action);
        }

        void submit(Action action) {
            dispatcher.dispatchSubmit(action);
        }
```
接下来sbumit之后的操作，是通过这个Dispatcher分发
```
class Dispatcher
    //这就是dispatcher类中的submit方法，使用线程自己的handler
    void dispatchSubmit(Action action) {
        handler.sendMessage(handler.obtainMessage(REQUEST_SUBMIT, action));
    }
```
消息的处理在DispatcherHandler的handleMessage方法中，又调用的Dispatcher的performSubmit(action)方法
```
class Dispatcher
    //主要是获取到真正需要处理的bitmaphunter
    void performSubmit(Action action, boolean dismissFailed) {
            ///检测暂停的请求是否包含此次请求，如果包含，将请求保存到map，返回
            if (pausedTags.contains(action.getTag())) {
              pausedActions.put(action.getTarget(), action);
              return;
            }
            //检测已经提交的请求是否包含此次请求，key就是根据url或者resourceid
            //这样的字段生成，如果已经包含了同样的请求，那么直接合并到同一个hunter中去。
            //bitmaphunter实现了runnable接口，这个类是最终请求网路并进行编码生成bitmap的类
            BitmapHunter hunter = hunterMap.get(action.getKey());
            if (hunter != null) {
              hunter.attach(action);
              return;
            }
        
            if (service.isShutdown()) {//shutdown的就不进行了
              return;
            }
        
            //在这里forRequest方法生成一个bitmaphunter
            hunter = forRequest(action.getPicasso(), this, cache, stats, action);
            //获取返回的结果
            hunter.future = service.submit(hunter);
            //保存到map中
            hunterMap.put(action.getKey(), hunter);
            if (dismissFailed) {
              failedActions.remove(action.getTarget());
            }
     }
     
 
 ```
forRequest方法的核心在于，根据request的不同，选择匹配的requesthandler。
requestHandler的核心方法是load。加载网络图片用的是NetworkRequestHandler。
对于多个requestHandler采用一个for循环方式的调用，直到返回正确结果才return，可以看做一种责任链。

service执行了submit之后，将会执行bitmaphunter中的run方法。
```
BitmapHunter
        @Override public void run() {
                  ...
                  result = hunt();
                  if (result == null) {
                        dispatcher.dispatchFailed(this);
                  } else {
                        dispatcher.dispatchComplete(this);
                  }
                  ...
        }
        
        Bitmap hunt() throws IOException {
                Bitmap bitmap = null;
            
                //是否是内存模式读取
                if (shouldReadFromMemoryCache(memoryPolicy)) {
                  bitmap = cache.get(key);
                  if (bitmap != null) {
                    stats.dispatchCacheHit();
                    loadedFrom = MEMORY;
                    if (picasso.loggingEnabled) {
                      log(OWNER_HUNTER, VERB_DECODED, data.logId(), "from cache");
                    }
                    return bitmap;
                  }
                }
                
                networkPolicy = retryCount == 0 ? NetworkPolicy.OFFLINE.index : networkPolicy;
                RequestHandler.Result result = requestHandler.load(data, networkPolicy);
                if (result != null) {
                  loadedFrom = result.getLoadedFrom();
                  exifOrientation = result.getExifOrientation();
                  bitmap = result.getBitmap();
            
                  // If there was no Bitmap then we need to decode it from the stream.
                  if (bitmap == null) {
                    Source source = result.getSource();
                    try {
                      bitmap = decodeStream(source, data);
                    } finally {
                      try {
                        //noinspection ConstantConditions If bitmap is null then source is guranteed non-null.
                        source.close();
                      } catch (IOException ignored) {
                      }
                    }
                  }
                }
            
                if (bitmap != null) {
                  if (picasso.loggingEnabled) {
                    log(OWNER_HUNTER, VERB_DECODED, data.logId());
                  }
                  stats.dispatchBitmapDecoded(bitmap);
                  if (data.needsTransformation() || exifOrientation != 0) {
                    synchronized (DECODE_LOCK) {
                      if (data.needsMatrixTransform() || exifOrientation != 0) {
                        bitmap = transformResult(data, bitmap, exifOrientation);
                        if (picasso.loggingEnabled) {
                          log(OWNER_HUNTER, VERB_TRANSFORMED, data.logId());
                        }
                      }
                      if (data.hasCustomTransformations()) {
                        bitmap = applyCustomTransformations(data.transformations, bitmap);
                        if (picasso.loggingEnabled) {
                          log(OWNER_HUNTER, VERB_TRANSFORMED, data.logId(), "from custom transformations");
                        }
                      }
                    }
                    if (bitmap != null) {
                      stats.dispatchBitmapTransformed(bitmap);
                    }
                  }
                }
            
                return bitmap;
        }
```
主要是通过hunt中的load方法获取到了需要的bitmap，然后查看各种属性配置，是否需要对bitmap做额外的处理等。
处理完之后，通过dispatcher将结果分发出去。
最终执行到 ```mainThreadHandler.sendMessage```，然后主线程处理完分发过来的消息，整个流程就算结束了。
