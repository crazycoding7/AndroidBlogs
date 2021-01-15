## Glide

[TOC]

#### 1. 介绍

​	Glide是一个高效的Android图片加载库，注重平滑的滚动。

- 优点
  1. 多种图片格式的缓存，适合更多的展现方式(gif、video、缩略图)；
  2. 生命周期集成;
  3. 高效处理Bitmap(bitmap的复用和主动回收，减轻系统压力)
  4. 高效的缓存策略，缓存的是多种规格，加载速度快且内存开销少;
- 框架涉及知识点
  1. 异步加载：线程池；
  2. 切换线程：Handler；
  3. 缓存策略：LruCache、DiskLurCache
  4. 防止OOM：软引用、LruCache、图片压缩、Bitmap像素存储；
  5. 内存泄漏：生命周期集成；
  6. 列表滑动加载问题：加载错乱、队列满任务多问题。

#### 2. 原理

- 异步加载

  主要用到三个线程池，然后通过handler进行UI更新

  。

  ```java
  public final class GlideBuilder {
    private GlideExecutor sourceExecutor;   //加载源文件的线程池，包括网络加载
    private GlideExecutor diskCacheExecutor; //加载硬盘缓存的线程池
    private GlideExecutor animationExecutor; //动画线程池
  }
  // 通过handler通知UI更新
  private val MAIN_THREAD_EXECUTOR = object : Executor {
          private val handler = Handler(Looper.getMainLooper())
          override fun execute(command: Runnable) {
              handler.post(command)
          }
  }
  ```

  对象池技术request、弱引用 WeakHashMap<Request, Boolean>、内置线程池也支持okhttp等。

- 缓存

  图片三级缓存：内存缓存、硬盘缓存、网络。

  1. 内存缓存

     利用LRUCache算法。利用LinkedHashmap实现有序(hashmap+双向链表)。

     LruCache小结：

     - **LinkHashMap 继承HashMap，在 HashMap的基础上，新增了双向链表结构，每次访问数据的时候，会更新被访问的数据的链表指针，具体就是先在链表中删除该节点，然后添加到链表头header之前，这样就保证了链表头header节点之前的数据都是最近访问的（从链表中删除并不是真的删除数据，只是移动链表指针，数据本身在map中的位置是不变的）。**
     - **LruCache 内部用LinkHashMap存取数据，在双向链表保证数据新旧顺序的前提下，设置一个最大内存，往里面put数据的时候，当数据达到最大内存的时候，将最老的数据移除掉，保证内存不超过设定的最大值。**

  2. 硬盘缓存类似

- 防止OOM

  1. LruCache缓存大小设置，可以有效防止OOM；

  2. LruCache里存的是软引用对象，那么当内存不足的时候，Bitmap会被回收，也就是说通过SoftReference修饰的Bitmap就不会导致OOM；

     ```java
      private static LruCache<String, SoftReference<Bitmap>> mLruCache = new LruCache<String, SoftReference<Bitmap>>(10 * 1024){
             @Override
             protected int sizeOf(String key, SoftReference<Bitmap> value) {
                 //默认返回1，这里应该返回Bitmap占用的内存大小，单位：K
                 //Bitmap被回收了，大小是0
                 if (value.get() == null){
                     return 0;
                 }
                 return value.get().getByteCount() /1024;
             }
         };
     ```

  3. onLowMemory

     ```java
     //Glide
     public void onLowMemory() {
         clearMemory();
     }
     
     public void clearMemory() {
         // Engine asserts this anyway when removing resources, fail faster and consistently
         Util.assertMainThread();
         // memory cache needs to be cleared before bitmap pool to clear re-pooled Bitmaps too. See #687.
         memoryCache.clearMemory();
         bitmapPool.clearMemory();
         arrayPool.clearMemory();
       }
     ```

  4. Bitmap像素存储考虑

      **Bitmap的像素数据大小 = 宽 \* 高 \* 1像素占用的内存。**

     ```java
     	// 1像素占用字节数
       public static final int ALPHA_8_BYTES_PER_PIXEL = 1;
       public static final int ARGB_4444_BYTES_PER_PIXEL = 2;
       public static final int ARGB_8888_BYTES_PER_PIXEL = 4;
       public static final int RGB_565_BYTES_PER_PIXEL = 2;  // glide
       public static final int RGBA_F16_BYTES_PER_PIXEL = 8;
     ```

     如果Bitmap使用 `RGB_565` 格式，则1像素占用 2 byte，`ARGB_8888` 格式则占4 byte。
      **在选择图片加载框架的时候，可以将内存占用这一方面考虑进去，更少的内存占用意味着发生OOM的概率越低。** Glide内存开销是Picasso的一半，就是因为默认Bitmap格式不同。

     **那是否可以让像素数据不放在java堆中，而是放在native堆中呢**？据说Android 3.0到8.0 之间Bitmap像素数据存在Java堆，而8.0之后像素数据存到native堆中。

- 内存泄漏

  当然，修改也比较简单粗暴，**将ImageView用WeakReference修饰**就完事了。

  事实上，这种方式虽然解决了内存泄露问题，但是并不完美，例如在界面退出的时候，我们除了希望ImageView被回收，同时希望加载图片的任务可以取消，队未执行的任务可以移除。

  Glide的做法是监听生命周期回调，看 `RequestManager` 这个类,在Activity/fragment 销毁的时候，取消图片加载任务，细节大家可以自己去看源。

  ```java
  public void onDestroy() {
      targetTracker.onDestroy();
      for (Target<?> target : targetTracker.getAll()) {
        //清理任务
        clear(target);
      }
      targetTracker.clear();
      requestTracker.clearRequests();
      lifecycle.removeListener(this);
      lifecycle.removeListener(connectivityMonitor);
      mainHandler.removeCallbacks(addSelfToLifecycle);
      glide.unregisterRequestManager(this);
    }
  ```

  

- 其他

###### 图片错乱

由于RecyclerView或者LIstView的复用机制，网络加载图片开始的时候ImageView是第一个item的，加载成功之后ImageView由于复用可能跑到第10个item去了，在第10个item显示第一个item的图片肯定是错的。

常规的做法是给ImageView设置tag，tag一般是图片地址，更新ImageView之前判断tag是否跟url一致。

当然，可以在item从列表消失的时候，取消对应的图片加载任务。要考虑放在图片加载框架做还是放在UI做比较合适。

###### 线程池任务过多

列表滑动，会有很多图片请求，如果是第一次进入，没有缓存，那么队列会有很多任务在等待。所以在请求网络图片之前，需要判断队列中是否已经存在该任务，存在则不加到队列去。

#### 3. 如何加载大图和长图

图片压缩(利用BitmapFactory.Options inSampleSize)；

局部显示(BitmapRegionDecoder);

`Android`里面是利用`BitmapRegionDecoder`来局部展示图片的，展示的是一块矩形区域。为了完成这个功能那么就需要一个方法设置图片，另一个方法设置展示的区域。



[参考1](https://juejin.im/post/5dbeda27e51d452a161e00c8)

[参考2](https://muyangmin.github.io/glide-docs-cn/int/okhttp3.html)

