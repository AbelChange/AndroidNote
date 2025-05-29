## OkHttp源码解读

![okHttp拦截器](img/OkHttp%E7%9A%84%E4%BD%BF%E7%94%A8/%E6%8B%A6%E6%88%AA%E5%99%A8%E9%93%BE.webp)

![okHttp拦截器](img/OkHttp%E7%9A%84%E4%BD%BF%E7%94%A8/okHttp%E6%8B%A6%E6%88%AA%E5%99%A8.webp)

| **特性** | addInterceptor              | addNetworkInterceptor                    |
| -------- | --------------------------- | ---------------------------------------- |
| 层级     | 应用层                      | 网络层                                   |
| 用途     | 添加统一 header、日志、重试 | 网络内容修改、监控下载、响应 header 操作 |



### 1.基本使用

```java
client.newCall(request).execute(); 
client.newCall(request).enqueue(Callback responseCallback);         
```

### 2.源码分析

#### 2.1拦截器

```kotlin
 @Throws(IOException::class)
  internal fun getResponseWithInterceptorChain(): Response {
    // Build a full stack of interceptors.
    val interceptors = mutableListOf<Interceptor>()
		//通过addInterceptor加入的拦截器
    interceptors += client.interceptors
		//对某些错误进行重试，这些错误包括401、408 、部分3XX的错误
		//可以通过retryOnConnectFailure进行配置是否生效，默认有效
    interceptors += RetryAndFollowUpInterceptor(client) 
	  //处理header
    interceptors += BridgeInterceptor(client.cookieJar)
		//缓存处理
    interceptors += CacheInterceptor(client.cache)
		//判断是复用连接还是新建连接
    interceptors += ConnectInterceptor
    if (!forWebSocket) {
			//通过addNetworkInterceptor加入的拦截器
      interceptors += client.networkInterceptors
    }
		//发起请求
    interceptors += CallServerInterceptor(forWebSocket)

    val chain = RealInterceptorChain(
        call = this,
        interceptors = interceptors,
        index = 0,
        exchange = null,
        request = originalRequest,
        connectTimeoutMillis = client.connectTimeoutMillis,
        readTimeoutMillis = client.readTimeoutMillis,
        writeTimeoutMillis = client.writeTimeoutMillis
    )

    var calledNoMoreExchanges = false
    try {
      val response = chain.proceed(originalRequest)
      if (isCanceled()) {
        response.closeQuietly()
        throw IOException("Canceled")
      }
      return response
    } catch (e: IOException) {
      calledNoMoreExchanges = true
      throw noMoreExchanges(e) as Throwable
    } finally {
      if (!calledNoMoreExchanges) {
        noMoreExchanges(null)
      }
    }
  }


```

#### 2.2Dispatcher控制call执行分发

   ```java

public final class Dispatcher {
 internal fun enqueue(call: AsyncCall) {
    synchronized(this) {
	 //加入readyQueue
      readyAsyncCalls.add(call)

      // Mutate the AsyncCall so that it shares the AtomicInteger of an existing running call to
      // the same host.
      if (!call.call.forWebSocket) {
        val existingCall = findExistingCallWithHost(call.host)
        if (existingCall != null) call.reuseCallsPerHostFrom(existingCall)
      }
    }
    promoteAndExecute()
  }

  private boolean promoteAndExecute() {
    assert (!Thread.holdsLock(this));

    List<AsyncCall> executableCalls = new ArrayList<>();
    boolean isRunning;
    synchronized (this) {
	//	遍历readyAsyncCalls，判断是否满足下列条件，如果满足则加入runningAsyncCalls
      for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
        AsyncCall asyncCall = i.next();
        if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.def = 64
        if (asyncCall.callsPerHost().get() >= maxRequestsPerHost) continue; // Host max capacity. def=5
        i.remove();
        asyncCall.callsPerHost().incrementAndGet();
        executableCalls.add(asyncCall);
        runningAsyncCalls.add(asyncCall);
      }
      isRunning = runningCallsCount() > 0;
    }
	// 放入线程池执行
    for (int i = 0, size = executableCalls.size(); i < size; i++) {
      AsyncCall asyncCall = executableCalls.get(i);
      asyncCall.executeOn(executorService());
    }
    return isRunning;
  }

//执行AsyncCall的线程池
  public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }

}
   ```



4.连接池


____

### OkHttp缓存策略

> 服务器支持缓存，即响应的字段中包含Header:Cache-Control, max-age=xxx,(本地存储时间)

```java
OkHttpClient okHttpClient = new OkHttpClient();
OkHttpClient newClient = okHttpClient.newBuilder()
               .cache(new Cache(mContext.getCacheDir(), 10240*1024))
               .connectTimeout(20, TimeUnit.SECONDS)
               .readTimeout(20, TimeUnit.SECONDS)
               .build();
```



> 服务器不支持缓

```java
//需要使用Interceptor来重写Respose的头部信息，从而让okhttp支持缓存。
public class CacheInterceptor implements Interceptor {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        Response response = chain.proceed(request);
        Response response1 = response.newBuilder()
                .removeHeader("Pragma")
                .removeHeader("Cache-Control")
                //cache for 30 days
                .header("Cache-Control", "max-age=" + 3600 * 24 * 30)
                .build();
        return response1;
    }
```















