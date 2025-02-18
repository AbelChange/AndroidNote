## Flow

[toc]



### 1.Channel 热流

- 多个 launch 共享一个 Channel，但**每个数据项只会被** **一个消费者处理**，类似于 **多线程队列BlockingQueue消费**，一般不这么用
- Capacity 可以让channel具备预生产数据的能力
- Channel(capacity=1,onBufferOverflow=BufferOverflow.DROP_OLDEST) == Channel(CONFLATED)
- channel.receiveAsFlow 多处收集还是遵循**每个数据项只会被** **一个消费者处理**
- channel.consumeAsFlow 多处消费会抛异常  

### 2.Sequence 

- 消极生产（yield），相对于buildList的先生产

### 3.Flow 一般是冷流

- 每次 collect  分别启动互不干扰/分散的生产
- ***冷热辨别：上游生产是否依赖于下游消费***
- ***冷/热不能从类型上进行判断，应该从 业务 角度去判断***

### 4.channelFlow{ }/callBackFlow

- 内部可以启动子协程，支持跨协程

- 而flow{}不可以，开启协程进行emit会出现Emission from another coroutine is detected 
- 搭配javaCallBack,但是一般是使用suspendableCoroutineScope{},如果是多次连续的回调可以使用callBackFlow{}

```kotlin
channelFlow{
		waitJavaCallBack(object:JavaCallBack{
				onSuccess(data:String){
					trySend()
					close()
				}
				onFail(error:Throwable){
					cancel(CancelationException(error))
				}
		})
		//挂起协程，暂不允许退出
		awaitClose()
}
```



```kotlin
val sharedFlow = MutableSharedFlow<Int>(
    replay = 0,//>0时候相当于粘性数据 
    extraBufferCapacity = 0,//消费慢的时候，多余的事件进入缓冲区
    onBufferOverflow = BufferOverflow.SUSPEND //背压策略
)

//bufferCapacity = replay + extraBufferCapacity

public enum class BufferOverflow {
    /**
     * Suspend on buffer overflow.
     */
    SUSPEND,//挂起emit等到消费之后才继续emit

    /**
     * Drop **the oldest** value in the buffer on overflow, add the new value to the buffer, do not suspend.
     */
    DROP_OLDEST,

    /**
     * Drop **the latest** value that is being added to the buffer right now on buffer overflow
     * (so that buffer contents stay the same), do not suspend.
     */
    DROP_LATEST
}

```

//StateFlow/LiveData 可以认为**`replay`**为1，无缓冲即extraBufferCapacity的sharedFlow

### 5.操作符

- **dropWhile/drop  takeWhile/take  断->流 ，流->断.  takeWhile <= tranformWhile**
- **map vs transform.    transfrom  > map 更自由的emit**
- **Reduce/fold 大火收汁/折叠. === Fold允许初始值参与reduce**
- Scan > fold 比fold更灵活
- **chunked 按个数分包 `T->List<T>`**

### 6.Flow的异常管理

- 生产者不要掩盖异常，不要用tryCatch包裹emit，这样会覆盖掉下游试图catch collect的异常
  如果必须要try catch 需要再次 throw出去
- catch操作符，只捕获上游异常

### 7.retry/retyWhen

碰到异常后重启flow流程，从源头重新开始
```kotlin
flow {
    emit(1)
    throw RuntimeException("Error occurred") // 模拟错误
}
.retryWhen { cause, attempt ->
    if (attempt < 3) { // 允许重试 3 次
        val delayTime = (attempt + 1) * 500L // 500ms, 1000ms, 1500ms
        println("Retrying after $delayTime ms due to ${cause.message}")
        delay(delayTime) // 延迟一段时间再重试
        true
    } else {
        false // 超过 3 次则不再重试
    }
}
.collect { value ->
    println("Received: $value")
}
```

### 8.onStart/onCompletion

- 启动之前的准备工作
- 完成后的善后工作，能拿到throwable，不影响catch{}

### 9.flowOn

- 修改coroutineContext
- 连续flowOn是右边 + 左边，即连续重复的话以左边为准

### 10.buffer（背压控制，缓冲区挂起/丢弃上游数据，避免数据堆积）

​	连续buffer以右边为准

```kotlin
conflate()
buffer(CONFLATED) //简单地丢弃旧数据,效果完全一致
```

### 11.collectLatest（取消未完成的collect）

取消前一个数据项的处理并开始处理最新的数据项，不会影响生产过程，
即在极限情况下，即便一直生产，结果是只处理了最后一个事件

