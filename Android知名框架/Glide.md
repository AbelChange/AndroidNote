Glide

- LruCache

​		基于LinkedHashMap实现,遍历顺序默认为插入顺序
​		构造为访问顺序(accessOrder)

```java
 private final Map<T, Entry<Y>> cache = new LinkedHashMap<>(100, 0.75f, true);
```

这样使用get之后会将数据放在尾巴，在trim2size之后，会移除head元素

https://juejin.cn/user/4371313961737181 大佬



![glide缓存命中](..\image\glide缓存命中.jpg)

ActiveResourceCache 作用？
Map<Key, ResourceWeakReference>
- 其容量没有限制，可以避免使用中的图片被MemoryLruCache回收机制Recycle了
- 分层效率

ActiveResourceCache 为何使用弱引用？ 

- 更快地释放内存，减少内存压力

- 通过 WeakReference构造中的referenceQueue 监测Resource引用数，计数为0时，在被GC回收之前，将EngineResource.resource放入MemoryCache （类似LeakCanary），然后放到memorycache



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

