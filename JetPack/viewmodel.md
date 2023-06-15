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



highlighter- code-theme-dark Java

```
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

ViewModel 创建的时候提到过实例化了一个 `HolderFragment` 。并且实例化的时候通过上面 createHolderFragment 方法将其`fragmentManager.beginTransaction().add(holder, HOLDER_TAG).commitAllowingStateLoss();`，在 commit 之后，fragment 将会获得生命周期。

我们看看其 onDestroy 方法：



```java
@Override
public void onDestroy() {
    super.onDestroy();
    mViewModelStore.clear();
}
```

在HolderFragment的onDestroy()方法中调用了ViewModelStore.clear()方法，在该方法中会调用ViewModel的onCleared()方法：


```java
/**
 *  Clears internal storage and notifies ViewModels that they are no longer used.
 */
public final void clear() {
    for (ViewModel vm : mMap.values()) {
        vm.onCleared(); //调用ViewMode的onCleared方法
    }
    mMap.clear();
}
```

## Fragment 的 setRetainInstance 方法

Fragment具有属性retainInstance，默认值为false，当设备旋转时，fragment会随托管activity一起销毁并重建。

调用setRetainInstance(true)方法可保留fragment，已保留的fragment不会随着activity一起被销毁，相反，它会一直保留(进程不消亡的前提下)，并在需要时原封不动地传递给新的Activity。

**保留Fragment的原理**

当设备配置发生变化时，FragmentManager首先销毁队列中fragment的视图（因为可能有更合适的匹配资源），紧接着，FragmentManager将检查每个fragment的retainInstance属性值。

- 如果retainInstance属性值为false，FragmentManager会立即销毁该fragment实例，随后，为适应新的设备配置，新的Activity的新的FragmentManager会创建一个新的fragment及其视图。
- 如果retainInstance属性值为true，则该fragment本身不会被销毁。为适应新的设备配置，当新的Activity创建后，新的FragmentManager会找到被保留的fragment，并重新创建其视图(而不会重建fragment本身)。

虽然保留的fragment没有被销毁，但它已脱离消亡中的activity并处于保留状态。尽管此时的fragment仍然存在，但已经没有任何activity托管它。

需要说明的是，只有调用了fragment的setRetainInstance(true)方法，并且因设备配置改变，托管Activity正在被销毁的条件下，fragment才会短暂的处于保留状态。如果activity是因操作系统需要回收内存而被销毁，则所有的fragment也会随之销毁。

**那么问题来了，为什么横竖屏切换 ViewModel 不会回调 onCleared？**
看 HolderFragment 的构造方法里有个`setRetainInstance(true);`



highlighter- code-theme-dark Java

```
public HolderFragment() {
    setRetainInstance(true);
}
```

setRetainInstance(boolean) 是Fragment中的一个方法。将这个方法设置为true就可以使当前Fragment在Activity重建时存活下来。在setRetainInstance(boolean)为true的 Fragment 中放一个专门用于存储ViewModel的Map, 自然Map中所有的ViewModel都会幸免于Activity重建，让Activity, Fragment都绑定一个这样的Fragment, 将ViewModel存放到这个 Fragment 的 Map 中, ViewModel 组件就这样实现了。
