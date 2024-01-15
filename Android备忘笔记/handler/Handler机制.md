## Handler机制

[TOC]

![handler模式](https://img-blog.csdnimg.cn/e44e8cb0d48e40819dc10b792e118ee4.png)

### 1.消息入队

```
Handler#sendMessage()
	->Handler#sendMessageDelayed(msg, 0)
     	->Handler#sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis)
     		->MessageQueue#.enqueueMessage(msg, when) 
```



```java
public final class MessageQueue {
    boolean enqueueMessage(Message msg, long when) {
	 		if (p == null || when == 0 || when < p.when) {
                //当前消息when已到
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // 存在同步屏障，并且当前消息是异步消息，则需要唤醒
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
				//遍历插入
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // 执行唤醒
            if (needWake) {
                nativeWake(mPtr);
            }
    }
}
```

### 2.轮询消息


```java
//开启轮询

​```java
class ActivityThread{
    public static void main(String[] args) {
		//looper创建：new之后通过threadLocal绑定当前线程绑定当前线程
		Looper.prepareMainLooper(); 
		new Handler(); //此处判断是否已经创建looper,如果未创建则抛异常
        Looper.loop();//Looper持有MessageQueue,将消息不断的取出，发送到Message的target(Handler)
    }
} 
```



```java
public final class Looper {
   final Looper me = myLooper();
   final MessageQueue queue = me.mQueue;
   public static void loop() {
       //....
       for (;;) {
           if (!loopOnce(me, ident, thresholdOverride)) {
               return;
           }
   }
		
   private static boolean loopOnce() {
       Message msg = me.mQueue.next(); //block here
       if (msg == null) {
           // No message indicates that the message queue is quitting.
           return false;
       }
       //取到消息后开始分发到target
       // logging + observer
       //dispatch start
       msg.target.dispatchMessage(msg);
       //dispatch end
 
    }
}
```



```java
public final class MessageQueue {
	
    Message next() {
		
        for (;;) {
				//发现同步屏障，优先异步消息
			   if (msg != null && msg.target == null) {
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
				//判断when
				if(未到时间){
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                }else{
                     return msg;
                }
		
				//这里如果有消息未到时间，则通知Idle，返回值代表是否是单次任务（自取消）
				try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
        }

    }


	//同步屏障消息，无target,立即执行
	//绘制消息，input消息等
    public int postSyncBarrier() {
        return postSyncBarrier(SystemClock.uptimeMillis());
    }

}
```



### 3.handler泄露引用链条

- 主线程 —>（ThreadLocal中的Looper）MessageQueue —> Message —> Handler —> Activity
- 注意区分匿名子线程 对外部类的引用 导致的泄露

### 4.IdleHandler之GC时机

```java

final class GcIdler implements MessageQueue.IdleHandler {
     //当线程将要进入block时候，消息未到执行时间
        @Override
        public final boolean queueIdle() {
            doGcIfNeeded();
			//调用后是否自动移除 
            return false;
        }
}
```
