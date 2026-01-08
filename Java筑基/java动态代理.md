[TOC]

## java反射，注解，动态代理

### 1.泛型（类型参数化）

#### 为什么使用泛型？

- 扩展性（求和 int,float,double），代码复用
- 类型安全（list） 编译期检查，让通用代码不再不安全

#### 泛型可以修饰 类 接口 方法

```java
class Test<K，T extends Comparable>{
	private K k;
    //并不是带有泛型的方法就是泛型方法
	//泛型方法的泛型 位于 返回值 与 权限修饰符之间
    //该方法泛型T与类泛型T无关
	public <T> T getter(T any){
		return any;
	}
	//不是泛型方法
	public K getK(){
        returan k;
    }
}
```

#### 泛型约束与局限性

- 类的泛型  不能 在静态中使用，泛型确定在运行时
- 不允许是基本类型
- 不允许类型判断  instanceof 

#### 泛型通配符   ?extends   |    ? super 

作用：保证类型安全的前提下，允许父子类型在泛型中协作

会有读写限制pecs

```java
//pecs 
//producer extends(out)- 上界，只安全读 kotlin中的List
//consumer super(in)   - 下界，只安全写
public class GenericType<T>{
	public T data;
} 

GenericType<Fruit>与GenericType<Apple>无关
public void test(GenericType<? extends Fruit> type){
}

```





### 2.注解

元注解：注解上的注解 @Target(ElementType.ANNOTATION_TYPE)

```java
@Target(ElementType.TYPE)  // 作用类型 
@Retention(RetentionPolicy.SOURCE)
public @interface Lance {
    String value() default "default";
}

//保留策略
public enum RetentionPolicy {
   /**
    * Annotations are to be discarded by the compiler.
	* 场景： 编译阶段
     *     APT -  AnnotationProcessorTool
	 *	   编译时语法检查，@DrawableRes，@IntDef
      */
    SOURCE,
 /**
    * Annotations are to be recorded in the class file by the compiler
	* but need not be retained by the VM at run time.  This is the default
     * behavior.
	 *  场景：字节码层级 
	  *  修改class文件,字节码插桩 ASM，统计方法耗时
       */
    CLASS,

    /**
     * Annotations are to be recorded in the class file by the compiler and
     * retained by the VM at run time, so they may be read reflectively.
     *	场景：运行时反射
     *  ButterKnife
     */
    RUNTIME
}

```

### 3.注解 加 反射 简单使用

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface AutoWired {
    String key();
}

public class WiredUtil {
    public static void inject(Activity activity) {
        Class<? extends Activity> clas = activity.getClass();
        //获得此类所有成员
        Field[] declaredFields = clas.getDeclaredFields();

        Intent intent = activity.getIntent();
        Bundle extras = intent.getExtras();
        if (extras == null){
            return;
        }

        for (Field field : declaredFields) {
            //筛选注解有InjectView的Field
            if (field.isAnnotationPresent(AutoWired.class)) {
                //获取注解信息
                AutoWired annotation = field.getAnnotation(AutoWired.class);
                String key;
                if (annotation!=null){
                    if (TextUtils.isEmpty(annotation.key())){
                        key = annotation.key();
                    }else {
                        key = field.getName();
                    }
                    if (!extras.containsKey(key)){
                        break;
                    }
                    //反射赋值
                    field.setAccessible(true);
                    try {
                        field.set(activity,  extras.get(key));
                    } catch (Exception e) {
                        e.printStackTrace();
                    }s
                }
            }
        }

    }
}

```



### 4.Retrofit反射，注解，动态代理

> 动态代理：“在不修改原始代码的情况下，**对接口的方法调用进行一层“包装”或“修饰”**。”

https://blog.csdn.net/botai2120/article/details/100951283?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-17.no_search_link&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7Edefault-17.no_search_link

