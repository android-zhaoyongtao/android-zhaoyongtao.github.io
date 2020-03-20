---
layout:     post
title:      开源框架剖析-内存leakcanary
subtitle:   实时监测内存泄露
date:       2020-03-9
author:     ZYT
header-img: img/post-bg-offer.jpeg
catalog: true
tags:
    - Android
    - Offer
    - 内存泄露
    - 开源框架剖析
    - leakcanary
---

# 开源框架剖析-内存leakcanary

内存泄露对外不可见
leakcanary是Square开源的一款轻量第三方内存泄露检测工具，原理是监控一个即将要销毁的对象

### 内存泄露主要发生的部位有

1，栈stack
基本类型，对象引用

2，堆heap

3，方法区method

### 为什么会有内存泄露

1，当一个对象已经不需要再使用了

2，有些对象只有有限的生命周期

### 内存泄露会导致什么问题

OOM，程序崩溃

### 常见的内存泄露

1，单例造成的内存泄露

    单例生命周期和应用一样长，如果持有一个Activity类型的Context，如果activity退出后，就造成无法回收

    解决办法：弄一个Application的Context。context.getApplicationContext()

2，非静态内部类创建静态实例

    此时肥静态内部类对象持有外部类的引用，导致外部类不能释放

3，Handler造成的内存泄露。handler，Message，MessageQueue
    
    TLS，Handler生命周期和Activity不一致。Message持有handler，handler持有Activity，从而导致内存泄露
    
    解决办法：将Handler声明为静态，从而将handler和activity生命周期解绑；通过弱引用方式引入Activity；activity销毁时移除handler回调

4，子线程造成的内存泄露

    定义子线程时，没有使用静态内部类导致持有外部类引用
   
    解决办法：定义成静态的，从而不持有外部；在销毁时，取消掉任务。
    
5，WebView造成的内存泄露

    WebView加载Html页面，会申请Native堆内存来保存页面，内存占用严重。关闭WebView也不能释放。
    
    解决办法：将WebView放在一个单独进程，当不再使用时，killProgress杀死进程
    
6，动画导致的内存泄露

### LeakCanary原理

1，Activity Destory之后将它放在一个WeakReference

2，将这个WeakReference关联到一个ReferenceQueue

3，查看ReferenceQueue是否存在Activity的引用

4，如果该Activity泄露了，Dump出堆Heap信息，然后再去分析泄露路径。本质上还是用命令控制生成hprof文件分析检查内存泄露，然后发送通知。

软引用/弱引用的对象被垃圾回收，Java虚拟机就会把这个引用加入到与之关联的引用队列中。

### LeakCanary如何检测内存泄露
1，首先创建一个refwatcher，启动一个ActivityRefWatcher

