---
layout: post
title:  "Android属性动画原理分析(一)-ValueAnimator"
date:   2022-02-02
---
源码角度分析ValueAnimator内部实现原理   



# 介绍

属性动画系统是一个强健的框架，用于为几乎任何内容添加动画效果。您可以定义一个随时间更改任何对象属性的动画，无论其是否绘制到屏幕上。属性动画会在指定时长内更改属性（对象中的字段）的值。要添加动画效果，请指定要添加动画效果的对象属性，例如对象在屏幕上的位置、动画效果持续多长时间以及要在哪些值之间添加动画效果。

借助属性动画系统，您可以定义动画的以下特性：

- 时长：您可以指定动画的时长。默认时长为 300 毫秒。
- 时间插值：您可以指定如何根据动画的当前已播放时长来计算属性的值。
- 重复计数和行为：您可以指定是否在某个时长结束后重复播放动画以及重复播放动画多少次。您还可以指定是否要反向播放动画。如果将其设置为反向播放，则会先播放动画，然后反向播放动画，直到达到重复次数。
- Animator 集：您可以将动画分成多个逻辑集，它们可以一起播放、按顺序播放或者在指定的延迟时间后播放。
- 帧刷新延迟：您可以指定动画帧的刷新频率。默认设置为每 10 毫秒刷新一次，但应用刷新帧的速度最终取决于整个系统的繁忙程度以及系统为底层计时器提供服务的速度。

# 简单使用

```kotlin
ValueAnimator.ofFloat(1f, 0f).apply {
    duration = 200
    interpolator = LinearInterpolator()
    addUpdateListener { animation: ValueAnimator ->
        val value = animation.animatedValue as Float
        mHeadView?.scaleX = value
        mHeadView?.scaleY = value
        mHeadView?.alpha = value
    }
    addListener(
        object : AnimatorListenerAdapter() {
            override fun onAnimationEnd(animation: Animator?) {
                removeView()
            }
        }
    )
    start()
}
```

上述代码定义了一个时长为200ms，值从1线性递减到0的`ValueAnimator`,并且通过向`ValueAnimator`添加`AnimatorUpdateListener`来在动画执行过程中更改mHeadView的部分属性，添加`AnimatorListener`在动画结束时移除mHeadView。

对于上述代码来说，Android开发同学应该再熟悉不过了，下面就以上述代码为出发点一窥属性动画的内部实现原理。

# 源码分析

## ofFloat

从`ValueAnimator.ofFloat(1f, 0f)`看起

```java
public static ValueAnimator ofFloat(float... values) {
  ValueAnimator anim = new ValueAnimator();
  anim.setFloatValues(values);
  return anim;
}

public ValueAnimator() {
}

public void setFloatValues(float... values) {
  if (values == null || values.length == 0) {
    return;
  }
  if (mValues == null || mValues.length == 0) {
    setValues(PropertyValuesHolder.ofFloat("", values));
  } else {
    PropertyValuesHolder valuesHolder = mValues[0];
    valuesHolder.setFloatValues(values);
  }
  mInitialized = false;
}
```

可以看到`ofFloat(1f, 0f)`方法首先会去创建一个空构造的`ValueAnimator`对象，然后把外部传进来的`values`数组存在`PropertyValuesHolder`对象里，最后把`ValueAnimator`对象返回。那么，`PropertyValuesHolder`是什么呢？

```java
/**
 * This class holds information about a property and the values that that property
 * should take on during an animation. PropertyValuesHolder objects can be used to create
 * animations with ValueAnimator or ObjectAnimator that operate on several different properties
 * in parallel.
 */
public class PropertyValuesHolder implements Cloneable {

  private PropertyValuesHolder(String propertyName) {
    mPropertyName = propertyName;
  }
  
  public static PropertyValuesHolder ofFloat(String propertyName, float... values) {
    return new FloatPropertyValuesHolder(propertyName, values);
  }
  
  static class FloatPropertyValuesHolder extends PropertyValuesHolder {
    public FloatPropertyValuesHolder(String propertyName, float... values) {
      super(propertyName);
      setFloatValues(values);
    }

    public void setFloatValues(float... values) {
      mValueType = float.class;
      mKeyframes = KeyframeSet.ofFloat(values);
    }
	}
}
```

