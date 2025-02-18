[TOC]

## RxJava2

### 1.简单理解

- 事件流向 Observable-->Obsever

- 建立契约  observable.subscribe(observer);  

- 链式



### 2.subcribe方法重载

```java
	//不关心任何事件	
	public final Disposable subscribe() {}
	//只关心onNext()
    public final Disposable subscribe(Consumer<? super T> onNext) {}
	//关心所有 onSubcribe(); onNext(); onError();onComplete();
    public final void subscribe(Observer<? super T> observer) {}
```

### 3.线程切换

```java
//线程切换
observable.subscribeOn(Schedulers.newThread())     
       //  .subscribeOn(Schedulers.io())      //多次调用subcribeOn  只有第一次  有效，一夫一妻         
         .observeOn(AndroidSchedulers.mainThread()) // //多次调用oberverOn，均有效切换线程
         .observeOn(Schedulers.io())                
         .subscribe(consumer);                                      
```

### 4.创建操作符

**创建被观察者对象**

![创建操作符](img/RxJava2/%E5%88%9B%E5%BB%BA%E6%93%8D%E4%BD%9C%E7%AC%A6.png)

#### 4.1  create

```java
 // 1. 通过creat（）创建被观察者对象
        Observable.create(new ObservableOnSubscribe<Integer>() {
            // 2. 在复写的subscribe（）里定义需要发送的事件
            @Override
            public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                    emitter.onNext(1);
                    emitter.onNext(2);
                    emitter.onNext(3);
                emitter.onComplete();
            }  // 至此，一个被观察者对象（Observable）就创建完毕
        }).subscribe(new Observer<Integer>() {
            // 3. 通过通过订阅（subscribe）连接观察者和被观察者
            @Override
            public void onSubscribe(Disposable d) {
                Log.d(TAG, "1.subscribe);
            }
   
            @Override
            public void onNext(Integer value) {
                Log.d(TAG, "2.onNext"+ value);
            }

            @Override
            public void onError(Throwable e) {
                Log.d(TAG, "对Error事件作出响应");
            }

            @Override
            public void onComplete() {
                Log.d(TAG, "3.对Complete事件作出响应");
            }
        });
    }
```

#### 4.2 just 10个以下事件快速创建

```java
 Observable.just (1,2,3,4).subcribe()
```

#### 4.3 fromArray,数组对象，10个以上事件，不可传list

#### 4.4 fromIterable,list

#### 4.5 defer  延迟创建Observable

#### 4.6 timer,subcribe后，延迟发送事件

#### 4.7 interval无限轮询

```java
//参数1 =   delay
//参数2 =  interval 
Observable.interval(3,1,TimeUnit.SECONDS)
              // [0，∞);
```

#### 4.8 range

```java
     // 参数1 = 起点
      // 参数2 = 事件总量
Observable.range(3,10)
```

#### 4.8 intervalRange有限计时

```java
        // 参数1 = 起点
        // 参数2 = 事件总量
        // 参数3 = delay
        // 参数4 = interval
        Observable.intervalRange(3,10,2, 1, TimeUnit.SECONDS)
            //[3,3+10-1);
```



### 5.0转换操作符

#### 5.1map 一对一转换

```java
Student[] students = ...;
Observable.from(students)
    .map(new Func1<Student, String>() {
        @Override
        public String call(Student student) {
            return student.getName();
        }
    })
    .subscribe(new Subscriber<String>() {
    @Override
    public void onNext(String name) {
        Log.d(tag, name);
    }
	...
});
```

#### 5.2flatmap 

将有序的事件转化为新的并行事件

- flatMap 会同时订阅并转换每个发射的项目，可以并行处理发射的项目，但不按照发射顺序。
- 提高并发性
- flatMap 可以交错合并多个 Observable 的发射物

```kotlin
Observable.just(1, 2, 3)
    .flatMap {
        Observable.just(it * 10)
            .subscribeOn(Schedulers.io())
    }
    .subscribe { println("flatMap result: $it") }

```

#### 5.3concatMap

- `concatMap` 会按顺序处理每个发射的项目，保证项目的顺序。

  

