## Flow

[toc]

### 1.Channel 热流

- 单个事件只能被单次消费（点对点通信）

- 多个 launch 共享一个 Channel，类似于 **多线程队列BlockingQueue消费**，一般不这么用
- channel.receiveAsFlow 多处收集还是遵循**每个数据项只会被** **一个消费者处理**
- channel.consumeAsFlow 多处消费会抛异常  
- Capacity 可以让channel具备「预生产数据」的能力，允许发送方先生产一部分，send不会立即挂起
- Channel(capacity=1,onBufferOverflow=BufferOverflow.DROP_OLDEST) == Channel(CONFLATED)

### 2.Sequence 对比 list

- 消极生产（yield），相对于buildList**（或 List 构造）**的先生产

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

- **map vs transform.    transfrom 可以处理emit过程，map只能对发射值处理**

- **Reduce/fold 大火收汁/折叠. === Fold允许初始值参与reduce**

- Scan > fold 比fold更灵活

- **chunked 按个数分包 `T->List<T>`**

  

### 6.合并操作符

**🔥 flatMapConcat vs flatMapMerge **

| **对比项**   | **`concat`（按顺序执行）**                  | **`merge`（并行执行）**                                     |
| ------------ | ------------------------------------------- | ----------------------------------------------------------- |
| **执行方式** | 串行                                        | 并行执行                                                    |
| **适用场景** | 任务必须按顺序执行（如本地缓存 → 网络请求） | 多个 Flow 可同时进行（如并行数据获取，对UIEventFlow的合并） |
| **性能**     | 等待前一个 Flow 结束，可能较慢              | 多个 Flow 并行，性能更优                                    |
| **数据顺序** | 严格按照 Flow 顺序                          | 按各自的速度输出，顺序不保证                                |



**🚀flatten**
处理**Flow<Flow>**，将若干个flow 铺平

flattenConcat()/flattenMerge


**🚀 zip vs combine**


| **功能对比** | **`zip`（一一配对）**                                      | **`combine`（最新值对计算）**                                |
| ------------ | ---------------------------------------------------------- | ------------------------------------------------------------ |
| **合并逻辑** | 按索引一一匹配                                             | 取两个 Flow 的最新值                                         |
| **数据节奏** | **两个 Flow 都有新值才能触发collect**                      | **任意一个 Flow 发射新值**，combine 就会重新计算 **最新值组合后的结果**，前提是两个flow都有值 |
| **适用场景** | 需要严格配对的数据（比如 API 请求 vs. 响应、双人配对游戏） | 需要最新数据参与计算（比如超速场景：车速 combine 限速）      |



### 7.Flow的异常管理

```kotlin
flow {
    try {
        emit(data) // ❌ 不要这样做
    } catch (e: Throwable) {
        // 会吞掉下游 collect 时可能捕获的异常
    }
}
```

- 生产者不要掩盖异常，不要用tryCatch包裹emit
  如果必须要try catch 需要再次 throw出去
- catch操作符，只捕获上游异常

### 8.retry/retyWhen

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

### 9.onStart/onCompletion

- 启动之前的准备工作
- 完成后的善后工作，能拿到throwable，不影响catch{}

### 10.flowOn

- 修改coroutineContext
- 连续flowOn是右边 + 左边，即连续重复的话以左边为准

### 11.buffer（背压控制，缓冲区挂起/丢弃上游数据，避免数据堆积）

​	连续buffer以右边为准

```kotlin
conflate()
buffer(CONFLATED) //简单地丢弃旧数据,效果完全一致
```

### 12.collectLatest（取消未完成的collect）

取消前一个数据项的处理并开始处理最新的数据项，不会影响生产过程，
即在极限情况下，即便一直生产，结果是只处理了最后一个事件

```kotlin
scrollingFlow
    .collectLatest { scrollPosition ->
        // 执行滚动动画更新，只处理最后一次的滚动
        updateScrollPosition(scrollPosition)
    }
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

//注意区分缓存与replay
//总容量 = replay + extraBufferCapacity，两个区域相互隔离
//当这个容量被占满，并且被消费者挂起时候，再调用 emit(value)，会触发被压策略
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
**不同点：flow每次 collect 都重新执行，sharedFlow数据只生产一次，广播给所有订阅者**

### 14.StateFlow

- 场景：用来做**状态订阅** 

- 缓冲与缓存都只为1的SharedFlow （**收集行为视角**）
  ```kotlin
  MutableSharedFlow<T>(
      replay = 1,//新的 collect 会立即收到最近的一条数据，类似 StateFlow.value
      extraBufferCapacity = 1//可以缓冲 1 条新值，不阻塞 emit()，具备一定“并发承压”能力
  )
  ```

| **特性**        | SharedFlow                  | StateFlow                              |
| --------------- | --------------------------- | -------------------------------------- |
| 本质            | 多播的事件流（无状态）      | 多播的状态流（有状态）                 |
| 是否有当前值    | 没有                        | ✅ 永远有一个当前值                     |
| 是否必须初始化  | 不需要                      | ✅ 必须提供初始值                       |
| 是否粘性        | ⚠️ 可选（通过 replay 设置）  | ✅ 永远粘性（新订阅立即收到当前值）     |
| 是否支持 replay | ✅ 自定义 replay 缓冲区      | 固定为 1（当前值）                     |
| 用途            | 事件（如导航、Toast、点击） | 状态（如 UI 状态、登录状态、页面数据） |
| .value 可读写   | 不可                        | 可                                     |
