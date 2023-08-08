Glide

https://juejin.cn/user/4371313961737181 大佬



![glide缓存命中](..\image\glide缓存命中.jpg)

ActiveResourceCache 作用：避免使用中的图片被MemoryLruCache   Recycle了，其容量没有限制

ActiveResourceCache 为何使用弱引用？ 

通过 Weak构造的referenceQueue 监测Resource引用数，计数为0时，在被GC回收之前，将EngineResource.resource放入MemoryCache （类似LeakCanary）

```kotlin

class KeyedWeakReference(
  referent: Any,
  val key: String,
  val description: String,
  val watchUptimeMillis: Long,
  referenceQueue: ReferenceQueue<Any>
) : WeakReference<Any>(
  referent, referenceQueue
)
```


```java
//在单线程线程池执行， 
@Synthetic
  void cleanReferenceQueue() {
    while (!isShutdown) {
      try {
      	//public WeakReference(T referent, ReferenceQueue<? super T> q)
				//如果没有元素，则阻塞线程
        ResourceWeakReference ref = (ResourceWeakReference) resourceReferenceQueue.remove();
        cleanupActiveReference(ref);
 		
      } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
      }
    }
  }
```

