## ActivityThread源码

[TOC]

### 1. 介绍

​	它管理应用程序主线程的执行(main入口函数)，并负责调度和执行activities、service和其他操作。都是基于事件驱动机制完成。

### 2. ActivityThread

```java
public final class ActivityThread {
	 final ApplicationThread mAppThread = new ApplicationThread();
   final Looper mLooper = Looper.myLooper();
   final H mH = new H();
   final ArrayMap<IBinder, ActivityClientRecord> mActivities = new ArrayMap<>();
  
   private class ApplicationThread extends IApplicationThread.Stub {...}
   private class H extends Handler {...}
	 public static void main(String[] args) {...}
   ...
}  
```

### 3. Main函数

​	app入口函数，负责app创建工作。

```java
 public static void main(String[] args) {
        //....
        //创建Looper和MessageQueue对象，用于处理主线程的消息
        Looper.prepareMainLooper();
        //创建ActivityThread对象
        ActivityThread thread = new ActivityThread(); 
        //建立Binder通道 (创建新线程)
        thread.attach(false);
        Looper.loop(); //消息循环运行
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

### 4. ApplicationThread

​	通过Binder机制建立与AMS通信。ApplicationThread作为server，AMS作为client。

```java
private void attach(boolean system) {
	...			
  //将mAppThread放到RuntimeInit类中的静态变量,也就是ApplicationThreadNative中的 this
  RuntimeInit.setApplicationObject(mAppThread.asBinder());
  //创建ActivityManagerProxy对象
  final IActivityManager mgr = ActivityManagerNative.getDefault();
  try {
    //将mAppThread传入ActivityManagerService中
    mgr.attachApplication(mAppThread);
  } catch (RemoteException ex) {
  }
}
```

https://www.cnblogs.com/mingfeng002/p/10323668.html

### 5. class H

