---
categories: [面试复习,知识点]
title: Choreographer详解
date: 2023-05-20 09:00:00 +0800
last_modified_at:
tags: [转载,复习]
keywords: [面试,Android,choreographer]
---

>转载自：<https://github.com/zhpanvip/AndroidNote>
{: .prompt-info}

## [屏幕刷新机制](/posts/屏幕刷新机制)

## 一、Choreographer概述
1. ViewRootImpl的构造方法中会调用Choreographer的getInstance,并将Choreographer做成成员变量进行保存。

2. Choreographer的getInstance则是从sThreadInstance中取出Choreographer

3. sThreadInstance是一个ThreadLocal类型的静态成员变量，ThreadLocal初始化值实例化了一个Choreographer。

4. Choreographer的构造方法中会实例化一个Handler，然后根据是否使用Vsync来创建FrameDisplayEventReceiver，接着创建一个容量为4的CallbackQueues数组，并且实例化四个Callback存入数组中。

   - CallbackQueues数组用于保存4中不同类型的Callback，分别为输入类型的callback、动画类型的callback、绘制类型的callback以及提交类型的callback
   - Vsync的注册、申请、接收都是通过FrameDisplayEventReceiver实现的，注册之后会监听SurfaceFlinger的传来的Vsync事件，在收到Vsync事件后回调onVsync

   - onVsync中通过Handler将FrameDisplayEventReceiver自身添加到Message中post了出去，
   - FrameDisplayEventReceiver实现了Runnable,因此执行Message时会执行它的run方法
   - 在run方法中调用了doFrame开启帧绘制流程。

5. doFrame中主会进行掉帧逻辑计算、记录帧绘制的信息、然后执行doCallback。

   - Choreographer.doFrame() 处理 Choreographer 的第一个 callback ： input
   - Choreographer.doFrame() 处理 Choreographer 的第二个 callback ： animation
   - Choreographer.doFrame() 处理 Choreographer 的第三个 callback ： insets animation
   - Choreographer.doFrame() 处理 Choreographer 的第四个 callback ： traversal

6. 开始执行View的绘制流程

## 二、Choreographer详解

在Android 4.1版本之前，由于屏幕的绘制没有一个良好的绘制周期和缓冲，丢帧和屏幕撕裂的问题非常严重。Android4.1版本之后引入了Vsync、TripleBuffer和Choreographer来优化显示性能。

Choreographer与Vsync配合，为上层APP的渲染提供一个稳定的Message处理时机。目前大部分手机都是60Hz的刷新率，即16.6ms刷新一次。系统为了配合屏幕的刷新频率，将Vsync的周期也设置为16.6ms。每隔16.6ms，Vsync信号就会唤醒Chroeographer来执行APP的绘制操作。

Choreographer 扮演 Android 渲染链路中承上启下的角色：

- **承上** 负责接收和处理APP的各种更新消息和回调，等到Vsync到来时统一处理。比如集中处理Input事件、Animation、Traversal(measure、layout、draw)，判断卡顿掉帧情况，记录Callback耗时等。

- **启下** 负责请求和接收Vsync信号。

 **Choreographer + SurfaceFlinger + Vsync + TripleBuffer** 这一套从上到下的机制，保证了 Android App 可以以一个稳定的帧率运行，减少帧率波动带来的不适感.

## Choreographer 的初始化

Choreographer的初始化是使用ThreadLocal，可以看到，在ThreadLocal的初始化值时候实例化了Choreographer

```java
// Thread local storage for the choreographer.
private static final ThreadLocal<Choreographer> sThreadInstance =
  new ThreadLocal<Choreographer>() {
  @Override
  protected Choreographer initialValue() {
    // 获取当前线程Looper
    Looper looper = Looper.myLooper();
    if (looper == null) {
      throw new IllegalStateException("The current thread must have a looper!");
    }
    // 实例化Choreographer
    return new Choreographer(looper);
  }
};
```

从上述代码可以看到，每个Looper都有自己的Choreographer，其他线程发送的回调只能运行在对应的Choreographer所属的Looper线程上。



可以通过Choreographer的getInstance方法从ThreadLocal中获取Choreographer

```java
public static Choreographer getInstance() {
  return sThreadInstance.get();
}
```

在ViewRootImpl的构造方法中就有此代码：

```java
public ViewRootImpl(Context context, Display display) {
  ...
    mChoreographer = Choreographer.getInstance();

}
```

