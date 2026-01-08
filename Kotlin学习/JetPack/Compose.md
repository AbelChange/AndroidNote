View overlay 写角标

compose drawWithContent


[toc]

### JetBrains 推动的方向

> 使用 KMP 复用业务逻辑 + 使用 CMP 构建跨平台 UI，统一语言、统一工具链。

### 为什么需要 State Hoisting？**

```kotlin
@Composable
fun Parent() {
    var name by remember { mutableStateOf("Tom") }

    Child(
        name = name, // 传值
        onNameChange = { name = it } // 传递修改逻辑
    )
}

@Composable
fun Child(
    name: String,
    onNameChange: (String) -> Unit
) {
    TextField(value = name, onValueChange = onNameChange)
}
```



1. **更好地管理状态的来源**（单一来源）

1. **提升组件的可复用性**（子组件无状态，逻辑可复用）
2. **更清晰的 UI 与状态分离**



Single Source Of Truth 唯一数据源

Unidirectional Data Flow 单向数据流

Recompose Scope 



### Jetpack Compose 的渲染分成三大步骤

1. **Recomposition**：执行 Composable 函数 → 生成 UI Tree（描述）

   **重组**是当入参发生变化时再次调用可组合函数的过程

2. **Layout（测量+布局）**：确定每个组件的大小和位置

3. **Draw（绘制）**：渲染到屏幕

   



### mutableStateOf

> mutableStateOf 创建一个 **可观察的状态值**，当这个值发生变化时，会自动触发使用它的 Composable 重新执行（Recomposition）

```
SnapshotMutableStateImpl 在getter/setter中有钩子，进行了注册与通知
```



> by只是委托语法糖， 修改了State中value的访问方式，使可以方便的操作值



> remember 是用来在 **Composable 函数中保存状态或计算结果** 的关键工具
>
> **在重组（recomposition）过程中保存值**，避免每次重组都重新计算或初始化；相对于不添加的remember
>
> 通常和 mutableStateOf 搭配使用，来创建可观察的状态值。




>**remember 初始化的值依赖于外部状态（传入参数），就应该把这些参数写进 remember(...) 的参数列表中**

```kotlin
@Composable
fun Parent() {
    var name by remember { mutableStateOf("Tom") }
    //不可变列表（List），names = names + "NewName"，推荐
    var names by remember { mutableStateOf(listOf("Tom", "Jerry")) }

     Column {
        // 修改单个 name
        TextField(
            value = name,
            onValueChange = { name = it },
            label = { Text("Enter Name") }
        )

        // 添加一个新名字
        Button(onClick = {
            names = names + name // 添加当前 name 到列表中
        }) {
            Text("Add to List")
        }

        // 传递 name 和 names 到子组件
        Child(name = name, names = names)
    }
}

@Composable
fun Child(name: String, names: List<String>) {
    // 单个字符串映射：大写
    val displayName by remember(name) {
        derivedStateOf { name.uppercase() }
    }

    // 列表映射：大写
    val displayNames by remember(names) {
        derivedStateOf { names.map { it.uppercase() } }
    }

    Column {
        Text("Single Name: $displayName")

        Spacer(modifier = Modifier.height(8.dp))

        Text("Name List:")
        displayNames.forEach {
            Text(text = it)
        }
    }
}

// ✅ derivedStateOf 做组合监听
@Composable
fun ChildDerivedCombined(name: String, age: Int) {
    val displayInfo by remember(name, age) {
        derivedStateOf {
            "${name.uppercase()} is $age years old"
        }
    }

    Text(text = displayInfo)
}

```






| **类型**                | **创建方式**                                      | **是否需** by **解包** | **是否自动触发 recomposition** | **访问方式举例**   |
| ----------------------- | ------------------------------------------------- | ---------------------- | ------------------------------ | ------------------ |
| mutableStateOf+by       | var count by remember { mutableStateOf(0) }       | ✅ 是                   | ✅ 是                           | count++            |
| mutableStateOf (未解包) | val count = remember { mutableStateOf(0) }        | ❌ 否                   | ✅ 是                           | count.value++      |
| mutableStateListOf      | val items = remember { mutableStateListOf(...) }  | ❌ 否                   | ✅ 是                           | items.add("A")     |
| mutableStateMapOf       | val map = remember { mutableStateMapOf(...) }     | ❌ 否                   | ✅ 是                           | map["key"] = value |
| derivedStateOf          | val derived = remember { derivedStateOf { ... } } | ❌ 否                   | ✅ 是（自动跟随依赖）           |                    |

前两种写法完全等价

### Recompose Scope

状态被使用的scope

