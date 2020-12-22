## 内存优化

[TOC]

### 一、概述

##### 1. 手机内存和PC内存区别？

​	PC内存：DDR,全名 **双倍数据速率内存**。

​	手机内存：LPDDR，全称是**“低功耗双倍数据速率内存”**，其中 LP 就是“Lower Power”低功耗的意思。

​	区别：体积小、功耗小。

​	以 LPDDR4 为例，带宽 = 时钟频率 × 内存总线位数 ÷ 8，即 1600 × 64 ÷ 8 = 12.8GB/s，因为是 DDR 内存是双倍速率，所以最后的带宽是 12.8 × 2 = 25.6GB/s。

![](images/apm_memory_phone.png)

目前市面上的手机，主流的运行内存有 LPDDR3、LPDDR4 以及 LPDDR4X。可以看出 LPDDR4 的性能要比 LPDDR3 高出一倍，而 LPDDR4X 相比 LPDDR4 工作电压更低，所以也比 LPDDR4 省电 20%～40%。当然图中的数据是标准数据，不同的生成厂商会有一些低频或者高频的版本，性能方面高频要好于低频。

#### 2. 设备内存大小与年限表

其中：2010、2013位分水岭。

| RAM   | condition | Year Class |
| ----- | --------- | ---------- |
| 768MB | 1 core    | 2009       |
|       | 2+ cores  | 2010       |
| 1GB   | <1.3GHz   | 2011       |
|       | 1.3GHz+   | 2012       |
| 1.5GB | <1.8GHz   | 2012       |
|       | 1.8GHz+   | 2013       |
| 2GB   |           | 2013       |
| 3GB   |           | 2014       |
| 5GB   |           | 2015       |
| more  |           | 2016       |

#### 3. 引发问题

1. 异常。比如：OOM、重启、low memory kill 杀进程。
2. 卡顿。比如：GC回收频繁。

**两个误区：**

误区一：内存占用越少越好；

误区二：Native 内存不用管。

### 二、测量方式

#### 1. Java内存分配

​	跟踪Java堆内存使用情况，最常用的工具有Allocation Tracker 和 MAT(Memory Analyzer Tool)，一个是AS工具自带的，一个是eclipse 插件。

Allocation Tracker 的三个缺点。

​	获取的信息过于分散，中间夹杂着不少其他的信息，很多信息不是应用申请的，可能需要进行不少查找才能定位到具体的问题。

​	跟 **Traceview(统计函数执行时间)** 一样，无法做到自动化分析，每次都需要开发者手工开始 / 结束，这对于某些问题的分析可能会造成不便，而且对于批量分析来说也比较困难。

​	虽然在 Allocation Tracking 的时候，不会对手机本身的运行造成过多的性能影响，但是在停止的时候，直到把数据 dump 出来之前，经常会把手机完全卡死，如果时间过长甚至会直接 ANR。

​	**解决方法（困难多多）：**

​	实现一个自定义的“Allocation Tracker”，实现对象内存的自动化分析。通过这个工具可以获取所有对象的申请信息（大小、类型、堆栈等），可以找到一段时间内哪些对象占用了大量的内存。

#### 2. Native内存分配

	1. 利用Allocation Tracker工具，不太友好。
 	2. 利用[Malloc 调试](https://android.googlesource.com/platform/bionic/+/master/libc/malloc_debug/README.md)、[Malloc Hooks](https://android.googlesource.com/platform/bionic/+/master/libc/malloc_hooks/README.md)。

### 三、内存优化方向探讨

#### 1. 设备分级



#### 2. 图片优化



#### 3. 内存泄漏



#### 4. 本地或线上内存监控

**参考：**

[1. Allcation Tracker帮助文档](https://developer.android.com/studio/profile/memory-profiler?hl=zh-cn#performance)