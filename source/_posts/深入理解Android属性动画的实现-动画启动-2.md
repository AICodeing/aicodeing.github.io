---
title: 深入理解Android属性动画的实现(动画启动)-2
date: 2017-12-21 11:46:13
tags:
	- Android
	- 动画
category: Android
comments: true
---
### 属性动画的启动分析
在本文中,我们会分析属性动画如何启动的而且和Andoid `黄油计划`有什么关系

我们看看 `ObjectAnimator.start()`

```java
public void start() {
        AnimationHandler.getInstance().autoCancelBasedOn(this);
        if (DBG) {
            Log.d(LOG_TAG, "Anim target, duration: " + getTarget() + ", " + getDuration());
            for (int i = 0; i < mValues.length; ++i) {
                PropertyValuesHolder pvh = mValues[i];
                Log.d(LOG_TAG, "   Values[" + i + "]: " +
                    pvh.getPropertyName() + ", " + pvh.mKeyframes.getValue(0) + ", " +
                    pvh.mKeyframes.getValue(1));
            }
        }
        super.start();
    }
```
`AnimationHandler.getInstance().autoCancelBasedOn(this)` cancel 相同Target和相同属性的动画  
AnimationHandler 实例在线程局部单例。`autoCancelBasedOn(this)`会遍历 `AnimationHandler `实例持有的所有未完成的 `ValueAnimator`实例，cancel 掉符合条件的动画。

紧接着 `super.start()` 调用了`ValueAnimator.start()`  

```java
@Override
 public void start() {
     start(false);
 }
```
调用了带参数的 `start`方法

```java
private void start(boolean playBackwards) {
        if (Looper.myLooper() == null) {
            throw new AndroidRuntimeException("Animators may only be run on Looper threads");
        }
        mReversing = playBackwards;
        
			....
			
        mStarted = true;
        mPaused = false;
        mRunning = false;
        mAnimationEndRequested = false;
        // Resets mLastFrameTime when start() is called, so that if the animation was running,
        // calling start() would put the animation in the
        // started-but-not-yet-reached-the-first-frame phase.
        mLastFrameTime = 0;
        AnimationHandler animationHandler = AnimationHandler.getInstance();
        animationHandler.addAnimationFrameCallback(this, (long) (mStartDelay * sDurationScale));

 		....
    }
```

我们略去与主流程无关的逻辑代码。这个方法标记了动画的状态，如

```java
 mStarted = true;
 mPaused = false;
 mRunning = false;
```

然后这个方法结束啦，有没有很疑惑？有没有很懵逼？动画怎么执行的？什么时候执行的？  
要解答这问题，我们还是要分析 这两行不起眼的代码

```java
AnimationHandler animationHandler = AnimationHandler.getInstance();
animationHandler.addAnimationFrameCallback(this, (long) (mStartDelay * sDurationScale));
```
第一行在当前线程获取 `AnimationHandler`实例；  
第二行注册了一个callback。

```java
/**
     * Register to get a callback on the next frame after the delay.
     */
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
```  
***注释说明:callback 会在delay 时间后的下一个 frame 获得回调。***

```java
if (mAnimationCallbacks.size() == 0) {
   getProvider().postFrameCallback(mFrameCallback);
}
```
这两行代码 做了一件事情:保证一个`ValueAnimator`对象只向Provider注册一次 `mFrameCallback`

瞅瞅 `getProvider`是啥？

```java
private AnimationFrameCallbackProvider getProvider() {
    if (mProvider == null) {
        mProvider = new MyFrameCallbackProvider();
    }
    return mProvider;
    }
```
创建了一个 `MyFrameCallbackProvider`实例， `MyFrameCallbackProvider` 继承  `AnimationFrameCallbackProvider`

```java
public interface AnimationFrameCallbackProvider {
        void postFrameCallback(Choreographer.FrameCallback callback);
        void postCommitCallback(Runnable runnable);
        long getFrameTime();
        long getFrameDelay();
        void setFrameDelay(long delay);
    }
```
`AnimationFrameCallbackProvider`接口定义了一些回调接口，按照注释说明主要作用是 提高 `ValueAnimator`的可测性，通过这个接口隔离，我们可以自定义 定时脉冲，而不用使用系统默认的 `Choreographer`,这样我们可以在测试中使用任意的时间间隔来测试我们的定时脉冲.既然可以方便测试，那肯定有API来更改Provider 吧？

果不其然，我们猜对了。

```java
public void setProvider(AnimationFrameCallbackProvider provider) {
        if (provider == null) {
            mProvider = new MyFrameCallbackProvider();
        } else {
            mProvider = provider;
        }
    }
```

`ValueAnimator`提供了一个 `setProvider` 通过自定义的Provider 提供我们想要的任意时间间隔的回调，来更新动画。

