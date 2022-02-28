---
layout: post
title:  "Choreographer"
date:   2022-02-12
toc:  true
tags: [Android源码解读]
---
`Android`绘制链路上承上启下的类`Choreographer`

## 介绍

`Choreographer`负责接收显示系统的时间脉冲(垂直同步VSync信号)，在接收并对其修正调整后，控制同步处理动画(Animations)、输入(Input)和绘制(Draw)这三种UI操作。开发过程中很少与`Choreographer`直接交互，但是会使用动画或`View`的更高层接口来间接与`Choreographer`交互，这些接口包括但不限于:

1. `ValueAnimation.start`
2. `View.postOnAnimation` / `View.postOnAnimationDelayed`
3. `View.invalidate` / `View.postInvalidateOnAnimation`

`Choreographer`中文翻译过来为舞蹈指挥，可以理解为`Choreographer`优雅地指挥所有UI操作同步跳舞。

## 源码分析

下面我们就来分析`Choreographer`的实现原理。我们以`Choreographer`的创建为切入点开始分析。

### `Choreographer`的创建

```java
// Choreographer
public static Choreographer getInstance() {
  return sThreadInstance.get();
}

private static final ThreadLocal<Choreographer> sThreadInstance =
  new ThreadLocal<Choreographer>() {
    @Override
    protected Choreographer initialValue() {
      Looper looper = Looper.myLooper();
      if (looper == null) {
        throw new IllegalStateException("The current thread must have a looper!");
      }
      Choreographer choreographer = new Choreographer(looper, VSYNC_SOURCE_APP);
      if (looper == Looper.getMainLooper()) {
        mMainInstance = choreographer;
      }
      return choreographer;
    }
};

private Choreographer(Looper looper, int vsyncSource) {
  mLooper = looper;
  mHandler = new FrameHandler(looper);
  mDisplayEventReceiver = USE_VSYNC ? new FrameDisplayEventReceiver(looper, vsyncSource) : null;
  // 上一次绘制时间点
  mLastFrameTimeNanos = Long.MIN_VALUE;
  // 帧间时长，一般为16.6ms
  mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());

  mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
  for (int i = 0; i <= CALLBACK_LAST; i++) {
    mCallbackQueues[i] = new CallbackQueue();
  }
  setFPSDivisor(SystemProperties.getInt(ThreadedRenderer.DEBUG_FPS_DIVISOR, 1));
}
```

可以看到，`Choreographer`是线程单例的，外部通过`getInstance`方法拿到当前线程的`Choreographer`，在当前线程首次获取`Choreographer`的时候会触发`Choreographer`的创建。

`Choreographer`只有一个私有构造方法，第一个参数为当前线程的`Looper`（一般都为主线程），第二个参数为`int`类型的值`vsyncSource`。在构造方法中，依次定义了：

- `FrameHandler`
- `FrameDisplayEventReceiver`
- `mLastFrameTimeNanos`： 上一次绘制时间点
- `mFrameIntervalNanos`： 帧间时长，一般为16.6ms
- `CallbackQueue`类型的数组

### `CallbackQueue`

`CallbackQueue`为一个根据时间排序的单向链表结构，支持根据时间自由地增加或删除多个节点以及指定删除任一节点，介于篇幅有限，数据结构也比较简单，就不贴代码了，感兴趣的同学可以自行了解。

链表节点的数据结构`CallbackRecord`如下：

```java
// Choreographer
private static final class CallbackRecord {
  public CallbackRecord next;
  // 执行时间，CallbackQueue根据dueTime排序
  public long dueTime;
  // FrameCallback类型或Runnable类型的回调
  public Object action;
  public Object token;

  public void run(long frameTimeNanos) {
    if (token == FRAME_CALLBACK_TOKEN) {
      ((FrameCallback)action).doFrame(frameTimeNanos);
    } else {
      ((Runnable)action).run();
    }
  }
}

public interface FrameCallback {
	public void doFrame(long frameTimeNanos);
}
```

回到`Choreographer`构造方法中

```java
// Choreographer
public static final int CALLBACK_INPUT = 0;
public static final int CALLBACK_ANIMATION = 1;
public static final int CALLBACK_INSETS_ANIMATION = 2;
public static final int CALLBACK_TRAVERSAL = 3;
public static final int CALLBACK_COMMIT = 4;
private static final int CALLBACK_LAST = CALLBACK_COMMIT;

private Choreographer(Looper looper, int vsyncSource) {  
  ...
	mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
  for (int i = 0; i <= CALLBACK_LAST; i++) {
    mCallbackQueues[i] = new CallbackQueue();
  }
}
```

