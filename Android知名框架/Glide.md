Glide




https://juejin.cn/user/4371313961737181 大佬

![glide缓存命中](..\image\glide缓存命中.jpg)

ActiveResource 作用：避免正在使用的图片被LruCache Recycle了

为何使用弱引用： 

 监听EnginResource引用数，计数为0时，在被GC回收之前，将EngineResource.resource放入LruCache


```java
  @Synthetic
  void cleanReferenceQueue() {
    while (!isShutdown) {
      try {
      	//public WeakReference(T referent, ReferenceQueue<? super T> q)
        ResourceWeakReference ref = (ResourceWeakReference) resourceReferenceQueue.remove();
        cleanupActiveReference(ref);
 		
      } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
      }
    }
  }
```