明白了 接口`AnimationFrameCallbackProvider`的作用，也知道了一个新的名词`Choreographer`,它就是 ***Android 黄油计划*** 的核心。使用vsync（垂直同步）来协调View的绘制和动画的执行间隔。  
我们知道了默认情况下系统使用`Choreographer`,我们可以简单的认为 它是一个与 绘制和动画有关的 _**消息处理器**_ 。  
继续我们的 代码 `AnimationHandler.addAnimationFrameCallback `

```java
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
```

执行了 `getProvider().postFrameCallback(mFrameCallback)` 通过上面的分析我们知道 `getProvider()` 得到的是一个`MyFrameCallbackProvider`

```java
	/**
     * Default provider of timing pulse that uses Choreographer for frame callbacks.
     */
    private class MyFrameCallbackProvider implements AnimationFrameCallbackProvider {

        final Choreographer mChoreographer = Choreographer.getInstance();

        @Override
        public void postFrameCallback(Choreographer.FrameCallback callback) {
            mChoreographer.postFrameCallback(callback);
        }

        @Override
        public void postCommitCallback(Runnable runnable) {
            mChoreographer.postCallback(Choreographer.CALLBACK_COMMIT, runnable, null);
        }

        @Override
        public long getFrameTime() {
            return mChoreographer.getFrameTime();
        }

        @Override
        public long getFrameDelay() {
            return Choreographer.getFrameDelay();
        }

        @Override
        public void setFrameDelay(long delay) {
            Choreographer.setFrameDelay(delay);
        }
    }
```

注释说明系统默认使用的 `Choreographer`做定时脉冲来协调 frame 的更新 。 

`MyFrameCallbackProvider`的`postFrameCallback()`方法调用了` mChoreographer.postFrameCallback(callback)` 这里说明一下 Choreographer 实例也是线程局部单例的。从这些信息中我们知道了动画可以在子线程中执行的。但是这个子线程必须有Looper。 
接着分析 `Choreographer`的 `postFrameCallback`方法