构造方法中定义了长度为`CALLBACK_LAST + 1`的`CallbackQueue`数组，每个元素分别对应一种操作的`CallbackQueue`，当`Choreographer`按如下顺序执行到某个操作时，会取出对应`CallbackQueue`中所有`dueTime`比当前时间小的`CallbackRecord`节点，调用其`run`方法，参数为某一个Vsync信号到来的时间点（下文会详细分析）。

1. `CALLBACK_INPUT`：处理输入事件
2. `CALLBACK_ANIMATION`：处理动画
3. `CALLBACK_INSETS_ANIMATION`：处理插入动画，与`CALLBACK_ANIMATION`区分是为了收集所有插入动画的更新
4. `CALLBACK_TRAVERSAL`：处理`View`的绘制
5. `CALLBACK_COMMIT`:  该操作调用`run`方法的参数`frameTimeNanos`值可能会比1-4要大N个帧间时长（1-4调用`run`方法的参数值相同），以反映1-4过重的绘制操作导致某些帧被跳过所发生的延迟。此操作的`frameTimeNanos`能够更精确地估计动画以及其他`View`的更新实际在哪一帧开始生效。该操作也是监测绘制操作耗时的时机。

### `FrameDisplayEventReceiver`

```java
// Choreographer
private final class FrameDisplayEventReceiver extends DisplayEventReceiver implements Runnable {

  public FrameDisplayEventReceiver(Looper looper, int vsyncSource) {
    super(looper, vsyncSource, CONFIG_CHANGED_EVENT_SUPPRESS);
  }
  
  ...
}

public abstract class DisplayEventReceiver {
  public DisplayEventReceiver(Looper looper, int vsyncSource, int configChanged) {
    ...
    mMessageQueue = looper.getQueue();
    mReceiverPtr = nativeInit(new WeakReference<DisplayEventReceiver>(this), mMessageQueue, vsyncSource, configChanged);
  }
  
  private static native long nativeInit(WeakReference<DisplayEventReceiver> receiver,
                                        MessageQueue messageQueue, int vsyncSource, int configChanged);
  
  ...
}
```

`FrameDisplayEventReceiver`的父类`DisplayEventReceiver`在创建时，调用了native方法`nativeInit`。

```c
// android_view_DisplayEventReceiver
static jlong nativeInit(JNIEnv* env, jclass clazz, jobject receiverWeak, jobject messageQueueObj) {
    sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);
    ...
    sp<NativeDisplayEventReceiver> receiver = new NativeDisplayEventReceiver(env,
            receiverWeak, messageQueue);
    status_t status = receiver->initialize();
    ...
    receiver->incStrong(gDisplayEventReceiverClassInfo.clazz);
    return reinterpret_cast<jlong>(receiver.get());
}
```