```kotlin
scrollingFlow
    .collectLatest { scrollPosition ->
        // 执行滚动动画更新，只处理最后一次的滚动
        updateScrollPosition(scrollPosition)
    }
```

### 12.合并操作符

**flatten相关 ==》展开铺平**

**🚀 zip vs combine**


| **功能对比**   | **`zip`（等待配对）**                                      | **`combine`（最新值计算）**                                  |
| -------------- | ---------------------------------------------------------- | ------------------------------------------------------------ |
| **合并逻辑**   | 按索引一一匹配                                             | 取两个 Flow 的最新值                                         |
| **数据节奏**   | 必须等另一方有数据才能继续                                 | 任意 Flow 更新，都会触发计算                                 |
| **适用场景**   | 需要严格配对的数据（比如 API 请求 vs. 响应、双人配对游戏） | 需要最新数据参与计算（比如车速 vs. 限速、股票价格 vs. 账户余额） |
| **流的独立性** | 需要两个 Flow 都有值才能继续                               | 不需要等待，只要有新数据就合并                               |

**🔥 concat vs merge **
flattenConcat()/flattenMerge/flattenMapConcat/flattenMapLatest/flattenMapMerge

| **对比项**   | **`concat`（按顺序执行）**                  | **`merge`（并行执行）**                                     |
| ------------ | ------------------------------------------- | ----------------------------------------------------------- |
| **执行方式** | 串行                                        | 并行执行                                                    |
| **适用场景** | 任务必须按顺序执行（如本地缓存 → 网络请求） | 多个 Flow 可同时进行（如并行数据获取，对UIEventFlow的合并） |
| **性能**     | 等待前一个 Flow 结束，可能较慢              | 多个 Flow 并行，性能更优                                    |
| **数据顺序** | 严格按照 Flow 顺序                          | 按各自的速度输出，顺序不保证                                |

- 热流引起的资源浪费 解决


```kotlin
    /**
     * 下游停止收集时取消流,2000是为了防止配置修改引起的意外取消
     * 如果downStream  downStream2 都不再收集 才 取消该流
     * 可避免重复创建上游带来的资源浪费
     */
    private val _whileSubscribed: Flow<Int> = flow<Int> {
        emit(1)
        emit(2)
        delay(3000)
        emit(3)
    }.shareIn(viewModelScope, SharingStarted.WhileSubscribed(2000))
        .distinctUntilChanged { old, new ->
            old - new == 0
        }

    val downStream: Flow<String> = _whileSubscribed.map { it.toString() }

    val downStream2: Flow<String> = _whileSubscribed.map { it.toString() }
```

### 13.SharedFlow

- 场景：用来做**事件订阅** *collector is called a subscriber*

```kotlin
public fun <T> Flow<T>.launchIn(scope: CoroutineScope): Job {
   return scope.launch {
      collect()
    }
}

/**
 * Sharing is started immediately and never stops.
 */
public val Eagerly: SharingStarted = StartedEagerly()

/**
 * Sharing is started when the first subscriber appears and never stops.
 */
public val Lazily: SharingStarted = StartedLazily()

/**
 * Sharing is started when the first subscriber appears, immediately stops when the last
 * subscriber disappears (by default), keeping the replay cache forever (by default).
 */
@Suppress("FunctionName")
public fun WhileSubscribed(
    stopTimeoutMillis: Long = 0,
    replayExpirationMillis: Long = Long.MAX_VALUE
): SharingStarted =
    StartedWhileSubscribed(stopTimeoutMillis, replayExpirationMillis)

public fun <T> Flow<T>.shareIn(
    scope: CoroutineScope,
    started: SharingStarted,//Lazily/Eagerly
    replay: Int = 0  
): SharedFlow<T> 

//注意区分缓冲与缓存
public fun <T> MutableSharedFlow(
    replay: Int = 0,
    extraBufferCapacity: Int = 0,
    onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND
): MutableSharedFlow<T>


/**
 * Represents this mutable shared flow as a read-only shared flow.
 *	给外部暴漏订阅 而不暴漏emit
 */
public fun <T> MutableSharedFlow<T>.asSharedFlow(): SharedFlow<T> =
    ReadonlySharedFlow(this, null)
```

**相同点：可以被收集多次**
**不同点：flow会重新启动数据生产，sharedFlow数据生产流程是单例的**

### 14.StateFlow

- 场景：用来做**状态订阅** 、
- 缓冲与缓存都只为1的SharedFlow