```java
public void postFrameCallback(FrameCallback callback) {
        postFrameCallbackDelayed(callback, 0);
}
    
public void postFrameCallbackDelayed(FrameCallback callback, long delayMillis) {
    if (callback == null) {
            throw new IllegalArgumentException("callback must not be null");
    }

    postCallbackDelayedInternal(CALLBACK_ANIMATION,
                callback, FRAME_CALLBACK_TOKEN, delayMillis);
} 

private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {
    ....
        synchronized (mLock) {
            final long now = SystemClock.uptimeMillis();
            final long dueTime = now + delayMillis;
            mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

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
动画使用的 `callbackType`为 `CALLBACK_ANIMATION`，而默认支持 三种类型：

 `CALLBACK_INPUT`:处理输出相关的回调，最先被处理  
 `CALLBACK_ANIMATION`:处理动画相关的回调，在 `CALLBACK_TRAVERSAL`类型之前处理  
 `CALLBACK_TRAVERSAL`: 处理View 的layout 和 draw。该类型在所有其他异步消息处理完后开始处理，也基于这一点保证了界面不卡顿。
 
 每一个类型的 callbackType 都拥有一个`CallbackQueue` 我们的动画callback 会通过 `mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);` 保存在 `CallbackQueue`中成为一个链表中的一个节点。  
 处理完 callback 之后 下面会执行`scheduleFrameLocked()`  
 
 ```java
 private void scheduleFrameLocked(long now) {
        if (!mFrameScheduled) {
            mFrameScheduled = true;
            if (USE_VSYNC) {
                ...
                if (isRunningOnLooperThreadLocked()) {
                    scheduleVsyncLocked();
                } else {
                    Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                    msg.setAsynchronous(true);
                    mHandler.sendMessageAtFrontOfQueue(msg);
                }
            } else {
                final long nextFrameTime = Math.max(
                        mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
                Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, nextFrameTime);
            }
        }
    }
 ```
 默认情况下`USE_VSYNC`为true，当然设备厂商也可以设置为 false。如果 为false 会以 10 ms 为间隔计算下一次 doFrame 的时间，然后使用Handler来处理。  
 
 我们看看 ` scheduleVsyncLocked()` 这行代码做了何事?  
 
 ```java
 private void scheduleVsyncLocked() {
        mDisplayEventReceiver.scheduleVsync();
 }
 ```
 `mDisplayEventReceiver` 当`USE_VSYNC `为true 时 实例化成 `FrameDisplayEventReceiver`对象 ，它主要是和 native层 做交互，协调 vsync 信号。  
 `mDisplayEventReceiver.scheduleVsync()` 请求当下一帧开始时同步vsync信号。
 
 当vsync 信号来时 回调 JNI 层会回调`DisplayEventReceiver.dispatchVsync`方法 
 
 ```java
 // Called from native code.
    @SuppressWarnings("unused")
    private void dispatchVsync(long timestampNanos, int builtInDisplayId, int frame) {
        onVsync(timestampNanos, builtInDisplayId, frame);
    }
  
  public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
    }
 ```
 
 `FrameDisplayEventReceiver`实现了`doVsync`方法
 
 ```java
 private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
        private boolean mHavePendingVsync;
        private long mTimestampNanos;
        private int mFrame;

        public FrameDisplayEventReceiver(Looper looper) {
            super(looper);
        }

        @Override
        public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
            ....
            
            long now = System.nanoTime();
            if (timestampNanos > now) {
         			....
                timestampNanos = now;
            }

            if (mHavePendingVsync) {
                ....log
            } else {
                mHavePendingVsync = true;
            }

            mTimestampNanos = timestampNanos;
            mFrame = frame;
            Message msg = Message.obtain(mHandler, this);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
        }

        @Override
        public void run() {
            mHavePendingVsync = false;
            doFrame(mTimestampNanos, mFrame);
        }
    }
 ```
 主要实现:根据JNI层的 `timestampNanos`纳秒值计算成 毫秒，通过 Handler 执行
 `Runable`，即执行了` doFrame(mTimestampNanos, mFrame);`这行代码。  
 
 ```java
 void doFrame(long frameTimeNanos, int frame) {
        final long startNanos;
        synchronized (mLock) {
            if (!mFrameScheduled) {
                return; // no work to do
            }

            long intendedFrameTimeNanos = frameTimeNanos;
            startNanos = System.nanoTime();
            final long jitterNanos = startNanos - frameTimeNanos;
            if (jitterNanos >= mFrameIntervalNanos) {
                final long skippedFrames = jitterNanos / mFrameIntervalNanos;
                if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
     				....
                }
                final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
               ....
                frameTimeNanos = startNanos - lastFrameOffset;
            }

            if (frameTimeNanos < mLastFrameTimeNanos) {
			....
                scheduleVsyncLocked();
                return;
            }

            mFrameInfo.setVsync(intendedFrameTimeNanos, frameTimeNanos);
            mFrameScheduled = false;
            mLastFrameTimeNanos = frameTimeNanos;
        }

        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");
            AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);

            mFrameInfo.markInputHandlingStart();
            doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

            mFrameInfo.markAnimationsStart();
            doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);

            mFrameInfo.markPerformTraversalsStart();
            doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);

            doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
        } finally {
            AnimationUtils.unlockAnimationClock();
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
		.....
    }
 ```
 与动画相关的关键代码: 
   
 ```java
 mFrameInfo.markAnimationsStart();
            doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
 ```
 
 以下是 doCallbacks的实现，有没有感觉即将看见胜利的曙光？
 
 ```java
 void doCallbacks(int callbackType, long frameTimeNanos) {
        CallbackRecord callbacks;
        synchronized (mLock) {
            final long now = System.nanoTime();
            //取出可以执行的CallbackRecord 链表。
            callbacks = mCallbackQueues[callbackType].extractDueCallbacksLocked(
                    now / TimeUtils.NANOS_PER_MS);
            if (callbacks == null) {
                return;
            }
            mCallbacksRunning = true;
				....略去与动画无关的代码
        }
        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, CALLBACK_TRACE_TITLES[callbackType]);
            for (CallbackRecord c = callbacks; c != null; c = c.next) {
					.... log
					//循环执行链表
                c.run(frameTimeNanos);
            }
        } finally {
            synchronized (mLock) {
                mCallbacksRunning = false;
                do {
                    final CallbackRecord next = callbacks.next;
                    recycleCallbackLocked(callbacks);
                    callbacks = next;
                } while (callbacks != null);
            }
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
 ```
 
 所以该方法最核心的功能是找到需要执行的 `CallbackRecord`链表，然后循环执行 它们的 `run`方法。
 
 ```java
 private static final class CallbackRecord {
        public CallbackRecord next;
        public long dueTime;
        public Object action; // Runnable or FrameCallback
        public Object token;

        public void run(long frameTimeNanos) {
            if (token == FRAME_CALLBACK_TOKEN) {
                ((FrameCallback)action).doFrame(frameTimeNanos);
            } else {
                ((Runnable)action).run();
            }
        }
    }
 ```
 在前面 注册回调`postFrameCallbackDelayed`时 token 是 `FRAME_CALLBACK_TOKEN` 所以执行 run 方法中的第一个if 分支`    ((FrameCallback)action).doFrame(frameTimeNanos);`
  回调 	`FrameCallback.doFrame(frameTimeNanos)` 方法。
  下面是 `Choreographer.FrameCallback` 的实现
  
  ```java
  private final Choreographer.FrameCallback mFrameCallback = new Choreographer.FrameCallback() {
        @Override
        public void doFrame(long frameTimeNanos) {
            doAnimationFrame(getProvider().getFrameTime());
            if (mAnimationCallbacks.size() > 0) {
                getProvider().postFrameCallback(this);
            }
        }
    };
  ```
 