---
title: Android内存分析工具
urlname: Android内存分析工具
date: 2022-04-03 12:09:42
tags: Android
categories: Android
description: Android内存分析工具LeakCanary的理解
---

#### 一、Activity生命周期监控

调用`Application#registerActivityLifecycleCallbacks(ActivityLifecycleCallbacks callback)`监控Activity的生命周期。

```java
public final class ActivityRefWatcher {
  
  //生命周期监听
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
  
  public void watchActivities() {
    // Make sure you don't get installed twice.
    stopWatchingActivities();
    application.registerActivityLifecycleCallbacks(lifecycleCallbacks);
  }

  public void stopWatchingActivities() {
    application.unregisterActivityLifecycleCallbacks(lifecycleCallbacks);
  }
}
```

然后会调用`RefWatcher.watch(Object watchedReference)`。

流程简要图：

![leak_canary_memory_leak](/images/leak_canary_memory_leak.png)

<center><font size=2  color=gray>图1：LeakCanary内存泄漏检测图</font></center>



​	核心代码如下：

```java
public final class RefWatcher {
  ...
    
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

  @SuppressWarnings("ReferenceEquality") // Explicitly checking for named null.
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

  private boolean gone(KeyedWeakReference reference) {
    return !retainedKeys.contains(reference.key);
  }

  private void removeWeaklyReachableReferences() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    KeyedWeakReference ref;
    while ((ref = (KeyedWeakReference) queue.poll()) != null) {
      retainedKeys.remove(ref.key);
    }
  }
}
```

####  二、任务调度框架AndroidWatchExecutor

核心代码RefWatcher#ensureGone()是通过WatchExecutor负责调度执行的。

具体就是当主线程处于空闲状态时，往HandlerThread的Handler中postDelayed一个延时任务，默认5s后执行。

如果任务执行失败、需要重试时采用**指数退避**法，即延时时间5, 10, 20, 40, ..., 依次递增。

```java
public final class AndroidWatchExecutor implements WatchExecutor {
  @Override 
  public void execute(Retryable retryable) {
    if (Looper.getMainLooper().getThread() == Thread.currentThread()) {
      waitForIdle(retryable, 0);
    } else {
      postWaitForIdle(retryable, 0);
    }
  }
  
  void waitForIdle(final Retryable retryable, final int failedAttempts) {
    // This needs to be called from the main thread.
    Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
      @Override
      public boolean queueIdle() {
        postToBackgroundWithDelay(retryable, failedAttempts);
        return false;
      }
    });
  }
  
  void postToBackgroundWithDelay(final Retryable retryable, final int failedAttempts) {
    long exponentialBackoffFactor = (long) Math.min(Math.pow(2, failedAttempts), maxBackoffFactor);
    //delayMilles以指数退避依次递增.
    long delayMillis = initialDelayMillis * exponentialBackoffFactor;
    backgroundHandler.postDelayed(new Runnable() {
      @Override
      public void run() {
        Retryable.Result result = retryable.run();
        if (result == RETRY) {
          postWaitForIdle(retryable, failedAttempts + 1);
        }
      }
    }, delayMillis);
  }
}
```



