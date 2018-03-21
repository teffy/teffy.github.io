---
title: read_fucking_source_code_leakcanary
date: 2018-02-09 11:38:54
categories: 
- read_fucking_source_code
tags: 
- read_fucking_source_code
- leakcanary
---

本文主要分享leakcanary的一些核心代码和一些流程分析。

<!-- more -->

Read fucking source code:leakcanary

## 关键代码
```
com.squareup.leakcanary.LeakCanary#install
  public static RefWatcher install(Application application) {
    return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
        .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
        .buildAndInstall();
  }
  public static AndroidRefWatcherBuilder refWatcher(Context context) {
    return new AndroidRefWatcherBuilder(context);
  }
  public RefWatcher buildAndInstall() {
    RefWatcher refWatcher = build();
    if (refWatcher != DISABLED) {
      LeakCanary.enableDisplayLeakActivity(context);
      ActivityRefWatcher.install((Application) context, refWatcher); ==》com.squareup.leakcanary.ActivityRefWatcher#install
    }
    return refWatcher;
  }
```

```
 public static void install(Application application, RefWatcher refWatcher) {
    new ActivityRefWatcher(application, refWatcher).watchActivities();
  }

  private final Application.ActivityLifecycleCallbacks lifecycleCallbacks =
      new Application.ActivityLifecycleCallbacks() {
        @Override public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
        }

        @Override public void onActivityStarted(Activity activity) {
        }

        @Override public void onActivityResumed(Activity activity) {
        }

        @Override public void onActivityPaused(Activity activity) {
        }

        @Override public void onActivityStopped(Activity activity) {
        }

        @Override public void onActivitySaveInstanceState(Activity activity, Bundle outState) {
        }

        @Override public void onActivityDestroyed(Activity activity) {
          ActivityRefWatcher.this.onActivityDestroyed(activity);
        }
      };
 void onActivityDestroyed(Activity activity) {
    refWatcher.watch(activity);
  }
```
```
  public void watch(Object watchedReference, String referenceName) {
    if (this == DISABLED) {
      return;
    }
    checkNotNull(watchedReference, "watchedReference");
    checkNotNull(referenceName, "referenceName");
    final long watchStartNanoTime = System.nanoTime();
    String key = UUID.randomUUID().toString();
    retainedKeys.add(key);
    final KeyedWeakReference reference =
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
```
```
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
    if (!gone(reference)) {
      long startDumpHeap = System.nanoTime();
      long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);

      File heapDumpFile = heapDumper.dumpHeap();
      if (heapDumpFile == RETRY_LATER) {
        // Could not dump the heap.
        return RETRY;
      }
      long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
      heapdumpListener.analyze(
          new HeapDump(heapDumpFile, reference.key, reference.name, excludedRefs, watchDurationMs,
              gcDurationMs, heapDumpDurationMs));
    }
    return DONE;
  }
```
```
 Override public void execute(Retryable retryable) {
    if (Looper.getMainLooper().getThread() == Thread.currentThread()) {
      waitForIdle(retryable, 0);
    } else {
      postWaitForIdle(retryable, 0);
    }
  }
```
```
  void postWaitForIdle(final Retryable retryable, final int failedAttempts) {
    mainHandler.post(new Runnable() {
      @Override public void run() {
        waitForIdle(retryable, failedAttempts);
      }
    });
  }
```
```
  void waitForIdle(final Retryable retryable, final int failedAttempts) {
    // This needs to be called from the main thread.
    Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
      @Override public boolean queueIdle() {
        postToBackgroundWithDelay(retryable, failedAttempts);
        return false;
      }
    });
  }
```
```
  void postToBackgroundWithDelay(final Retryable retryable, final int failedAttempts) {
    long exponentialBackoffFactor = (long) Math.min(Math.pow(2, failedAttempts), maxBackoffFactor);
    long delayMillis = initialDelayMillis * exponentialBackoffFactor;
    backgroundHandler.postDelayed(new Runnable() {
      @Override public void run() {
        Retryable.Result result = retryable.run();
        if (result == RETRY) {
          postWaitForIdle(retryable, failedAttempts + 1);
        }
      }
    }, delayMillis);
  }
```
## 步骤分析
1、refWatcher(application) 返回一个AndroidRefWatcherBuilder建造者
2、.listenerServiceClass(DisplayLeakService.class) 指定分析Heapdump的listener
3、.excludedRefs(AndroidExcludedRefs.createAppDefaults().build()) 排除一些已知的Android系统以及各个手机场数系统内存泄露的问题
4、buildAndInstall() 创建出RefWatcher，然后ActivityRefWatcher.install((Application) context, refWatcher); 设置Application.ActivityLifecycleCallbacks，监听Activity的生命周期回调，在onActivityDestroyed方法中开始观察这个activity
5、refWatcher.watch(activity);为这个activity生成一个UUID，加入缓存keyset中，创建一个KeyedWeakReference，使用同一个ReferenceQueue
6、ensureGoneAsync方法用于确认activity有没有被正常回收，com.squareup.leakcanary.AndroidWatchExecutor#execute，
7、
```
  mainHandler.post(new Runnable() {
      @Override public void run() {
        waitForIdle(retryable, failedAttempts);
      }
    }); 
```
使用handler把处理Looper.myQueue().addIdleHandler的任务加进messagequeue中，
8、Looper.myQueue().addIdleHandler 添加一个监听MessageQueue空闲等待新的message的监听，然后在
```
  public boolean queueIdle() {
        postToBackgroundWithDelay(retryable, failedAttempts);
  }
```
处理对于reference的检测处理
9、
```
Retryable.Result result = retryable.run();
```
==>
```
com.squareup.leakcanary.RefWatcher#ensureGoneAsync
```
==>
```
com.squareup.leakcanary.Retryable#run
```
==>
```
com.squareup.leakcanary.RefWatcher#ensureGone
```
10、ensureGone方法中，第一次removeWeaklyReachableReferences(),检测ReferenceQueue中有已经标记要gc但还没有进行gc的对象，那么就从第5步骤中keyset删除上次的UUID；
retryJava 在debug的时候的情况，不需要去dumpheap
```
   gcTrigger.runGc(); 执行一次gc
   Runtime.getRuntime().gc();
      enqueueReferences();
      System.runFinalization();
```
gone(reference))方法用来确认这次要处理的reference是否在removeWeaklyReachableReferences()的时候已经删除了key，
11、如果已经删除了说明刚才已经标记要gc掉，那么就return DONE结束了，
12、否则就要开始处理heap, File heapDumpFile = heapDumper.dumpHeap();先将内存dump出来存进文件，
13、dumpHeap过程使用了CountDownLatch弹出一个toast，弹出toast之后给looper添加IdleHandler，在looper空闲时，把toast设置给FutureResult，然后回到dumpHeap()过程中，调用Debug.dumpHprofData(heapDumpFile.getAbsolutePath());开始dump内存到hprof文件中去
14、然后调用之前设置的listener进行分析，heapdumpListener.analyze==》com.squareup.leakcanary.HeapAnalyzer#checkForLeak，分析过程，


## 知识拓展
CountDownLatch都有哪些使用场景。
1、实现最大的并行性：有时我们想同时启动多个线程，实现最大程度的并行性。例如，我们想测试一个单例类。如果我们创建一个初始计数为1的CountDownLatch，并让所有线程都在这个锁上等待，那么我们可以很轻松地完成测试。我们只需调用 一次countDown()方法就可以让所有的等待线程同时恢复执行。
2、开始执行前等待n个线程完成各自任务：例如应用程序启动类要确保在处理用户请求前，所有N个外部系统已经启动和运行了。
3、死锁检测：一个非常方便的使用场景是，你可以使用n个线程访问共享资源，在每次测试阶段的线程数目是不同的，并尝试产生死锁。
[这篇文章分析的不错](http://www.importnew.com/15731.html)




 