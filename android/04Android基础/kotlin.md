## Kotlin

[TOC]

### 一、简介



### 二、特性

#### 1. 协程Coroutines

​	本质上，协程是轻量级的线程。 它们在某些 [CoroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html) 上下文中与 [launch](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html) *协程构建器* 一起启动。 这里我们在 [GlobalScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html) 中启动了一个新的协程，这意味着新协程的生命周期只受整个应用程序的生命周期限制。

可以将 `GlobalScope.launch { …… }` 替换为 `thread { …… }`，并将 `delay(……)` 替换为 `Thread.sleep(……)` 达到同样目的。 试试看（不要忘记导入 `kotlin.concurrent.thread`）

```kotlin
import kotlinx.coroutines.*

fun main() {
   // 非阻塞
    GlobalScope.launch { // 在后台启动一个新的协程并继续
        delay(1000L) // 非阻塞的等待 1 秒钟（默认时间单位是毫秒）
        println("World!") // 在延迟后打印输出
    }
  	// 阻塞 调用了 runBlocking 的主线程会一直 阻塞 直到 runBlocking 内部的协程执行完毕。
  	runBlocking {     // 但是这个表达式阻塞了主线程
        delay(2000L)  // ……我们延迟 2 秒来保证 JVM 的存活
    } 
  	// 也是阻塞
    println("Hello,") // 协程已在等待时主线程还在继续
    Thread.sleep(2000L) // 阻塞主线程 2 秒钟来保证 JVM 存活
}
```





### 三、使用场景

#### 1. 网络请求像同步

```java
 CoroutineScope(Dispatchers.Main).launch {
            val result = async(Dispatchers.IO) { //或者withContext(Dispatchers.IO) {
                  //使用okhttp使用同步请求，完事将response返回
                  val request = Request.Builder().url("http://www.baidu.com").build()
                  val response= OkHttpClient().newCall(request).execute()
                  response
            }
            //等待异步执行的结果
            val response = result.await()
            //返回的结果，直接显示在sample_text这个textview上，也就是更新UI
            sample_text.text = response.body()?.string()
            Log.e("----", "CoroutineScope : ${Thread.currentThread().name}")

```

#### 2. 线程切换

```java
// 2.
override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_thread)
        loadData()
    }

    private fun loadData() {
        GlobalScope.launch(Dispatchers.IO) { //在IO线程开始
            //IO 线程里拉取数据
            val result = fetchData()
            //主线程里更新 UI
            withContext(Dispatchers.Main) { //执行结束后，自动切换到UI线程
                tvShowContent.text = result
            }
        }
    }
    
    //关键词 suspend
    private suspend fun fetchData(): String {
        delay(2000) // delaying for 2 seconds to keep JVM alive
        return "content"
    }

// 2
val request = Request.Builder().url("http://www.baidu.com").build()
        val call = OkHttpClient().newCall(request)
        call.enqueue(object : Callback{
            override fun onResponse(call: Call, response: Response) {
                response.body()?.string()?.let { threadToMain(it) }
            }
 
            override fun onFailure(call: Call, e: IOException) {
 
            }
 
})
 
  fun threadToMain(string: String){
        MainScope().launch {
            sample_text.text = string
        }
    } 
```









- 参考

  [kotlin学习网站](https://www.kotlincn.net/docs/reference/coroutines-overview.html)

  [协程理解](https://segmentfault.com/a/1190000021068283)

