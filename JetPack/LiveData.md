LiveData

对observer进行了wrapper,

感知记录当前Livecylcle的状态，在dispatchingValue时，根据状态和版本号判断是否通知

https://www.cnblogs.com/baiqiantao/p/10621008.html




- 观察者模式
- 生命周期感知

```java
 public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
		//同一个 observer 不能 bind不同的 lifecyle 
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
        if (existing != null && !existing.isAttachedTo(owner)) {
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
		
        owner.getLifecycle().addObserver(wrapper);
    }


 //绑定当前lifecyle，感知其生命周期变化，并记录
  class LifecycleBoundObserver implements LifecycleObserver {
	
      final Observer<? super T> mObserver;
      boolean mActive;
      int mLastVersion = START_VERSION;

        LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer)

        @Override
        boolean shouldBeActive() {
            return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
        }

        @Override
        public void onStateChanged(@NonNull LifecycleOwner source,
                @NonNull Lifecycle.Event event) {
				
      				
        }

    }

 
//调用栈
setValue
    ->dispatchValue 遍历所有observer通知
    ->considerNotify() 版本号 + 当前observer状态
    ->onChanged
    

```

使用：

ViewModel类

- 不持有view ，  避免了内存泄露
- 配置更改时，不需要重新获取数据

```java

public LiveData<String> getMediatorLiveData(String url) {
//创建 中介     LiveData
MediatorLiveData<String> mediatorLiveData = new MediatorLiveData<>();
	//远程liveData
	 LiveData mutableLiveData = mutableLiveData().getRemoteLiveData();
		//关联中介与远程
	 mediatorLiveData.addSource(mutableLiveData, new Observer<BaseResp<String>>() {
            @Override
            public void onChanged(BaseResp<ReportRoomDto> resp) {
					//通知到容器
               mediatorLiveData.setValue(new InfoResult(true));
            }
        });	  	
	return mediatorLiveData;
}
```



2.容器

```java

//创建ViewModel，持有当前容器的生命周期
LiveDataViewModel mViewModel = ViewModelProviders.of(this).get(LiveDataViewModel.class);
//观察livedata
mViewModel.getMediatorLiveData("").observe(LifeCycleOwener owener,new Observer<Result>(){
    @Override
    public void onChanged(Result result){
        //ui更新
    }   
});
//3.订阅时机
```

activity --onCreate() --subcribeUI(LiveData)

fragment --onAcCreated -- subscribeToModel(Model)

3.
// `observe`通常，只有active的owener会收到变化通知

 LifecycleOwner is considered as active, if its state is `STARTED`     从onStart 到 onPause的 change都能观察到



// `observeForever`无论owener什么状态都会获得通知，直到remove

An observer added via `observeForever(Observer)` is considered as always active and thus will be always notified about modifications. For those observers, you should manually call `removeObserver(Observer)`.

4.mutableLiveData 易变的LiveData，提供两个方法改变value

- //main/ui   livedata.setValue();
- //io/cpu    livedata.postvalue();

5.MediatorLiveData 用于对多个LiveData进行合并

7.SingleLiveEvent,（onlyFromUser）只有setValue才会调用该方法，生命周期引起的变化不会激活该方法



### LiveData为啥连续postValue两次，第一次值会丢失?

- 只有pendingData值为初始值才会 postRunnable,处理结束会设置为初始值
- 如果post的 r 还未被主线程处理，即pendingData还有值时
- 再次post，只是修改pendingData，不会发送新的 r ，避免频繁切换线程带来开销

```java
    protected void postValue(T value) {
        boolean postTask;
        synchronized (mDataLock) {
            postTask = mPendingData == NOT_SET;
            mPendingData = value;
        }
        if (!postTask) {
            return;
        }
        ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
    }

```



```java
    private final Runnable mPostValueRunnable = new Runnable() {
        @SuppressWarnings("unchecked")
        @Override
        public void run() {
            Object newValue;
            synchronized (mDataLock) {
                newValue = mPendingData;
                mPendingData = NOT_SET;
            }
            setValue((T) newValue);
        }
    };
```