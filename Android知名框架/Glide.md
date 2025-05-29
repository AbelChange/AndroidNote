Glide

- LruCache

​		基于LinkedHashMap实现,遍历顺序默认为插入顺序
​		这里使用构造访问顺序(accessOrder)
​		这样使用get之后会将数据放在尾部，在put导致的trim2size之后，会移除头部元素

```java
 private final Map<T, Entry<Y>> cache = new LinkedHashMap<>(100, 0.75f, accessOrder = true);
```



https://juejin.cn/user/4371313961737181 大佬



![glide缓存命中](img/Glide/glide%E7%BC%93%E5%AD%98%E5%91%BD%E4%B8%AD.jpg)

3.Glide三级缓存，

MemoryCache：

- ActiveResources（活动缓存，弱引用缓存，提高命中效率）

  为何使用弱引用？ 

  > 资源以 **WeakReference + ReferenceQueue** 管理，GC回收时可感知。
  >
  > 如果放到lru里会被主动回收，如果图片在使用中就会出问题

- Memorycache（Lru）

  主动回收图片

DiskCache：

- Resource 解码，转换后的图片
- Data 解码之前的数据

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





• 页面生命周期 fragment管理



 当页面Stop时暂停请求；页面resume恢复请求；页面销毁时销毁请求。

 • 网络连接状态
 如果应用有监控网络状态的权限，那么 Glide 会监听网络连接状态，并在网络重新连接时重新启动失败的请求。
 • 内存状态
 Glide 则根据系统内存紧张级别（level）进行 memoryCache / bitmapPool / arrayPool 的回收，而 RequestManager 在 TRIM_MEMORY_MODERATE 级别会暂停请求

作者：Bfmall
链接：https://www.jianshu.com/p/660117e7b1c1
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