#### 5.4buffer

```java
Observable.just(1, 2, 3, 4, 5)
		// 参数1：缓存区大小
        //参数2： 步长
                .buffer(3, 1)

consumer的onNext中依次会123，234，345，45，5，之后onComplete();
```

### 6.组合操作符

![zip操作符](img/RxJava2/zip%E6%93%8D%E4%BD%9C%E7%AC%A6.webp)

|  操作符   | 执行方式 |          产物           |
| :----: | :--: | :-------------------: |
| concat |  串行  |      被选中的某一个生产者       |
| merge  |  并行  | 产物无序排列，0-2-1,需要自己合并产物 |
|  zip   |  并行  |       0a-1b-2c        |

#### 6.1concat/concat array

组合多个被观察者一起发送数据，串行执行，concat小于4个，concatArray >=4

```java
    /*
         * 通过concat（） 和 firstElement（）操作符实现缓存功能
         **/

        // 1. 通过concat（）合并memory、disk、network 3个被观察者的事件（即检查内存缓存、磁盘缓存 & 发送网络请求）
        //    并将它们按顺序串联成队列
        Observable.concat(memory, disk, network)
                // 2. 通过firstElement()，从串联队列中取出并发送第1个有效事件（Next事件），即依次判断检查memory、disk、network
                .firstElement()
                // 即本例的逻辑为：
                // a. firstElement()取出第1个事件 = memory，即先判断内存缓存中有无数据缓存；由于memoryCache = null，即内存缓存中无数据，所以发送结束事件（视为无效事件）
                // b. firstElement()继续取出第2个事件 = disk，即判断磁盘缓存中有无数据缓存：由于diskCache ≠ null，即磁盘缓存中有数据，所以发送Next事件（有效事件）
                // c. 即firstElement()已发出第1个有效事件（disk事件），所以停止判断。
                
                // 3. 观察者订阅
                .subscribe(new Consumer<String>() {
                    @Override
                    public void accept( String s) throws Exception {
                        Log.d(TAG,"最终获取的数据来源 =  "+ s);
                    }
                });
```



#### 6.2merge / mergeArray

- **按时间线合并产物**
- 0-->a->1->2->b->c

```java
         
        Observable.merge(network, file)
                .subscribe(new Observer<String>() {
                    @Override
                    public void onNext(String value) {
                        Log.d(TAG, "数据源有： "+ value  );
                        result += value + "+";
                    }
                    @Override
                    public void onError(Throwable e) {
                        Log.d(TAG, "对Error事件作出响应");
                    }
                    // 接收合并事件后，统一展示
                    @Override
                    public void onComplete() {
                        Log.d(TAG, "获取数据完成");
                        Log.d(TAG,  result  );
                    }
                });
```



#### 6.3concatDelayError（） / mergeDelayError（）

不delay时，如果一个观察者出现error，另一个终止发送，这里delay到所有观察者都完成再onError

#### 6.4zip操作符，这里将两个观察者产物解压后再展示

![zip操作符](img/RxJava2/zip%E6%93%8D%E4%BD%9C%E7%AC%A6.webp)

- 最终合并的事件数量 = 多个被观察者（`Observable`）中数量最少的数量
- 最终合并的事件数量 = 多个被观察者（`Observable`）中数量最少的数量
- 0a->1b->2c

#### 6.5combineLatest