注释的意思大致是：`PropertyValuesHolder`是动画作用于的属性及值的容器，用于协助`ValueAnimator`，接着往下看`KeyframeSet`

```java
public static KeyframeSet ofFloat(float... values) {
  boolean badValue = false;
  int numKeyframes = values.length;
  FloatKeyframe keyframes[] = new FloatKeyframe[Math.max(numKeyframes,2)];
  if (numKeyframes == 1) {
    keyframes[0] = (FloatKeyframe) Keyframe.ofFloat(0f);
    keyframes[1] = (FloatKeyframe) Keyframe.ofFloat(1f, values[0]);
    if (Float.isNaN(values[0])) {
      badValue = true;
    }
  } else {
    keyframes[0] = (FloatKeyframe) Keyframe.ofFloat(0f, values[0]);
    for (int i = 1; i < numKeyframes; ++i) {
      keyframes[i] =
        (FloatKeyframe) Keyframe.ofFloat((float) i / (numKeyframes - 1), values[i]);
      if (Float.isNaN(values[i])) {
        badValue = true;
      }
    }
  }
  ...
  return new FloatKeyframeSet(keyframes);
}

```

可以看到，`KeyframeSet`会根据传入的values数组的长度构建一个`Keyframe`（关键帧）类型的keyframes数组，相当于将整个动画分成了多段， 每段的开始点和结束点都是一个`KeyFrame`。

具体地说，当`ValueAnimator.ofFloat()`传入一个参数时，动画仅有一段，起始点为`Keyframe.ofFloat(0f)`， 结束点为 `Keyframe.ofFloat(1f, values[0])`，传入两个参数时，动画也仅有一段，起始点为`Keyframe.ofFloat(0f, values[0])`，结束点为`Keyframe.ofFloat(1f, values[1])`，传入三个参数时，动画有两段，第一段起始点为`Keyframe.ofFloat(0f, values[0])`，第一段结束点也同为第二段起始点为`Keyframe.ofFloat(0.5f, values[1])`，第二段结束点为`Keyframe.ofFloat(1f, values[2])`，以此类推。

```java
/**
 * This class holds a time/value pair for an animation. The Keyframe class is used
 * by {@link ValueAnimator} to define the values that the animation target will have over the course
 * of the animation. As the time proceeds from one keyframe to the other, the value of the
 * target object will animate between the value at the previous keyframe and the value at the
 * next keyframe.
 */
public abstract class Keyframe implements Cloneable {
  public static Keyframe ofFloat(float fraction, float value) {
    return new FloatKeyframe(fraction, value);
  }
}

static class FloatKeyframe extends Keyframe {
  FloatKeyframe(float fraction, float value) {
    mFraction = fraction;
    mValue = value;
    mValueType = float.class;
    mHasValue = true;
  }
  
  FloatKeyframe(float fraction) {
    mFraction = fraction;
    mValueType = float.class;
  }
}
```

每个`Keyframe`都持有一个`fraction-value`键值对， 这个`fraction-value`键值对是后续用于计算在动画执行到某个位置时相应的值为多少而用的。

到此`ValueAnimator.ofFloat()`就结束了，简单总结一下：`ValueAnimator.ofFloat()`方法会根据参数的长度将动画分为多段，并将每一段的开始点和结束点以及值按顺序存放在`PropertyValuesHolder`里。

回过头继续看我们的demo,`ValueAnimator.setDuration()`, `setInterpolator()`, `addUpdateListener()`, `addListener()`都仅仅是在内部记了个变量，我们重点看下`ValueAnimator.start()`方法。

## start

```java
@Override
public void start() {
  start(false);
}

private void start(boolean playBackwards) {
  ...  
  addAnimationCallback(0);

  if (mStartDelay == 0 || mSeekFraction >= 0 || mReversing) {
    // If there's no start delay, init the animation and notify start listeners right away
    // to be consistent with the previous behavior. Otherwise, postpone this until the first
    // frame after the start delay.
    startAnimation();
    if (mSeekFraction == -1) {
      // No seek, start at play time 0. Note that the reason we are not using fraction 0
      // is because for animations with 0 duration, we want to be consistent with pre-N
      // behavior: skip to the final value immediately.
      setCurrentPlayTime(0);
    } else {
      setCurrentFraction(mSeekFraction);
    }
  }
}
```

