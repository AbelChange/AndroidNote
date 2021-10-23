

## 绘制流程与Handler

[TOC]



### ActivityThread源码解析

```java

public class ActivityThread {
    //内部类 ApplicationThread收到AMS发来的Binder消息，
   // H 收到后handleLaunchActivity
    private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        //构造Acitvity，回调onCreate，将contentview放到decorview中
        Activity a = performLaunchActivity(r, customIntent);
        handleResumeActivity()
    }

        final void handleResumeActivity() {
            //1、调用activity的onResume方法，会调用到Activity的onResume方法
            ActivityClientRecord r = performResumeActivity(token, clearHide);
            //......
            if (r != null) {
                final Activity a = r.activity;
                final int forwardBit = isForward ?
                        WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;
                //.......................
                if (r.window == null && !a.mFinished && willBeVisible) {
                    r.window = r.activity.getWindow();
                    View decor = r.window.getDecorView();
                    //2、decorView先暂时隐藏
                    decor.setVisibility(View.INVISIBLE);
                    ViewManager wm = a.getWindowManager();
                    WindowManager.LayoutParams l = r.window.getAttributes();
                    a.mDecor = decor;
                    l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                    l.softInputMode |= forwardBit;
                    if (a.mVisibleFromClient) {
                        a.mWindowAdded = true;
                        //3、关键函数 添加到window
                        wm.addView(decor, l);//windowmanagerGlobal.addView -> ViewRootImpl.setView
                    }
                    //..............
                    r.activity.mVisibleFromServer = true;
                    mNumVisibleActivities++;
                    if (r.activity.mVisibleFromClient) {
                        //添加decorView之后，设置可见，从而显示了activity的界面
                        r.activity.makeVisible();
                    }
                }
            }

        }
    }
```



```java

class ViewRootImpl{
   // setView()-->requestLayout()-->scheduleTraversals()
        void scheduleTraversals() {
                if (!mTraversalScheduled) {
                    mTraversalScheduled = true;
                    //1.发送同步屏障Msg,该Msg没有Target
                    mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
                    //2.将TraversalRunnable存放到Choreographer的“待执行队列中”
                    //Choreographer收到Vsync信号时将发送异步消息（绘制任务）到主线程执行
                    mChoreographer.postCallback(
                            Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
                    if (!mUnbufferedInputDispatch) {
                        scheduleConsumeBatchedInput();
                    }
                    notifyRendererOfFramePending();
                    pokeDrawLockIfNeeded();
                }
            }

            final class TraversalRunnable implements Runnable {
                @Override
                public void run() {
                    doTraversal();
                }
            }

         void doTraversal() {
                if (mTraversalScheduled) {
                    mTraversalScheduled = false;
                    //移除同步屏障并执行三大流程
                    mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
                    performTraversals();
                }
          }

         private void performTraversals() {
                 if (mFirst) {
					//重点:之后view.Post的Runnable才会执行,才能拿到view尺寸
					//同理，当View的mAttachInfo ！=null时也说明肯定完成过UI绘制
					dispatchAttachedToWindow()
                 }				

                //1. View树的测量-可以不测量直接走布局和绘制
                if (mFirst || windowShouldResize || insetsChanged || viewVisibilityChanged || ...) {
                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                    layoutRequested = true;  //需要进行布局
                }
                //2. View树的布局-可以不布局直接走绘制
                final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
                if (didLayout) {
                    performLayout(lp, mWidth, mHeight);
                }
                //3. View树的绘制-可以不进行绘制
                boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;
                if (!cancelDraw && !newSurface) {
                    performDraw();
                }
        }
}

```

### Handler机制

- Handler构造需要Looper（子线程创建Handler异常）
- Looper.prepare()将Looper绑定到当前线程（ThreadLocal）
- Looper持有MessageQueue,将消息不断的取出，发送到Message的target(Handler)

```java
public final class Looper {
        final Looper me = myLooper();
        final MessageQueue queue = me.mQueue;
        public static void loop() {
            for (;;) {
			// might block
                Message msg = queue.next(); 
               if (msg == null) {
                    return;
                }
			   // 开始分发
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
                //分发超过16ms 认为卡（因为安卓是消息驱动的，绘制消息 无法16ms 执行则认为发生了卡顿）
                try {
                    msg.target.dispatchMessage(msg);
                } finally {
                    if (traceTag != 0) {
                        Trace.traceEnd(traceTag);
                    }
                }
			  //分发结束
			 logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }
        }
    }
```

```java
public final class MessageQueue {

    boolean enqueueMessage(Message msg, long when) {
		nativeWake(mPtr);
		//唤醒队列
    }
		
    private int postSyncBarrier(long when) {
        // Enqueue a new sync barrier token.
        // We don't need to wake the queue because the purpose of a barrier is to stall it.
        synchronized (this) {
            final int token = mNextBarrierToken++;
            final Message msg = Message.obtain();
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token;
            //...插入消息
            return token;
        }
    }
    //这里保证了异步消息的优先级最大
    Message next() {
		int nextPollTimeoutMillis = 0;
        for (;;) {
            //阻塞超时唤醒
            nativePollOnce(ptr, nextPollTimeoutMillis);
            synchronized (this) {
                //关键地方，存在同步屏障，先处理异步消息
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                //再处理同步消息
                if (msg != null) {
				   if(msg.when没到时间){
                      nextPollTimeoutMillis = msg.when
                 	}else{
                       return msg;
                   }
                }   
            }
        }
    }
}
```

### IdleHandler-LeakCanary 

```java
public final class MessageQueue{
      	private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();
		
   /**
     * Callback interface for discovering when a thread is going to block
     * waiting for more messages.
     */
    public static interface IdleHandler {
        /**
         * Called when the message queue has run out of messages and will now
         * wait for more.  Return true to keep your idle handler active, false
         * to have it removed.  This may be called if there are still messages
         * pending in the queue, but they are all scheduled to be dispatched
         * after the current time.
         */
        boolean queueIdle();
    }

		public void addIdleHandler(IdleHandler handler){
             synchronized (this) {
            	mIdleHandlers.add(handler);
       		 }
        }

        //手动移除
        public void removeIdleHandler(@NonNull IdleHandler handler) {
                synchronized (this) {
                    mIdleHandlers.remove(handler);
                }
        }
}
```

```java
//ActivityThread 使用该IdleHandler完成Gc
final class GcIdler implements MessageQueue.IdleHandler {
     //当messageQueue 用完所有的消息（不包括延迟消息）
        @Override
        public final boolean queueIdle() {
            doGcIfNeeded();
			//调用后是否自动移除 
            return false;
        }
}
```








