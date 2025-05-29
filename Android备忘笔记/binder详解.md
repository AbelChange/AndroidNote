### 进程间通信

https://github.com/linxu-link/CarServerArch

 aidl支持传输的数据类型

- 八种基本数据类型：byte、char、short、int、long、float、double、boolean
- String，CharSequence
- 实现了Parcelable接口的数据类型
- List 类型。List承载的数据必须是AIDL支持的类型，或者是其它声明的AIDL对象
- Map类型。Map承载的数据必须是AIDL支持的类型，或者是其它声明的AIDL对象



重要的几句话：

只能创建binder对象，不能创建binderProxy对象

第一个binder是servicemanager创建的，其他进程可以通过其拿到注册进去的Binder对应的BinderProxy

Parcel解包过程中如果发现如果在同一进程，返回binder，反之返回binderProxy

1.socket通信，应用A初始化请求Zygote孵化出新的进程
 2.匿名共享内存，因为Binder不支持图像这样子的大数据传输，所以用匿名共享内存， cursor的实现就是匿名共享内存



<img src="img/binder%E8%AF%A6%E8%A7%A3/webp-20240729194547125" alt="img"  style="zoom: 33%; " />



Bundle  ContentProvider BroadcastReceiver Aidl 

Socket  从用户空间拷贝到内核空间，再拷贝到用户空间 涉及两次拷贝 

共享内存： mmap 方式 (mmkv)  + ashmem(MemoryFile)   一次拷贝就可以 内存读写取代I/O读写，更加高效	

- mmap: 多个进程将同一个文件映射到自己的虚拟地址空间中 操作同一个文件

- ashmem :直接操作同一个块内存


Binder驱动：

- 传统概念：磁盘-->内核空间-->用户空间
  使用mmap,用户对内核空间的操作直接映射到磁盘
- binder:客户端只需要将数据copy_from_user一次即可
  - Binder不存在物理磁盘，Binder驱动使用mmap是为了在内核空间创建数据接收缓存区，限制单次传输1M

```java
public interface IRemote extends android.os.IInterface

//对于服务端来讲Binder就是本地对象Stub
override fun onBind(intent: Intent?): IBinder? {
    return object : IRemote.Stub() {
      	
    }
}


public static abstract class Stub extends Binder implements IRemote{
		//客户端调用
    public static IRemote asInterface(android.os.IBinder obj)
        {
          //参数 binder驱动提供的binder对象
          //如果本地找到，说明在同一个进程，该binder不需要代理
          android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
          if (((iin!=null)&&(iin instanceof com.ablec.myarchitecture.aidl.IRemote))) {
            return ((com.ablec.myarchitecture.aidl.IRemote)iin);
          }
          //没找到，说明是远程对象，需要代理
          return new com.ablec.myarchitecture.aidl.IRemote.Stub.Proxy(obj);
        }
		
		//对于客户端来讲binder就是BinderProxy
    private static class Proxy implements IRemote{
        private android.os.IBinder mRemote;//BinderProxy

        Proxy(android.os.IBinder remote){
          mRemote = remote;
        }

        @Override public android.os.IBinder asBinder(){
          return mRemote;
        }

        //客户端拿到BinderProxy方法后即可调用业务方法
        @Override public java.lang.String plus(int a, int b) throws android.os.RemoteException{
          Parcel _data = android.os.Parcel.obtain();
          Parcel _reply = android.os.Parcel.obtain();
          //对应BinderProxy的transact方法
          mRemote.transact(Stub.TRANSACTION_plus, parcelable, _reply, 0);
          return 	_reply.readString()
        }
			
}



```