`ValueAnimator.start()`调用了几个比较重要的方法：

1. `addAnimationCallback(0)`
2. `startAnimation()`
3. `setCurrentPlayTime(0)`/`setCurrentFraction(mSeekFraction)`

###   `addAnimationCallback()`

```java
private void addAnimationCallback(long delay) {
  ...
  getAnimationHandler().addAnimationFrameCallback(this, delay);
}

public AnimationHandler getAnimationHandler() {
  return mAnimationHandler != null ? mAnimationHandler : AnimationHandler.getInstance();
}

/**
 * This custom, static handler handles the timing pulse that is shared by all active
 * ValueAnimators. This approach ensures that the setting of animation values will happen on the
 * same thread that animations start on, and that all animations will share the same times for
 * calculating their values, which makes synchronizing animations possible.
 */
public class AnimationHandler {
  public final static ThreadLocal<AnimationHandler> sAnimatorHandler = new ThreadLocal<>();
  
  public static AnimationHandler getInstance() {
    if (sAnimatorHandler.get() == null) {
      sAnimatorHandler.set(new AnimationHandler());
    }
    return sAnimatorHandler.get();
  }
  
  public void addAnimationFrameCallback(final AnimationFrameCallback callback, long delay) {
    if (mAnimationCallbacks.size() == 0) {
      getProvider().postFrameCallback(mFrameCallback);
    }
    if (!mAnimationCallbacks.contains(callback)) {
      mAnimationCallbacks.add(callback);
    }

    if (delay > 0) {
      mDelayedCallbackStartTime.put(callback, (SystemClock.uptimeMillis() + delay));
    }
  }
}
```

可以看到，`ValueAnimator.addAnimationCallback()`会先拿到当前线程的`AnimationHandler`单例，然后再把实现了`AnimationFrameCallback`接口的自己注册到`AnimationHandler`里去。

那么`AnimationHandler`是干什么的呢？通过注释，可以了解到：`AnimationHandler`负责给所有活动的`ValueAnimator`定时回调，使用`AnimationHandler`的优点是: 1.所有动画在相同线程设置动画的值 2.所有动画共享相同时间计算值，这使得各个动画是同步的。

继续往下，`AnimationHandler.addAnimationFrameCallback()`做了三件事：

1. 如果`mAnimationCallbacks`长度为0，调用`getProvider().postFrameCallback(mFrameCallback)`
2. 将实现了`AnimationFrameCallback`接口的`callback`添加进`mAnimationCallbacks` 
3. 如果`delay` > 0,将`callback - (SystemClock.uptimeMillis() + delay)`键值对添加进`mDelayedCallbackStartTime`中

```java
private AnimationFrameCallbackProvider getProvider() {
  if (mProvider == null) {
    mProvider = new MyFrameCallbackProvider();
  }
  return mProvider;
}

// Default provider of timing pulse that uses Choreographer for frame callbacks.
private class MyFrameCallbackProvider implements AnimationFrameCallbackProvider {

  final Choreographer mChoreographer = Choreographer.getInstance();
  
  @Override
  public void postFrameCallback(Choreographer.FrameCallback callback) {
    mChoreographer.postFrameCallback(callback);
  }
}
```

`getProvider()`方法会得到一个`AnimationHandler`的内部类`P`，其实就是负责与`Choreographer`交互的中间人，那么`Choreographer`是什么呢？这里先简单介绍下，`Choreographer`是android系统中所有动画和绘制的管理者，以接近恒定的16.6ms(60HZ)为通知上层做绘制操作。介于篇幅有限，这里先略过`Choreographer`实现原理，先把主流程走完，我们将在[Android属性动画原理(二)-Choreographer](2022-02-03-Android属性动画原理(二)-Choreographer.markdown)中详细介绍`Choreographer`。

