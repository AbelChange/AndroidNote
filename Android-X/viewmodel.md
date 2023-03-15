mvvm

### viewmodel工厂

```java
/**
 * Factory for ViewModels
 */
public class ViewModelFactory implements ViewModelProvider.Factory {

    private final UserDataSource mDataSource;

    public ViewModelFactory(UserDataSource dataSource) {
        mDataSource = dataSource;
    }

    @Override
    public <T extends ViewModel> T create(Class<T> modelClass) {
        if (modelClass.isAssignableFrom(UserViewModel.class)) {
            return (T) new UserViewModel(mDataSource);
        }
        throw new IllegalArgumentException("Unknown ViewModel class");
    }
}
```

//2.业务vm

```
public class LogicViewModel extends AndroidViewModel {
       public LogicViewModel(@NonNull Application application, String topic) {
        super(application);//通过getApplictaion（）可以拿到
    }
}



//3.容器get, viewmodel被provider的viewstore管理（hashmap存储，类似单例）
ViewModelProviders.of(this, ViewModelFactory).get(ViewModel.class)；

```

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



