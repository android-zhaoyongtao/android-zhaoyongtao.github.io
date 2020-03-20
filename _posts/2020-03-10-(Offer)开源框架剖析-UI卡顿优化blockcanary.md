---
layout:     post
title:      UI卡顿优化blockcanary
subtitle:   巧用handler.dispatchMessage实现
date:       2020-03-11
author:     ZYT
header-img: img/post-bg-offer.jpeg
catalog: true
tags:
    - Android
    - Offer
    - UI卡顿优化
    - 开源框架剖析
    - blockcanary
---

# UI卡顿优化blockcanary

项目越来越复杂，代码复杂度也越来越深。如果出现了UI卡顿，很难定位到是哪里的问题。
一个Activity可能包括了很多页面很多回调，怎么找到这些卡顿的元凶呢？
BlockCanary是一个非侵入的性能监控组件，解决UI卡顿

### UI卡顿原理

1，60fps / 16ms/帧

2，尽量在16ms内处理完所有的CPU也Gpu计算、绘制、渲染等操作。否则会UI卡顿

### UI卡顿常见原因

1，UI线程做了耗时操作。

2，Layout过于复杂，无法在16ms完成

3，View过度绘制

4，View频繁触发measure、layout

5，内存频繁触发GC。内存频繁创建变量

### BlockCanary简单使用

1，gradle引入依赖
```
implementation 'com.github.markzhai:blockcanary-android:1.5.0'
```
1，Application中注册
```
BlockCanary.install(this, BlockCanaryContext.get()).start();
```
### BlockCanary原理

android在ActivityThread的main方法定了了mainLooper

在handler.dispatchMessage方法前后计算出时间差，如果超过了阈值，就是dump出堆栈信息、cpu信息、内存信息。

通过往Looper.getMainLooper().setMessageLogging插入一个回调实现的

1，注册、初始化
```
BlockCanary.install(）

BlockCanary{
    public static BlockCanary install(Context context, BlockCanaryContext blockCanaryContext) {
        BlockCanaryContext.init(context, blockCanaryContext);
        setEnabled(context, DisplayActivity.class, BlockCanaryContext.get().displayNotification());
        return get();
    }
    //get()方法调用到构造方法
    private static BlockCanary sInstance;
    private BlockCanaryInternals mBlockCanaryCore;
    private boolean mMonitorStarted = false;

    private BlockCanary() {
        BlockCanaryInternals.setContext(BlockCanaryContext.get());
        mBlockCanaryCore = BlockCanaryInternals.getInstance();
        mBlockCanaryCore.addBlockInterceptor(BlockCanaryContext.get());
        if (!BlockCanaryContext.get().displayNotification()) {
            return;
        }
        mBlockCanaryCore.addBlockInterceptor(new DisplayService());

    }
}   

public final class BlockCanaryInternals {
    LooperMonitor monitor;
    StackSampler stackSampler;
    CpuSampler cpuSampler;
}
```
2，注册、初始化
```
BlockCanary.start()
BlockCanary{
     public void start() {
            if (!mMonitorStarted) {
                mMonitorStarted = true;
                Looper.getMainLooper().setMessageLogging(mBlockCanaryCore.monitor);
            }
    }
}

Looper {
    public static void loop() {
        for (;;) {
                ...
                final Printer logging = me.mLogging;
                if (logging != null) {
                    logging.println(">>>>> Dispatching to " + msg.target + " " +
                            msg.callback + ": " + msg.what);
                }
                
                if (logging != null) {
                    logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
                }
         }
}

LooperMonitor {
     @Override
    public void println(String x) {
            if (mStopWhenDebugging && Debug.isDebuggerConnected()) {
                return;
            }
            if (!mPrintingStarted) {
                mStartTimestamp = System.currentTimeMillis();
                mStartThreadTimestamp = SystemClock.currentThreadTimeMillis();
                mPrintingStarted = true;
                startDump();
            } else {
                final long endTime = System.currentTimeMillis();
                mPrintingStarted = false;
                if (isBlock(endTime)) {
                    notifyBlockEvent(endTime);
                }
                stopDump();
            }
    }
    
    private boolean isBlock(long endTime) {
            return endTime - mStartTimestamp > mBlockThresholdMillis;
    }
}
```