后续链路很长，我们不再继续跟下去。简单来说，在`DisplayEventReceiver`初始化过程中，建立了一条通过`BitTube`(本质是一个socket pair)来传递和请求Vsync事件的链路。当`SurfaceFlinger`收到Vsync事件之后，通过`appEventThread`将这个事件通过`BitTube`传给`DisplayEventDispatcher`，并调用`DisplayEventReceiver.dispatchVsync`方法。感兴趣的同学可以看一下[袁大佬的文章](http://gityuan.com/2017/02/11/surface_flinger/)。

```c++
void NativeDisplayEventReceiver::dispatchVsync(nsecs_t timestamp, int32_t id, uint32_t count) {
  JNIEnv* env = AndroidRuntime::getJNIEnv();
  ScopedLocalRef<jobject> receiverObj(env, jniGetReferent(env, mReceiverWeakGlobal));
  if (receiverObj.get()) {
    env->CallVoidMethod(receiverObj.get(),
                        gDisplayEventReceiverClassInfo.dispatchVsync, timestamp, id, count);
  }
  ...
}
```

```java
// DisplayEventReceiver
private void dispatchVsync(long timestampNanos, long physicalDisplayId, int frame) {
  onVsync(timestampNanos, physicalDisplayId, frame);
}

public void onVsync(long timestampNanos, long physicalDisplayId, int frame) {
}
```

Vsync信号最终会传到`DisplayEventReceiver.onVsync`方法中，子类`FrameDisplayEventReceiver`通过重载`onVsync`方法来监听Vsync信号。

```java
// Choreographer
private final class FrameDisplayEventReceiver extends DisplayEventReceiver implements Runnable {
  @Override
  public void onVsync(long timestampNanos, long physicalDisplayId, int frame) {
    long now = System.nanoTime();
    if (timestampNanos > now) {
      timestampNanos = now;
    }
    ...
    // 记录参数timestampNanos
    mTimestampNanos = timestampNanos;
    mFrame = frame;
    Message msg = Message.obtain(mHandler, this);
    msg.setAsynchronous(true);
    mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
  }
  
  @Override
  public void run() {
    mHavePendingVsync = false;
    // 将timestampNanos传入doFrame方法
    doFrame(mTimestampNanos, mFrame);
  }
}
```

`FrameDisplayEventReceiver`在收到Vsync信号后会发出一条以自己为`callback`的`Message`，当`mHandler`所在线程即主线程开始处理这条消息时，会调用`doFrame`方法，参数为Vsync信号到来的时间点`timestampNanos`。

```java
// Choreographer
void doFrame(long frameTimeNanos, int frame) {
  final long startNanos;
  synchronized (mLock) {
    // 如果mFrameScheduled为false，表示没有绘制工作要做，直接return
    if (!mFrameScheduled) {
      return; 
    }
		...
    // Vsync信号到来的时间frameTimeNanos
    long intendedFrameTimeNanos = frameTimeNanos;
    // 开始执行绘制的时间startNanos
    startNanos = System.nanoTime();
    // 计算startNanos与frameTimeNanos的时间差
    final long jitterNanos = startNanos - frameTimeNanos;
    // 如果时间差超过了帧间时长，表明中间跳过了N帧，即有N帧没有执行绘制操作(跳帧)
    if (jitterNanos >= mFrameIntervalNanos) {
      ...
      final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
      // 修正frameTimeNanos为离开始时间最近的上一次Vsync信号时间
      frameTimeNanos = startNanos - lastFrameOffset;
    }
    ...
    mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);
    // 标记mFrameScheduled为false，表示绘制工作已经完成
    mFrameScheduled = false;
    // 上一次Vsync信号到来的时间为frameTimeNanos
    mLastFrameTimeNanos = frameTimeNanos;
  }

  try {
    ...
    // 开始执行各种类型的绘制操作
    mFrameInfo.markInputHandlingStart();
    doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);
    mFrameInfo.markAnimationsStart();
    doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
    doCallbacks(Choreographer.CALLBACK_INSETS_ANIMATION, frameTimeNanos);
    mFrameInfo.markPerformTraversalsStart();
    doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
    doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
  } finally {
    ...
  }
}
```

`doFrame`做了如下几步操作：

1. 如果`mFrameScheduled`为`false`，表示没有绘制工作要做，直接`return`
2. 计算开始绘制时间`startNanos`与Vsync信号到来时间`frameTimeNanos`的时间差
3. 如果时间差大于帧间时长，则修正`frameTimeNanos`为离当前时间最近的上一次Vsync信号到来时间
4. `mFrameScheduled = false` & `mLastFrameTimeNanos = frameTimeNanos`
5. 调用`doCallbacks`方法，按顺序执行各种类型的绘制操作

```java
// Choreographer
void doCallbacks(int callbackType, long frameTimeNanos) {
  CallbackRecord callbacks;
  synchronized (mLock) {
    final long now = System.nanoTime();
    // 根据callbackType找到对应CallbackQueue并取出其中所有dueTime比当前时间小的节点
    callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(now / TimeUtils.NANOS_PER_MS);
    ...
    if (callbackType == Choreographer.CALLBACK_COMMIT) {
      // 当前仅当callbackType为CALLBACK_COMMIT进入该逻辑
      // 计算当前时间和Vsync信号到来时间的时间差
      final long jitterNanos = now - frameTimeNanos;
      if (jitterNanos >= 2 * mFrameIntervalNanos) {
        // 如果时间差超过2个帧间时长时，会对frameTimeNanos和mLastFrameTimeNanos进行修正，以反映前四个操作耗时过久导致某些帧被跳过所发生的延迟
        final long lastFrameOffset = jitterNanos % mFrameIntervalNanos + mFrameIntervalNanos;
        frameTimeNanos = now - lastFrameOffset;
        mLastFrameTimeNanos = frameTimeNanos;
      }
    }
  }
  try {
    // 调用CallbackRecord.run方法
    for (CallbackRecord c = callbacks; c != null; c = c.next) {
      c.run(frameTimeNanos);
    }
  } finally {
    ...
  }
}
```

如上文所述，`Choreographer`会按顺序执行所有UI操作，取出对应`CallbackQueue`中所有`dueTime`比当前时间小的`CallbackRecord`节点，调用其`run`方法。

### `FrameHandler`

```java
// Choreographer
private final class FrameHandler extends Handler {
  public FrameHandler(Looper looper) {
    super(looper);
  }

  @Override
  public void handleMessage(Message msg) {
    switch (msg.what) {
      case MSG_DO_FRAME:
        doFrame(System.nanoTime(), 0);
        break;
      case MSG_DO_SCHEDULE_VSYNC:
        doScheduleVsync();
        break;
      case MSG_DO_SCHEDULE_CALLBACK:
        doScheduleCallback(msg.arg1);
        break;
    }
  }
}
```

`FrameHandler`负责处理`MSG_DO_FRAME`、`MSG_DO_SCHEDULE_VSYNC`、`MSG_DO_SCHEDULE_CALLBACK`这三条消息，以及当Vsync信号到来时`callback`为`FrameDisplayEventReceiver`的回调。

`doFrame`之前已经介绍过了，`doScheduleVsync`和`doScheduleCallback`方法如下

```java
// Choreographer
void doScheduleVsync() {
  synchronized (mLock) {
    if (mFrameScheduled) {
      scheduleVsyncLocked();
    }
  }
}

private void scheduleVsyncLocked() {
  mDisplayEventReceiver.scheduleVsync();
}

// DisplayEventReceiver
public void scheduleVsync() {
  if (mReceiverPtr == 0) {
    ...
  } else {
    nativeScheduleVsync(mReceiverPtr);
  }
}

private static native void nativeScheduleVsync(long receiverPtr);

```

`doScheduleVsync`用于在`mFrameScheduled`为`true`即有工作需要处理的情况下请求Vsync信号。调用了该方法，`FrameDisplayEventReceiver`就能收到下一个Vsync信号通知。

```java
// Choreographer
void doScheduleCallback(int callbackType) {
  synchronized (mLock) {
    // 当前还没有任何工作需要处理
    if (!mFrameScheduled) {
      final long now = SystemClock.uptimeMillis();
      // 是否有满足时间条件的节点
      if (mCallbackQueues[callbackType].hasDueCallbacksLocked(now)) {
        scheduleFrameLocked(now);
      }
    }
  }
}

private static final boolean USE_VSYNC = SystemProperties.getBoolean("debug.choreographer.vsync", true);

private void scheduleFrameLocked(long now) {
  // 当前还没有任何工作需要处理
  if (!mFrameScheduled) {
    // 标记当前有工作需要处理
    mFrameScheduled = true;
    // 是否使用Vsync信号，一般都为true
    if (USE_VSYNC) {
      if (isRunningOnLooperThreadLocked()) {
        // 当前线程是主线程则直接请求Vsync信号
        scheduleVsyncLocked();
      } else {
        // 否则给FrameHandler发送MSG_DO_SCHEDULE_VSYNC消息，通过Handler在主线程上请求Vsync信号
        Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
        msg.setAsynchronous(true);
        mHandler.sendMessageAtFrontOfQueue(msg);
      }
    } else {
      // 	否则直接给FrameHandler发送MSG_DO_FRAME消息，进行绘制操作，无须等待Vsync信号的到来
      final long nextFrameTime = Math.max(
        mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
      Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
      msg.setAsynchronous(true);
      mHandler.sendMessageAtTime(msg, nextFrameTime);
    }
  }
}
```

`doScheduleCallback`做了如下几步操作：

1.如果`mFrameScheduled`为`false`，表示当前已经有其他需要处理的工作请求过Vsync信号，因此只需要等待下一个Vsync信号一起处理即可。如果当前还没有其他工作请求过Vsync信号即当前还没有任何工作需要处理，则继续往下

2.是否有满足时间条件的节点。如果没有直接返回

3.`mFrameScheduled = true` & 请求Vsync信号

到此，`Choreographer`源码已经介绍的差不多了，我们以在[属性动画ValueAnimator](../2022-02-02-属性动画ValueAnimator)中提到的🌰整体过一遍`Choreographer`的流程。

### 整体流程

```java
// AnimationHandler
private final Choreographer.FrameCallback mFrameCallback = new Choreographer.FrameCallback() {
  @Override
  public void doFrame(long frameTimeNanos) {
    doAnimationFrame(getProvider().getFrameTime());
    if (mAnimationCallbacks.size() > 0) {
      getProvider().postFrameCallback(this);
    }
  }
};

private class MyFrameCallbackProvider implements AnimationFrameCallbackProvider {

  final Choreographer mChoreographer = Choreographer.getInstance();
  
  @Override
  public void postFrameCallback(Choreographer.FrameCallback callback) {
    mChoreographer.postFrameCallback(callback);
  }
}
```

1.  `Choreographer.getInstance`，如果` Choreographer`在当前线程中没有被创建，则触发` Choreographer`的创建

1.  通过`FrameDisplayEventReceiver`建立一条传递和请求Vsync事件的链路

2. `Choreographer.postFrameCallback`，业务方请求绘制

   ```java
   // Choreographer
   public void postFrameCallback(FrameCallback callback) {
     postFrameCallbackDelayed(callback, 0);
   }
   
   public void postFrameCallback(FrameCallback callback) {
     postFrameCallbackDelayed(callback, 0);
   }
   
   public void postFrameCallbackDelayed(FrameCallback callback, long delayMillis) {
     if (callback == null) {
       throw new IllegalArgumentException("callback must not be null");
     }
     postCallbackDelayedInternal(CALLBACK_ANIMATION, callback, FRAME_CALLBACK_TOKEN, delayMillis);
   }
   
   private void postCallbackDelayedInternal(int callbackType, Object action, Object token, long delayMillis) {
   	...
     synchronized (mLock) {
       final long now = SystemClock.uptimeMillis();
       final long dueTime = now + delayMillis;
       // 根据dueTime向对应类型的CallbackQueue中添加节点
       mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);
   		// 一般情况不会走到，除非delayMillis为负？
       if (dueTime <= now) {
         scheduleFrameLocked(now);
       } else {
         Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
         msg.arg1 = callbackType;
         msg.setAsynchronous(true);
         mHandler.sendMessageAtTime(msg, dueTime);
       }
     }
   }
   ```

3. 按照`dueTime`顺序向对应类型的`CallbackQueue`中添加节点

4. 向`FrameHandler`发送`MSG_DO_SCHEDULE_CALLBACK`

5. 如果`mFrameScheduled`为`false`，表示当前已经有其他需要处理的工作请求过Vsync信号，因此只需要等待下一个Vsync信号一起处理即可。如果还没有其他工作请求过Vsync信号，则继续往下

6. 是否有满足时间条件的节点。如果没有直接返回

7. `mFrameScheduled = true` & 通过`FrameDisplayEventReceiver`请求Vsync信号

8. 等待Vsync信号到来

9. 在收到Vsync信号后，`FrameDisplayEventReceiver`向`FrameHandler`发送一条以自己为`callback`的`Message`

10. 当主线程处理到该消息时，调用`doFrame`方法，参数为Vsync信号到来时间`frameTimeNanos`

11. 如果`mFrameScheduled`为`false`，表示没有绘制工作要做，直接返回

12. 计算开始绘制时间`startNanos`与Vsync信号到来时间`frameTimeNanos`的时间差

13. 如果时间差大于帧间时长，则修正`frameTimeNanos`为离当前时间最近的上一次Vsync信号到来时间

14. `mFrameScheduled = false` & `mLastFrameTimeNanos = frameTimeNanos`

15. 调用`doCallbacks`方法，按顺序执行各种类型的绘制操作

16. 取出对应操作类型的`CallbackQueue`中所有`dueTime`比当前时间小的`CallbackRecord`节点，依次调用其`run`方法，参数为修正过的Vsync信号到来时间`frameTimeNanos`

17. `AnimationHandler`收到回调，并开始执行`FrameCallback.doFrame`绘制动画

18. 操作类型为`CALLBACK_COMMIT`时，计算当前时间和Vsync信号到来时间的时间差，如果时间差超过2个帧间时长，则再次修正`frameTimeNanos`后再调用`run`方法通知业务方

## 总结

[大图](../assets/img/Choreographer.png)

![ValueAnimator2](../assets/img/Choreographer.png)

`Choreographer`作为`Android`绘制链路上非常重要的一个类，起着承上启下的作用。承上：负责接收Vsync信号，启下：控制所有UI操作保持同步。本文介绍了其内部运转流程，让大家对`Android`的绘制链路有了更进一步的了解。

参考

1. [Choreographer 解析](https://juejin.cn/post/6844903886940012551)
1. [Choreographer原理](http://gityuan.com/2017/02/25/choreographer/)

注

1. 以上代码均基于API 30
2. 本文运用了大量`Handler`相关知识，对此不熟悉的同学可以阅读[Handler](../2022-02-27-Handler)