```java
// AnimationHandler
private final Choreographer.FrameCallback mFrameCallback = new Choreographer.FrameCallback() {
  @Override
  public void doFrame(long frameTimeNanos) {
    // getProvider().getFrameTime()依然是通过Choreographer得到
    doAnimationFrame(getProvider().getFrameTime());
    if (mAnimationCallbacks.size() > 0) {
      getProvider().postFrameCallback(this);
    }
  }
};
```

上述代码便是接收到`Choreographer`绘制通知的回调，可以看到，如果`mAnimationCallbacks`长度不为0，即还存在需要执行的动画，则会在回调中再次调用`getProvider().postFrameCallback(this)`,以此来与`Choreographer`交互并不断触发该回调来执行 `doAnimationFrame(getProvider().getFrameTime())`，如果`mAnimationCallbacks`为0，则暂时终止和`Choreographer`交互，等到有其他需要执行的动画，即再次调用`ValueAnimator.start()`时恢复和`Choreographer`的交互。

```java
private void doAnimationFrame(long frameTime) {
  long currentTime = SystemClock.uptimeMillis();
  final int size = mAnimationCallbacks.size();
  for (int i = 0; i < size; i++) {
    final AnimationFrameCallback callback = mAnimationCallbacks.get(i);
    if (callback == null) {
      continue;
    }
    if (isCallbackDue(callback, currentTime)) {
      callback.doAnimationFrame(frameTime);
      ...
    }
  }
  cleanUpList();
}

private boolean isCallbackDue(AnimationFrameCallback callback, long currentTime) {
  Long startTime = mDelayedCallbackStartTime.get(callback);
  if (startTime == null) {
    return true;
  }
  if (startTime < currentTime) {
    // delay时长已到，从mDelayedCallbackStartTime移除
    mDelayedCallbackStartTime.remove(callback);
    return true;
  }
  return false;
}
```

可以看到，在收到`Choreographer`通知后，会遍历`mAnimationCallbacks`，先将到达`delay`时长的动画会从`mDelayedCallbackStartTime`中移除，然后所有需要执行的动画都会触发`callback.doAnimationFrame(frameTime)`。

```java
// ValueAnimator
public final boolean doAnimationFrame(long frameTime) {
	...
  final long currentTime = Math.max(frameTime, mStartTime);
  boolean finished = animateBasedOnTime(currentTime);
  if (finished) {
    endAnimation();
  }
  return finished;
}

boolean animateBasedOnTime(long currentTime) {
	if (mRunning) {
    final long scaledDuration = getScaledDuration();
    // 值得一提的是，这里算出来的fraction = (当前时间 - 开始时间) / 动画时长，所以可能大于1，因为动画可以播好多遍，如果此时
    // 正处于第二遍动画的中点，则fraction = 1.5f
    final float fraction = scaledDuration > 0 ?
      (float)(currentTime - mStartTime) / scaledDuration : 1f;
		...
    mOverallFraction = clampFraction(fraction);
    // currentIterationFraction = 1.5 - 1 = 0.5f
    float currentIterationFraction = getCurrentIterationFraction(mOverallFraction, mReversing);
    animateValue(currentIterationFraction);
  }
  return done;
}

void animateValue(float fraction) {
  // 插值计算得到实际的fraction
  fraction = mInterpolator.getInterpolation(fraction);
  mCurrentFraction = fraction;
  int numValues = mValues.length;
  for (int i = 0; i < numValues; ++i) {
    // 	交由PropertyValuesHolder对象计算值
    mValues[i].calculateValue(fraction);
  }
  if (mUpdateListeners != null) {
    int numListeners = mUpdateListeners.size();
    for (int i = 0; i < numListeners; ++i) {
      // 通知外部
      mUpdateListeners.get(i).onAnimationUpdate(this);
    }
  }
}
```

可以看到，`callback.doAnimationFrame(frameTime)`主要就是根据当前时间，算出当前动画应该执行到的位置`fraction`(受插值器影响)，然后交由`PropertyValuesHolder`根据当前位置`fraction`，计算出动画的值，并通过`AnimatorUpdateListener.onAnimationUpdate()`通知外部。