- 保留所有`Observables`最新的数据，以便combine

  ```java
     /*
           * 步骤1：设置控件变量 & 绑定
           **/
          EditText name,age,job;
          Button confirm;
  
          name = (EditText) findViewById(R.id.name);
          age = (EditText) findViewById(R.id.age);
          job = (EditText) findViewById(R.id.job);
          confirm = (Button) findViewById(R.id.list);
  
          /*
           * 1. 此处采用了RxBinding：compile 'com.jakewharton.rxbinding2:rxbinding:2.0.0'
           * 2. 传入EditText控件，点击任1个EditText撰写时，都会发送数据事件 = Function3（）的返回值（下面会详细说明）
           * 3. 采用skip(1)原因：跳过 一开始EditText无任何输入时的空值
           **/
          Observable<CharSequence> nameObservable = RxTextView.textChanges(name).skip(1);
          Observable<CharSequence> ageObservable = RxTextView.textChanges(age).skip(1);
          Observable<CharSequence> jobObservable = RxTextView.textChanges(job).skip(1);
  
          /*
           * 步骤3：通过combineLatest（）合并事件 & 联合判断
           **/
          Observable.combineLatest(nameObservable,ageObservable,jobObservable,new Function3<CharSequence, CharSequence, CharSequence,Boolean>() {
              @Override
              public Boolean apply(@NonNull CharSequence charSequence, @NonNull CharSequence charSequence2, @NonNull CharSequence charSequence3) throws Exception {
                 //规定表单信息输入不能为空        
                  boolean isUserNameValid = !TextUtils.isEmpty(name.getText()) ;
                  boolean isUserAgeValid = !TextUtils.isEmpty(age.getText());
                  boolean isUserJobValid = !TextUtils.isEmpty(job.getText()) ;
                  return isUserNameValid && isUserAgeValid && isUserJobValid;
              }
                  }).subscribe(new Consumer<Boolean>() {
              @Override
              public void accept(Boolean s) throws Exception {
                  /*
                   * 步骤6：返回结果 & 设置按钮可点击样式
                   **/
                  Log.e(TAG, "提交按钮是否可点击： "+s);
                  list.setEnabled(s);
              }
          });
  ```

  

#### 6.6reduce

- 本质都是前2个数据聚合，然后与后1个数据继续进行聚合，依次类推

#### 6.7collect

- 将被观察者`Observable`发送的数据事件收集到一个数据结构里

#### 6.8startWith / startWithArray

- 发送事件前，追加发送, **后调用先追加**

#### 6.9count

- 统计个数

```java
//返回结果 = Long类型
        Observable.just(1, 2, 3, 4)
                  .count()
                  .subscribe(new Consumer<Long>() {
                    @Override
                    public void accept(Long aLong) throws Exception {
                        Log.e(TAG, "发送的事件数量 =  "+aLong);
                    }
                });
```

### 7.retryWhen 失败重试



### 8.Disposable

```java
public interface Disposable {
        void dispose();
        boolean isDisposed();
}
```

### 9.冷流-热流

```
在 Kotlin 中，可以使用以下几种方法来区分冷流和热流：

创建方式：使用 flowOf 函数、asFlow 函数、channelFlow 函数等创建的流通常是冷流，而使用 MutableStateFlow、SharedFlow 等创建的流通常是热流。

发射方式：冷流只有在有订阅者时才会开始发射元素，而热流会在创建后立即开始发射元素，即使没有任何订阅者。

订阅方式：对于冷流，每个订阅者将获得相同的元素序列，因为流会从头开始发射所有元素。而对于热流，每个订阅者可能会获得不同的元素序列，因为流可能已经发射了一些元素，而新的订阅者只能接收到后续的元素。

生命周期：热流通常需要特别小心以避免内存泄漏和其他问题，因为流会一直发射元素。如果您不再需要订阅者接收热流的元素，请确保及时取消订阅。而冷流通常不需要额外的处理，因为它只在有订阅者时才会开始发射元素。

需要注意的是，某些类型的流可能同时具有冷流和热流的特性，具体取决于使用方式和实际场景。例如，Flow 类型的流既可以使用 flowOf 函数创建一个冷流，也可以使用 channelFlow 函数创建一个热流。因此，在使用流时，需要根据具体情况来选择适合的类型和创建方式。
```

### 10.被压

发生条件：

- 异步订阅
- 生产的速度大于消费的速度

使用Observable,会导致上游数据堆积导致OOM = FLOWABLE + BUFFER
可以使用Flowable+合适的背压策略来处理背压问题

**背压策略**
MISSING ->MissingBackpressureException，每一个都必须消费完成
ERROR 数据堆积到128时抛出onError异常（推荐）
LATEST
DROP与LATEST相反
Buffer-缓存数据，不断扩容队列，直至OOM（默认的策略）