```kotlin
@Composable
fun Parent() {
  	//当 count 发生变化时，Compose 重新执行 Parent() 整个函数
  	//before	与 after都会打印
    val count by remember { mutableStateOf(0) }
		
    Log.d("Recompose", "A recomposed before")

    A()               // 不依赖 count，不会重组
    B(count)          // 依赖 count，重组
    C()               // 不依赖 count，不会重组

    Log.d("Recompose", "C recomposed after")
}
```



**拆分 Recompose Scope** 是 Jetpack Compose 中优化重组性能的一个重要技巧。尽量让每个 Composable 函数只关注其自己所需的部分，并通过 **拆分作用域** 让它们在状态变化时独立重组，这样能避免不必要的重组，提高应用的性能。

### Side-effects(需要的事件处理，非状态)

composables [should ideally be side-effect free](https://developer.android.google.cn/develop/ui/compose/mental-model).
However, sometimes side-effects are necessary, for example, to trigger a one-off event such as showing a snackbar

| SideEffect       | 通知外部系统（非挂起副作用） | **每次成功完成组合后** 执行一次                              | 用于将 Compose 内部状态同步给外部系统，如日志记录、埋点等    |
| ---------------- | ---------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| LaunchedEffect   | 启动异步副作用               | **首次组合** 或 **key 改变时执行**，key 变化时自动取消并重启 | 启动协程，执行一次性副作用任务，如加载数据、动画、事件流收集等 |
| DisposableEffect | 管理副作用的生命周期         | **首次组合或 key 改变时调用初始化逻辑**，**离开组合或 key 变更时清理** | 创建和清理资源（注册/注销监听器、启动/停止观察等），生命周期更细粒度控制 |

- derivedStateOf

  use a remembered derived state to  minimize unnecessary compositions

​	对Composable参数进行map+distinctUntilChanged处理，避免不必要的重组，提高性能

​	适合组合多个状态

​	适合Lazy计算

​		

### CompositionLocal

| **类型**            | **能否感知变化**                  | **是否能触发重组**    | **是否可以记住值**              | **声明方式**                                  | **Get 方式**               | **Set 方式**                       |
| ------------------- | --------------------------------- | --------------------- | ------------------------------- | --------------------------------------------- | -------------------------- | ---------------------------------- |
| mutableStateOf(...) | ✅（自身变更可追踪）               | ✅（直接重组依赖组件） | ✅（需配合 remember）            | var state by remember { mutableStateOf(...) } | 直接使用变量，如 state     | state = newValue                   |
| CompositionLocal    | ✅（依赖它的Composable可感知变化） | ✅（依赖项间接重组）   | ❌（每次访问是当前值，不会记住） | val LocalFoo = compositionLocalOf<Type>()     | val foo = LocalFoo.current | CompositionLocalProvider(...) 包裹 |

- ✅ 它能触发重组 → **“像状态”**
- ❌ 它不能直接修改 → **“不像状态”**
- ❌ 它不记值 → **“不像 remember”**
- ✅ 它影响 UI → **“具备状态表现”**

### Anim

AnimationSpec (变化曲线)--Interpolator

### Transition.  in/out状态的切换，转场

单个组件->AnimationVisibility-支持fade,translate, crop,scale

多个组件->简化版的AnimationVisibility，CrossFade 交叉渐变，**淡入淡出**，只有alpha,对始终两种visibility进行切换

AnimatedContent 上两者特性合并

- AnimatedContent 的transitionSpec 参数，里面的 + 和with怎么理解
  - +：**动画组合执行**，两种效果叠加
  - with:用于组合**进入动画（EnterTransition）** 和 **退出动画（ExitTransition）**



### Modifiler

Size : width/height/padding

background

Clickable{}

Clip()

作为参数传递 默认参数 伴生的object Modifier

 集合中有fold概念，可以在某个初始值上聚合元素，也有foldRight可以修改聚合方向，对比Compose中的Modifier foldIn/out

Modifier.composed 在 Jetpack Compose 中的主要用途之一，**就是为 Modifier 提供一种支持“独立状态”的机制**,可以延迟创建，大多用于自定义modifier，可以让你在每次使用时生成各自独立的状态实例。
这和普通的 Modifier 不同，普通 Modifier 是共享状态的

Modifier.layout（measurable,constraints） 修改尺寸和位置

Modifier.measure()
Clickable-CombinedClickable-pointerIntput(Unit)

生效顺序：后发先至   数组 + 链表  存储Node，后来的放表头
```kotlin
Modifier.pointInput(Unit){
	detectTapGesture{onTap = {}.onDoubleTap = {},onLongPoress = {}, onPress = {}}
}
```