最后，我们来看一下`PropertyValuesHolder`内部计算值的具体实现

```java
// PropertyValuesHolder
void calculateValue(float fraction) {
  Object value = mKeyframes.getValue(fraction);
  mAnimatedValue = mConverter == null ? value : mConverter.convert(value);
}

...
// FloatKeyframeSet
@Override
public float getFloatValue(float fraction) {
	...
  for (int i = 1; i < mNumKeyframes; ++i) {
    FloatKeyframe nextKeyframe = (FloatKeyframe) mKeyframes.get(i);
    // 遍历找到动画执行当前位置前后的两个KeyFrame
    if (fraction < nextKeyframe.getFraction()) {
      final TimeInterpolator interpolator = nextKeyframe.getInterpolator();
      float intervalFraction = (fraction - prevKeyframe.getFraction()) /
        (nextKeyframe.getFraction() - prevKeyframe.getFraction());
      float prevValue = prevKeyframe.getFloatValue();
      float nextValue = nextKeyframe.getFloatValue();
      if (interpolator != null) {
        intervalFraction = interpolator.getInterpolation(intervalFraction);
      }
      // 计算的值会受TypeEvaluator影响
      return mEvaluator == null ?
        prevValue + intervalFraction * (nextValue - prevValue) :
      ((Number)mEvaluator.evaluate(intervalFraction, prevValue, nextValue)).
        floatValue();
    }
    prevKeyframe = nextKeyframe;
  }
}
```

大家应该都还记得，在`ValueAnimator.ofFloat()`中已经将一个个`fraction-value`键值对存在`KeyFrame`对象中了，这里会根据`fraction`，拿到前后两个最相近的`KeyFrame`，通过公式 `prevValue + intervalFraction * (nextValue - prevValue)` 计算出来了，当然也可以手动给动画设置`TypeEvaluator`达到自定义的目的。

到此，我们简单总结一下，`ValueAnimator.addAnimationCallback()`:

1. 将自己即`ValueAnimator`对象注册进`AnimationHandler`
2. `AnimationHandler`发送通知给`Choreographer`
3. `AnimationHandler`等待`Choreographer`回调通知
4. `AnimationHandler`接收到`Choreographer`通知后，通知`ValueAnimator`对象
5. `ValueAnimator`根据`Choreographer`通知的当前时间计算出当前动画应该执行到的位置`fraction`
6. `ValueAnimator`内部的`PropertyValuesHolder`根据`fraction`计算出动画当前的值
7. `ValueAnimator`通过`AnimatorUpdateListener.onAnimationUpdate()`通知外部
7. 重复2

![ValueAnimator](../assets/img/ValueAnimator1.png)

`addAnimationCallback()`方法到此结束，下面我们再看一下`ValueAnimator.start()`方法中调用的另外两个方法。

### `startAnimation()`

```java
private void start(boolean playBackwards) {
	...
  mStartTime = -1;
  addAnimationCallback(0);
  if (mStartDelay == 0 || mSeekFraction >= 0 || mReversing) {
      // If there's no start delay, init the animation and notify start listeners right away
      // to be consistent with the previous behavior. Otherwise, postpone this until the first
      // frame after the start delay.
      startAnimation();
    	...
  }
}
```

可以看到在`ValueAnimator.start()`中，只要满足下面三个条件中任何一个，就会调用 `startAnimation()`：

1. `mStartDelay == 0` 即没有调用`setStartDelay()`设置延迟播放
2. `mSeekFraction >= 0`即调用了`setCurrentFraction()`设置动画起始点
3. `mReversing == true`即没有调用`reverse()`设置反转动画

由此我们可以得知，同时设置`mStartDelay`和`mSeekFraction` /  `mReversing`，`mStartDelay`是不会生效的，动画仍然会马上开始，从下面代码中也可以看出

