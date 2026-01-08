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


## ViewModel 的 onCleared 调用时机

ViewModel 有一个onCleared方法，那么他是在何时调用的呢?

```kotlin
public abstract class ViewModel {
    /**
     * This method will be called when this ViewModel is no longer used and will be destroyed.
     * It is useful when ViewModel observes some data and you need to clear this subscription to prevent a leak of this ViewModel.
     */
    @SuppressWarnings("WeakerAccess")
    protected void onCleared() {
    }
}
```



```java
public class  ComponentActivity 

getLifecycle().addObserver(new LifecycleEventObserver() {
            @Override
            public void onStateChanged(@NonNull LifecycleOwner source,
                    @NonNull Lifecycle.Event event) {
                if (event == Lifecycle.Event.ON_DESTROY) 
										//配置变更引起的activity重建 不销毁viewModel 即handleRelaunchActivity会变更此boolean
                    if (!isChangingConfigurations()) {
                        getViewModelStore().clear();
                    }
                    mReportFullyDrawnExecutor.activityDestroyed();
                }
            }
        });


    /**
     * @return If the activity is being torn down in order to be recreated with a new configuration,
     * returns true; else returns false.
     */
    public boolean isChangingConfigurations() {
        return mChangingConfigurations;
    }


```



//老版本viewmodel通过这样去判断是否销毁，新的api已经不是这样，但glide是

## Fragment 的 setRetainInstance 方法

Fragment具有属性retainInstance，默认值为false，当设备旋转时，fragment会随托管activity一起销毁并重建。

调用setRetainInstance(true)方法可保留fragment，已保留的fragment不会随着activity一起被销毁，相反，它会一直保留(进程不消亡的前提下)，并在需要时原封不动地传递给新的Activity。

**保留Fragment的原理**

当设备配置发生变化时，FragmentManager首先销毁队列中fragment的视图（因为可能有更合适的匹配资源），紧接着，FragmentManager将检查每个fragment的retainInstance属性值。

- 如果retainInstance属性值为false，FragmentManager会立即销毁该fragment实例，随后，为适应新的设备配置，新的Activity的新的FragmentManager会创建一个新的fragment及其视图。
- 如果retainInstance属性值为true，则该fragment本身不会被销毁。为适应新的设备配置，当新的Activity创建后，新的FragmentManager会找到被保留的fragment，并重新创建其视图(而不会重建fragment本身)。

虽然保留的fragment没有被销毁，但它已脱离消亡中的activity并处于保留状态。尽管此时的fragment仍然存在，但已经没有任何activity托管它。

需要说明的是，只有调用了fragment的setRetainInstance(true)方法，并且因设备配置改变，托管Activity正在被销毁的条件下，fragment才会短暂的处于保留状态。如果activity是因操作系统需要回收内存而被销毁，则所有的fragment也会随之销毁。

setRetainInstance(boolean) 是Fragment中的一个方法。将这个方法设置为true就可以使当前Fragment在Activity重建时存活下来。在setRetainInstance(boolean)为true的 Fragment 中放一个专门用于存储ViewModel的Map, 自然Map中所有的ViewModel都会幸免于Activity重建，让Activity, Fragment都绑定一个这样的Fragment, 将ViewModel存放到这个 Fragment 的 Map 中, ViewModel 组件就这样实现了。

### 进程被杀恢复

SavedState

```kotlin
class MyViewModel(private val state: SavedStateHandle) : ViewModel() {
  val name: MutableLiveData<String> = state.getLiveData("name")
}
```

