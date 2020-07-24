## 100

[TOC]

### 线程

#### 1.  为什么Android中要设计成只能在UI线程更新UI呢？

​	根本原因：为了达到流畅的用户体验，在16ms内完成一帧的绘制(刷新频率60HZ)。

​	单线程访问好处：

	1. 多线程设计复杂，并发问题难处理(加锁和同时更新UI不可控)，更新性能很难达到；
 	2. 单线程架构设计简单，提高界面更新性能问题；
 	3. Android的UI控件不是**线程安全**的，如果在多线程中并发访问可能会导致UI控件处于不可预期的状态。

#### 2. 主线程和UI线程区别？

​	ActivityThread的main函数是app的入口方法，也叫主线程；

​	UI线程是View绘制和刷新所在的线程，即ViewRootImpl创建算在的线程。

​	通常情况下，我们一般认为Main Thread就是UIThread。更确切的说，UIThread是**创建View所在的线程**，否则会报错`Only the original thread that created a view hierarchy can touch its views.`

#### 3. 非UI线程真的不能刷新UI吗？

​	答：不一定，之所以子线程不能更新界面，是因为Android在线程的方法里面采用checkThread进行判断是否是主线程，而这个方法是在ViewRootImpl中的，这个类是在onResume里面才生成的，因此，如果这个时候子线程在onCreate方法里面生成更新UI，而且没有做阻塞，就是耗时多的操作，还是可以更新UI的。

#### 4. SurfaceView为什么可以在子线程绘制呢？

​	窗口中的view共享一个window，window又对应一个Surface，所以窗口中的view共享一个Surface。他的绘制操作，最终都会调用到ViewRootImpl，那么这个就会被检查是否主线程了，所以只要在ViewRootImpl启动后，访问UI的所有操作都不可以在子线程中进行。

​	而SurfaceView本身就自带一个Surface，使用双缓存机制，与主线程和Window窗口是分离的，容易实现子线程访问(一般出现在游戏或被动更新功能中)。

### Handler

#### 1. 谈谈Handler机制？

1. 作用

   跨线程通信，因为子线程不能操作UI。

2. 组成

   Message(消息)、MessageQueue(消息队列)、Handler(消息处理器)、Looper(消息池)。

   组成比例(1个线程)：Thread(1):Looper(1):MessageQueue(1):Handler(N)。

3. 流程

   ​	主线程创建的时候会创建一个Looper，同时在Looper内部创建一个消息队列。而创建Handler的时候会取出当前线程的Looper和消息队列，然后Handler在子线程通过**MessageQueue.equeueMessage**在消息队列中添加一条Message；

   ​	通过**Looper.looper()**开启的消息循环会不断调用**MessageQueue.next()**，无消息时会阻塞。取得对应的Message并且通过**Handler.dispatchMessage**传递给Handler，最终调用Handler.handlerMessage处理消息。

#### 2. Hander引起的内存泄漏如何处理？

1. 泄漏原因

   **非静态内部类默认持有外部类的引用。**

   当用户关闭activity时，Hander持有activity的引用，导致activity泄漏。

2. 解决方法

   将Handler定义为静态内部类，在内部类持有Activity的弱引用，并在onDestory中调用**handler.removeMessages**。

#### 3. Looper死循环为什么不会导致ANR?

​	ANR发生条件：消息没有及时处理、消息处理事件过长。

​	而阻塞是没有消息时发生的，不符合ANR条件。**主线程就任务就是处理各种消息，没有消息就会阻塞等待。**

#### 4. Message如何存储？

​	按时间顺序的单链表结构存储，最近时间在链表头。其他时间Message会遍历插入链表对应位置。

#### 5. 子线程如何使用Hander?

```java
 Thread(Runnable {
            Looper.prepare() // 1. 创建looper和消息队里(绑定当前线程)
            val handler = object :Handler(){  //2. handler
                override fun handleMessage(msg: Message) {
                    super.handleMessage(msg)
                }
            }
            Looper.loop() // 3. 循环处理消息
            //4. 上面阻塞了，永远不会执行到这里。  
        }).start()
```

#### 6. Message如何创建效果好？

1. 直接生成 `Message m = new Message()`;

2. 通过`Message m = Message.obtain()`;