```java
public final boolean doAnimationFrame(long frameTime) {
  if (mStartTime < 0) {
    // First frame. If there is start delay, start delay count down will happen *after* this
    // frame.
    // 代码1
    mStartTime = mReversing
      ? frameTime
      : frameTime + (long) (mStartDelay * resolveDurationScale());
  }
  ...
  if (!mRunning) {
    // If not running, that means the animation is in the start delay phase of a forward
    // running animation. In the case of reversing, we want to run start delay in the end.
    // 代码2
    if (mStartTime > frameTime && mSeekFraction == -1) {
      // This is when no seek fraction is set during start delay. If developers change the
      // seek fraction during the delay, animation will start from the seeked position
      // right away.
      return false;
    } else {
      // If mRunning is not set by now, that means non-zero start delay,
      // no seeking, not reversing. At this point, start delay has passed.
      mRunning = true;
      startAnimation();
    }
  }
	...
  // 根据动画执行位置计算动画值，上文已做过解读
  final long currentTime = Math.max(frameTime, mStartTime);
  boolean finished = animateBasedOnTime(currentTime);
  if (finished) {
    endAnimation();
  }
  return finished;
}
```

首先，在满足条件`mStartTime < 0`即调用`ValueAnimator.start()`后第一次收到`AnimationHandler`回调时会进入`代码1`，`代码1`中`mStartTime`是只有在`mReversing == false`的情况下，才会将`mStartDelay`加入`mStartTime`,如果`mReversing == true`则`mStartTime`为参数`frameTime`。

其次，在满足条件`mRunning == false`即动画还没有真正开始执行时会进`代码2`，`代码2`中只有同时满足条件`mStartTime > frameTime`和`mSeekFraction == -1`时才会直接返回，如果任一条件不满足，便会触发`startAnimation()`并开始运行动画计算值。



最后我们来看一下`startAnimation()`内部究竟做了什么。

```java
// ValueAnimator
private void startAnimation() {
  mAnimationEndRequested = false;
  initAnimation();
  mRunning = true;
  if (mSeekFraction >= 0) {
    mOverallFraction = mSeekFraction;
  } else {
    mOverallFraction = 0f;
  }
  if (mListeners != null) {
    notifyStartListeners();
  }
}

void initAnimation() {
  if (!mInitialized) {
    int numValues = mValues.length;
    for (int i = 0; i < numValues; ++i) {
      mValues[i].init();
    }
    mInitialized = true;
  }
}

// PropertyValuesHolder
void init() {
  if (mEvaluator == null) {
    mEvaluator = (mValueType == Integer.class) ? sIntEvaluator :
    (mValueType == Float.class) ? sFloatEvaluator :
    null;
  }
  if (mEvaluator != null) {
    mKeyframes.setEvaluator(mEvaluator);
  }
}
```

`startAnimation()`比较简单，仅做了一些初始化操作，并调用AnimatorListener.onAnimationStart()通知外部。

### `setCurrentPlayTime()`/`setCurrentFraction()`

```java
// ValueAnimator
private void start(boolean playBackwards) {
  ...  
  mStartTime = -1;
  addAnimationCallback(0);

  if (mStartDelay == 0 || mSeekFraction >= 0 || mReversing) {
    // If there's no start delay, init the animation and notify start listeners right away
    // to be consistent with the previous behavior. Otherwise, postpone this until the first
    // frame after the start delay.
    startAnimation();
    if (mSeekFraction == -1) {
      // No seek, start at play time 0. Note that the reason we are not using fraction 0
      // is because for animations with 0 duration, we want to be consistent with pre-N
      // behavior: skip to the final value immediately.
      setCurrentPlayTime(0);
    } else {
      setCurrentFraction(mSeekFraction);
    }
  }
}

public void setCurrentPlayTime(long playTime) {
  float fraction = mDuration > 0 ? (float) playTime / mDuration : 1;
  setCurrentFraction(fraction);
}

public void setCurrentFraction(float fraction) {
  ...
  // 
  final float currentIterationFraction = getCurrentIterationFraction(fraction, mReversing);
  animateValue(currentIterationFraction);
}

```

可以看到无论是`setCurrentPlayTime()`还是`setCurrentFraction()`和之前`animateBasedOnTime()`的原理是一样的，都是先根据时间算出`fraction`，然后`PropertyValuesHolder`根据`fraction`算出动画值，并通过AnimatorUpdateListener.onAnimationUpdate()通知外部。

# 总结