Choreographer的构造方法如下：

```java
private Choreographer(Looper looper) {
  mLooper = looper;
  // 初始化FrameHandler
  mHandler = new FrameHandler(looper);
  // 根据是否使用了VSYNC来创建一个FrameDisplayEventReceiver
  mDisplayEventReceiver = USE_VSYNC ? new FrameDisplayEventReceiver(looper) : null;
  mLastFrameTimeNanos = Long.MIN_VALUE;

  mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());
  // CALLBACK_LAST + 1 = 4，创建一个容量为4的CallbackQueue数组，用来存放4种不同的Callback
  mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
  for (int i = 0; i <= CALLBACK_LAST; i++) {
    mCallbackQueues[i] = new CallbackQueue();
  }
}
```
上述代码中先创建了一个Handler，接着使用USE_VSYNC查看是否使用了Vsync同步机制。如果是，则创建一个FrameDisplayEventReceiver对象用来请求并接受Vsync事件。

接着，创建了一个大小为4的数组，用于保存不同类型的Callback,有如下四种：
```java
/**
 *输入Callback类型，Runs first.
 */
public static final int CALLBACK_INPUT = 0; 

/**
 * 动画Callback类型，Runs before traversals.
 */
public static final int CALLBACK_ANIMATION = 1;

/**
 * View绘制Callback类型，处理layout和draw.  Runsafter all other 
 * asynchronous messages have been handled.
 */
public static final int CALLBACK_TRAVERSAL = 2;

/**
 * 提交Callback类型.  Handles post-draw operations for the frame.
 * Runs after traversal completes.  
 */
public static final int CALLBACK_COMMIT = 3;
```



Vsync的注册、申请、接收都是通过FrameDisplayEventReceiver实现的。 FrameDisplayEventReceiver 继承 DisplayEventReceiver ， 有三个比较重要的方法，代码如下：



```java
private final class FrameDisplayEventReceiver extends DisplayEventReceiver
  implements Runnable {
  private boolean mHavePendingVsync;
  private long mTimestampNanos;
  private int mFrame;

  public FrameDisplayEventReceiver(Looper looper) {
    super(looper);
  }

  // Vsync信号回调
  @Override
  public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {

    if (builtInDisplayId != SurfaceControl.BUILT_IN_DISPLAY_ID_MAIN) {
      // 请求Vsync信号
      scheduleVsync();
      return;
    }

    long now = System.nanoTime();
    if (timestampNanos > now) {
      Log.w(TAG, "Frame time is " + ((timestampNanos - now) * 0.000001f)
            + " ms in the future!  Check that graphics HAL is generating vsync "
            + "timestamps using the correct timebase.");
      timestampNanos = now;
    }

    if (mHavePendingVsync) {
      Log.w(TAG, "Already have a pending vsync event.  There should only be "
            + "one at a time.");
    } else {
      mHavePendingVsync = true;
    }

    mTimestampNanos = timestampNanos;
    mFrame = frame;
    Message msg = Message.obtain(mHandler, this);
    msg.setAsynchronous(true);
    mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
  }
  // scheduleVsync方法位于DisplayEventReceiver中，用来请求Vsync信号,如下
  // public void scheduleVsync() {
  //    ...
  //    nativeScheduleVsync(mReceiverPtr);
  // }  

  // 执行doFrame
  @Override
  public void run() {
    mHavePendingVsync = false;
    doFrame(mTimestampNanos, mFrame);
  }
}
```

 FrameDisplayEventReceiver在初始化的时候通过监听文件句柄在Vsync信号到来时回调onVsync，如下：



```java
// android/view/DisplayEventReceiver.java
public DisplayEventReceiver(Looper looper, int vsyncSource) {
  ......
    mMessageQueue = looper.getQueue();
  mReceiverPtr = nativeInit(new WeakReference<DisplayEventReceiver>(this), mMessageQueue,vsyncSource);
}
```

简单来说，FrameDisplayEventReceiver 的初始化过程中，通过 BitTube(本质是一个 socket pair)，来传递和请求 Vsync 事件，当 SurfaceFlinger 收到 Vsync 事件之后，通过 appEventThread 将这个事件通过 BitTube 传给 DisplayEventDispatcher ，DisplayEventDispatcher 通过 BitTube 的接收端监听到 Vsync 事件之后，回调 Choreographer.FrameDisplayEventReceiver.onVsync ，触发开始一帧的绘制.



