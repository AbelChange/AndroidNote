

## Handler机制

[TOC]

```

### Handler机制

```java
class ActivityThread{
    public static void main(String[] args) {
		//looper创建：new之后通过threadLocal绑定当前线程绑定当前线程
		Looper.prepareMainLooper(); 
		new Handler(); //此处判断是否已经创建looper,如果未创建则抛异常
        Looper.loop();//Looper持有MessageQueue,将消息不断的取出，发送到Message的target(Handler)
    }
} 
```

- 

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
		//如果when == 0 或者小于队头when,则直接放入队列头部，会优先执行
		//否则 根据when大小插入队列

		//唤醒队列
		nativeWake(mPtr);
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
    //这里保证了异步消息的优先级最大，但是这里可以看出 异步消息 顺序不能保证，因为并没有插入队列的处理，只是取的时候进行了判断
//如果需要保证顺序执行，使用同步Handler
//其他情况可以使用异步消息，减轻looper压力
    Message next() {
		int nextPollTimeoutMillis = 0;
        for (;;) {
            //阻塞等待超时唤醒
            nativePollOnce(ptr, nextPollTimeoutMillis);
            synchronized (this) {
                //关键地方，存在同步屏障，不移除，寻找下一个异步消息返回。
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
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



### handler泄露引用链条

- 主线程 —>（ThreadLocal中的Looper）MessageQueue —> Message —> Handler —> Activity
- 注意区分匿名子线程 对外部类的引用 导致的泄露

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







