[toc]

### 加密

 MD5 算法是一种哈希算法,一般无法解密，所以不叫加密
 base64 可以进行直接还原，所以不叫加密，可用来传输图片，节省一次请求

### java8大基本数据类型比较

|      数据类型       |  字节  | 默认值 | 取值范围                 |
| :-----------------: | :----: | :----: | ------------------------ |
|    byte/boolean     | 1 Byte |   0    | -2^7 - 2^7-1/ (-128-127) |
| short/char(unicode) | 2Byte  |   0    | -2^15 - 2^15-1           |
|      int/float      | 4 Byte |   0    | -2^31 - 2^31-1           |
|     long/double     | 8 Byte |   0    | -2^63 - 2^63-1           |

- 1byte = 8 bit
- 位数：位数中，首位代替正负，其余的代表大小
- 包装类中，前六个中继承 Number，均实现了Comparable，可以直接Collection.sort()
- Interger面像对象，可以为null,而int只能为0，提供了一些静态API
- 与集合类合作使用时只能使用包装类型
- 表达式 5/2 = 2   int没有存储小数位置，因此取整
- switch 只适用于：byte +char + short + int(long除外的整型) ， enum/String
- 0.11 - 0.1 == 0.01 为false 二进制有误差

### LinkedList与ArrayList比较

|        | ArrayList                                                    | LinkedList                                                   |
| :----- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| get    | O(1)                                                         | O(n)                                                         |
| add    | 新增：正常O(1),扩容O(n)<br />随机插入：涉及元素移动O(n) <br /> | 新增：add<br />插入：addLast/addFirst  O(1)     <br />            其他位置涉及遍历  O(n） |
| remove | O(1)-O(n) 扩容导致复杂度增加                                 | removeFirst/RemoveLast   O(1)   ---remove(index)   O(n)      |

综上，在 `ArrayList` 的开头频繁插入元素，可能会导致大量的元素移动，从而导致性能下降
linkedList 提供了 first与last的便捷操作，其他位置都面对遍历查找带来的复杂度

### HashMap 的实现原理 

hashmap ：数组+链表

```java
//初始 table数组capacity 为 1 << 4 = 16 = table.lenth
//负载因子loadFactor = 0.75  
//threshold = capacity * loadFactor = 12

//put过程    
Node<K,V>[] table;//数据结构 :数组 + 链表/树

if(table == null || table.length == 0){
   table = resize()//第一次put时会初始化table.length = 16,2倍扩容，最大化避免出现hash碰撞使结构复杂
}
    //根据key计算元素在哈希表中的索引
    int i = (n - 1) & hash（key）
    Node<K,V> p = table[i];
    if(p == null){
       //没有头部node，直接放入
       table[i] = newNode()
    }else{
				   //覆盖队头 value
        if(hash && (== || equal){

           //遍历插入/覆盖
        }else if (是树){

      		//遍历插入/覆盖
     			//如果深度 > 8 且 size> 64转化为树 
        }else{
           
        }	
    }

    if(size > threshold){
        resize()
    }
}
   
```

如果两个键对象通过`equals()`方法相等，但它们的`hashCode()`方法返回的哈希码不同，那么它们会被错误地认为是不同的键，导致无法正确地进行查找和删除操作

不稳定性：
当你将对象作为 HashMap **键**时，**确保该对象在存储后不会更改**，尤其是那些影响 hashCode 和 equals 的字段。如果需要修改 User，最好避免将它作为 HashMap 的键，或者让它成为不可变对象（使用 val 字段）。
java: 参与hash/equal的属性使用final 修饰



### 并发修改异常ConcurrentModificationException

#### 1.单线程环境下抛出异常

- Iterator在遍历时，如果**list.add/remove**  会抛出该异常

- foreach的底层是Iterator

- 建议Iterator.add/remove(),普通for循环 会 漏删（当两个元素相邻时）

  ```java
  ArrayList<String> list = new ArrayList<String>(Arrays.asList("a","b","c","d"));
  Iterator<String> iterator = list.iterator();
  while(iter.hasNext()){
          String s = iter.next();
          if(s.equals("a")){
              iterator.remove();
  		//如果需要add set可以使用listiterator
      }
  }
  ```

  本质： hasnext方法会检测modCount(list的size修改次数)与iterator的修改次数是否相等

#### 2.多线程

| **类型**     | **实现类**                   | **线程安全** | **同步机制**        | **性能** | **适用场景**                     | **备注**                       |
| ------------ | ---------------------------- | ------------ | ------------------- | -------- | -------------------------------- | ------------------------------ |
| **List**     |                              |              |                     |          |                                  |                                |
| 非线程安全   | ArrayList                    | 否           | 无                  | 高       | 单线程或自行控制同步场景         | 使用最广泛                     |
| 线程安全旧版 | Vector                       | 是           | 方法级 synchronized | 低       | 多线程环境，但一般不推荐新用     | 兼容旧代码，推荐用更现代替代品 |
| 线程安全包装 | Collections.synchronizedList | 是           | 包装方法同步        | 低       | 简单线程安全需求，遍历需手动同步 | 需要显式 synchronized 遍历     |
| 读多写少场景 | CopyOnWriteArrayList         | 是           | 写时复制，读无锁    | 高       |                                  |                                |

| **类型**     | **实现类**                  | **线程安全** | **同步机制**                                | **性能** | **适用场景**                     | **备注**                                  |
| ------------ | --------------------------- | ------------ | ------------------------------------------- | -------- | -------------------------------- | ----------------------------------------- |
| **Map**      |                             |              |                                             |          |                                  |                                           |
| 非线程安全   | HashMap                     | 否           | 无                                          | 高       | 单线程或自行同步场景             | 使用最广泛                                |
| 线程安全旧版 | Hashtable                   | 是           | 方法级 synchronized                         | 低       | 多线程环境，但不推荐新用         | 兼容旧代码，建议用 ConcurrentHashMap 替代 |
| 线程安全包装 | Collections.synchronizedMap | 是           | 包装方法同步                                | 低       | 简单线程安全需求，遍历需手动同步 | 需要显式 synchronized 遍历                |
| 高性能并发   | ConcurrentHashMap           | 是           | 分段锁（Java8+用 CAS 和 synchronized 结合） | 高       | 高并发读写场景                   | 推荐多线程并发环境使用                    |



### String传参

- String的内容不能被动态地修改，因为底层是final字符数组实现的，数组的大小是在初始化时决定的；
- StringBuilder可以修改，底层是可变数组
- string与基本类型按值传递，不会影响变量本身



### Socket编程

心跳包实现原理

1.客户端发心跳包时启动一个定时任务
2.收到服务器心跳包返回时，移除上一个定时任务，并重新定时
3.如果定时任务被执行，意味着超时



### 多线程

 [线程相关.md](线程相关.md) 