## Choreographer的doFrame



Choreographer 处理绘制的逻辑核心在 Choreographer.doFrame 方法中，FrameDisplayEventReceiver.onVsync post 了自己，其 run 方法直接调用了 doFrame 开始一帧的逻辑处理。doFrame 函数主要做下面几件事：

- 计算掉帧逻辑
- 记录帧绘制信息
- 执行 CALLBACK_INPUT、CALLBACK_ANIMATION、CALLBACK_INSETS_ANIMATION、CALLBACK_TRAVERSAL、CALLBACK_COMMIT

### 计算掉帧逻辑

```java
void doFrame(long frameTimeNanos, int frame) {
  final long startNanos;
  synchronized (mLock) {
    ... 
    long intendedFrameTimeNanos = frameTimeNanos;
    startNanos = System.nanoTime();
    final long jitterNanos = startNanos - frameTimeNanos;
    if (jitterNanos >= mFrameIntervalNanos) {
      final long skippedFrames = jitterNanos / mFrameIntervalNanos;
      if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
        Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
              + "The application may be doing too much work on its main thread.");
      }
      ...
    }

    ...
  }
```

Vsync 信号到来的时候会标记一个 start_time ，执行 doFrame 的时候标记一个 end_time ，这两个时间差就是 Vsync 处理时延，也就是掉帧。



### 记录帧绘制信息

Choreographer 中 FrameInfo 来负责记录帧的绘制信息，doFrame 执行的时候，会把每一个关键节点的绘制时间记录下来，我们使用 dumpsys gfxinfo 就可以看到。当然 Choreographer 只是记录了一部分，剩余的部分在 hwui 那边来记录。

doFrame 函数记录从 Vsync time 到 markPerformTraversalsStart 的时间



```java
void doFrame(long frameTimeNanos, int frame) {
  ...
  mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);
  // 处理 CALLBACK_INPUT Callbacks 
  mFrameInfo.markInputHandlingStart();
  // 处理 CALLBACK_ANIMATION Callbacks
  mFrameInfo.markAnimationsStart();
  // 处理 CALLBACK_INSETS_ANIMATION Callbacks
  // 处理 CALLBACK_TRAVERSAL Callbacks
  mFrameInfo.markPerformTraversalsStart();
  // 处理 CALLBACK_COMMIT Callbacks
  ...
}
```



## 执行 Callbacks

```java
void doFrame(long frameTimeNanos, int frame) {
  ...
  // 处理 CALLBACK_INPUT Callbacks 
  doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);
  // 处理 CALLBACK_ANIMATION Callbacks
  doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
  // 处理 CALLBACK_INSETS_ANIMATION Callbacks
  doCallbacks(Choreographer.CALLBACK_INSETS_ANIMATION, frameTimeNanos);
  // 处理 CALLBACK_TRAVERSAL Callbacks
  doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
  // 处理 CALLBACK_COMMIT Callbacks
  doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
  ...
}
```



**Input 回调调用栈**一般是执行 ViewRootImpl.ConsumeBatchedInputRunnable

```java
// android/view/ViewRootImpl.java
final class ConsumeBatchedInputRunnable implements Runnable {
  @Override
  public void run() {
    doConsumeBatchedInput(mChoreographer.getFrameTimeNanos());
  }
}
void doConsumeBatchedInput(long frameTimeNanos) {
  if (mConsumeBatchedInputScheduled) {
    mConsumeBatchedInputScheduled = false;
    if (mInputEventReceiver != null) {
      if (mInputEventReceiver.consumeBatchedInputEvents(frameTimeNanos)
          && frameTimeNanos != -1) {
        scheduleConsumeBatchedInput();
      }
    }
    doProcessInputEvents();
  }
}
```

Input 时间经过处理，最终会传给 DecorView 的 dispatchTouchEvent，这就到了我们熟悉的 Input 事件分发.

**Traversal 调用栈** 

```java
// android/view/ViewRootImpl.java
void scheduleTraversals() {
  if (!mTraversalScheduled) {
    mTraversalScheduled = true;

    mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
    mChoreographer.postCallback(
      Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
  }
}
```

执行ViewRootImpl的scheduleTraversals开启测量绘制流程。

> [Choreographer源码分析](https://androidperformance.com/2019/10/22/Android-Choreographer/#Choreographer-%E7%9A%84%E5%88%9D%E5%A7%8B%E5%8C%96)
{: .prompt-tip}