3. 通过`Message m = mHander.obtainMessage()`，内部调用2方法;

   后两者效果更好，因为Android默认的消息池中消息数量是50(对象池，链表实现)，而后两者是直接在消息池中取出一个Message实例，这样做就可以避免多生成Message实例。

#### 7. HandlerThread干嘛用的？

​	是google提供的一个带looper的线程，方便与线程通信(UI消息的简单实现)。

```java
public class HandlerThread extends Thread {
	@Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
}
// 外界调用
Handler mThreadHandler = new Handler(handlerThread.loop); // 传入loop
```

#### 8. 消息阻塞的深层机制？

> IO事件：输入输出(input/output)的对象可以是文件(file)， 网络(socket)，进程之间的管道(pipe)。在linux系统中，都用文件描述符(fd)来表示。

​	epoll机制：epoll是Linux内核为处理大批量[文件描述符](https://baike.baidu.com/item/文件描述符/9809582)而作了改进的poll，是Linux下多路复用IO接口select/poll的增强版本，它能显著提高程序在大量[并发连接](https://baike.baidu.com/item/并发连接/3763280)中只有少量活跃的情况下的系统CPU利用率。

- `select` 和 `poll` 监听文件描述符list，进行一个线性的查找 O(n)

- `epoll`: 使用了内核文件级别的回调机制O(1)

  epoll高效的本质在于：

  减少了用户态和内核态的文件句柄拷贝

  减少了对可读可写文件句柄的遍历

  mmap 加速了内核与用户空间的信息传递，epoll是通过内核与用户mmap同一块内存，避免了无谓的内存拷贝

  IO性能不会随着监听的文件描述的数量增长而下降

  使用红黑树存储fd，以及对应的回调函数，其插入，查找，删除的性能不错，相比于hash，不必预先分配很多的空间

  #### 9. Native Looper?

  1. 目的

     Native Loope主要配合Java looper 实现线程通信机制。

  2. 实现

     ​	整个的结构很简单，JAVA Looper包含一个MessageQueue，MessageQueue对应的Native 实例是一个NativeMessageQueue实例，NativeMessageQueue在创建的时候生成一个Native Looper。  **其实Native Looper存在的意义就是作为JAVA Looper机制的开关器。**

     ```java
     public final class MessageQueue {
     	 private long mPtr; // 本地消息队列指针
     	 private native static long nativeInit();
        private native static void nativeDestroy(long ptr);
        private native void nativePollOnce(long ptr, int timeoutMillis); // 是否block 
        private native static void nativeWake(long ptr); // 唤醒本地线程
        private native static boolean nativeIsPolling(long ptr);
        private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);
     }  
     // C++
     NativeMessageQueue::NativeMessageQueue() {  
         mLooper = Looper::getForThread();  
         if (mLooper == NULL) {  
             mLooper = new Looper(false);  
             Looper::setForThread(mLooper);  
         }  
     }  
     ```

     两种情况：

     ​	当添加Message时，会唤醒本地线程，调用nativewake函数；

     ​	当消息队列为空时，调用nativePollOnce阻塞本地线程。

  3. 。。



### View

#### 1. Activity、Window、View关系？





### 架构

#### 1. 说说你对软件架构的理解？

 1. 软件架构的本质、设计三原则；

 2. 架构设计思想和方法(solid/分离关注点/依赖注入/面向切面)；

 3. android架构的发展。

    [参考](https://zhuanlan.zhihu.com/p/66535377)

### 开源

#### 1. Glide



#### 2. Retrofit

1. 如何与UI线程通信的？

   ```java
   static class Android extends Platform {
       @Override public Executor defaultCallbackExecutor() {
         return new MainThreadExecutor();
   }
   @Override CallAdapter.Factory defaultCallAdapterFactory(Executor callbackExecutor) {
     return new ExecutorCallAdapterFactory(callbackExecutor);
   } 
   static class MainThreadExecutor implements Executor {
     //!!! 通过建立绑定UI线程Looper的Hander实现。
     private final Handler handler = new Handler(Looper.getMainLooper());
     @Override public void execute(Runnable r) {
       handler.post(r);
     }
   }
   ```

2. ...





- 引用

[参考1](https://github.com/yangchong211/YCBlogs)

[参考2](https://juejin.im/post/5c81db916fb9a049d37fe6a1)

[参考3](https://github.com/gatieme/CodingInterviews)