```
Application
     LeakCanary.install()
最终调用到
ActivityRefWatcher
    public void watchActivities() {
    // Make sure you don't get installed twice.
    stopWatchingActivities();
    application.registerActivityLifecycleCallbacks(lifecycleCallbacks);
  }
```
2，通过ActivityLifecycleCallbacks把Activity的onDestory生命周期管理
```  
ActivityLifecycleCallbacks
     @Override 
     public void onActivityDestroyed(Activity activity) {
          ActivityRefWatcher.this.onActivityDestroyed(activity);
     }
     
ActivityRefWatcher
    void onActivityDestroyed(Activity activity) {
        refWatcher.watch(activity);
    }
    
```
3，最后在线程池中区开始分析内存泄露
```
RefWatcher
    public void watch(Object watchedReference, String referenceName) {
        ...
        final KeyedWeakReference reference =//这是一个弱引用
            new KeyedWeakReference(watchedReference, key, referenceName, queue);
        ensureGoneAsync(watchStartNanoTime, reference);
    } 
    
    private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
        watchExecutor.execute(new Retryable() {
              @Override public Retryable.Result run() {
                return ensureGone(reference, watchStartNanoTime);
              }
        });
    }
    
    //切换到子线程了
    Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
        long gcStartNanoTime = System.nanoTime();
        long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);
    
        removeWeaklyReachableReferences();
    
        if (debuggerControl.isDebuggerAttached()) {
          // The debugger can create false leaks.
          return RETRY;
        }
        if (gone(reference)) {
          return DONE;
        }
        gcTrigger.runGc();
        removeWeaklyReachableReferences();
        if (!gone(reference)) {//GC后还是没被回收
          long startDumpHeap = System.nanoTime();
          long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);
    
          File heapDumpFile = heapDumper.dumpHeap();
          if (heapDumpFile == RETRY_LATER) {
            // Could not dump the heap.
            return RETRY;
          }
          long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
          heapdumpListener.analyze(//去分析内存泄露
              new HeapDump(heapDumpFile, reference.key, reference.name, excludedRefs, watchDurationMs,
                  gcDurationMs, heapDumpDurationMs));
        }
        return DONE;
  }
  
ServiceHeapDumpListener
    public void analyze(HeapDump heapDump) {
        HeapAnalyzerService.runAnalysis(context, heapDump, listenerServiceClass);
    }
    
public final class HeapAnalyzerService extends IntentService 
    public static void runAnalysis(Context context, HeapDump heapDump, Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
        Intent intent = new Intent(context, HeapAnalyzerService.class);
        intent.putExtra(LISTENER_CLASS_EXTRA, listenerServiceClass.getName());
        intent.putExtra(HEAPDUMP_EXTRA, heapDump);
        context.startService(intent);
    }
    
    @Override
    protected void onHandleIntent(Intent intent) {
        if (intent == null) {
          CanaryLog.d("HeapAnalyzerService received a null intent, ignoring.");
          return;
        }
        String listenerClassName = intent.getStringExtra(LISTENER_CLASS_EXTRA);
        HeapDump heapDump = (HeapDump) intent.getSerializableExtra(HEAPDUMP_EXTRA);
    
        HeapAnalyzer heapAnalyzer = new HeapAnalyzer(heapDump.excludedRefs);
        
        //去走checkForLeak方法
        AnalysisResult result = heapAnalyzer.checkForLeak(heapDump.heapDumpFile, heapDump.referenceKey);
        AbstractAnalysisResultService.sendResultToListener(this, listenerClassName, heapDump, result);
    }
```
4，checkForLeak方法，是最重要的一个方法，主要做以下操作

    1），解析hprof转为Snapshot内存快照
    
    2），优化gcroots
    
    3），找出泄露的对象findLeakingReference/找出泄露对象的最短路径findLeakTrace
```
HeapAnalyzer
    public AnalysisResult checkForLeak(File heapDumpFile, String referenceKey) {
        long analysisStartNanoTime = System.nanoTime();
        if (!heapDumpFile.exists()) {
          Exception exception = new IllegalArgumentException("File does not exist: " + heapDumpFile);
          return failure(exception, since(analysisStartNanoTime));
        }
    
        try {
          //1，解析hprof转为Snapshot内存快照
          HprofBuffer buffer = new MemoryMappedFileBuffer(heapDumpFile);
          HprofParser parser = new HprofParser(buffer);
          Snapshot snapshot = parser.parse();
          //2，优化gcroots
          deduplicateGcRoots(snapshot);
    
          //找出泄露的对象findLeakingReference方法
          Instance leakingRef = findLeakingReference(referenceKey, snapshot);
    
          // False alarm, weak reference was cleared in between key check and heap dump.
          if (leakingRef == null) {
            return noLeak(since(analysisStartNanoTime));
          }
          //找出泄露对象的最短路径findLeakTrace
          return findLeakTrace(analysisStartNanoTime, snapshot, leakingRef);
        } catch (Throwable e) {
          return failure(e, since(analysisStartNanoTime));
        }
    }
```

