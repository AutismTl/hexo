---
title: Android中的线程池
date: 2017-06-08
tags: Android
---
----------
## 前言
使用线程池能给我们带来很多好处，线程池的优点可以概括为以下三点：
- 重用线程池中的线程，减少创建和销毁线程的性能开销。
- 有效控制线程池的最大并发数，避免因为大量的线程之间因为抢夺系统资源造成阻塞。
- 能对线程进行简单的管理，并提供定时执行以及指定时间间隔循环执行等。

## ThreadPoolExecutor
Android中的线程池概念来源于Java中的Executor,Executor是一个接口,真正的线程池实现为ThreadPoolExecutor,他提供一系列参数来配置线程池。其构造方法为：
<!--more-->
```java
public ThreadPoolExecutor(
    int corePoolSize,   //核心线程数
    int maximumPoolSize,   //最大线程数
    long keepAliveTime, //超时时长
    TimeUnit unit,  //指定时间单位，枚举类型
    BlockingQueue<Runnable> workQueue,  //任务队列
    ThreadFactory threadFactory//线程工厂接口
    )
```

**参数说明**
- corePoolSize :核心线程数，默认核心线程会在线程中一直存活，即使闲置。如果将ThreadPoolExecutor的allowCoreThreadTimeOut属性设置为true时，闲置的核心线程在等待新任务时会有超时策略，时间由keepAliveTime决定，超时将被终止。
- maximumPoolSize :最大线程数，活动线程数量超过它，后续任务就会被阻塞。
- keepAliveTime :非核心线程超时时长，超过便被回收。当allowCoreThreadTimeOut属性为true时，同样作用于核心线程。
- unit :keepAliveTime的单位，枚举类型，常用有 TimeUnit.MILLISECONDS(ms) TimeUnit.SECONDS(s) TimeUnit.MINUTES(min) 等等。
- workQueue :任务队列，通过线程池的execute方法提交的Runnable对象会存储在这个参数中。
- threadFactory :线程工厂，只有一个new Thread(Runnable r)方法，可为线程池创建新线程。

**ThreadPoolExecutor执行任务规则：**
1. 线程数量<核心线程数量 启动一个核心线程执行任务。
2. 线程数量>=核心线程数量，将任务插入任务队列等待执行（任务队列未满）。
3. 当任务队列满了时，立即启动一个非核心线程执行任务(线程数量未达到规定的最大线程数)。
4. 线程数量达到线程池规定最大值，拒绝执行，抛出异常。

## 线程池分类
根据不同的参数，Android中设计了四类不同功能特性的线程池，它们直接或者间接的通过设置ThreadPoolExecutor实现。它们分别为：FixedThreadPool、CachedThreadPool、ScheduledThreadPool、SingleThreadExecutor。

**1. FixedThreadPool**
创建:
```java
public static ExecutorService newFixThreadPool(int nThreads){  
    return new ThreadPoolExecutor(nThreads, nThreads, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>());  
}  
Executors.newFixThreadPool(5).execute(r);
```
说明: 从参数上可以看出，它只有核心线程，数量固定，而且不会回收(除非线程池被关闭了)，也没有超时限制。所以，当所有核心线程都处于活动状态时，新任务就处于等待状态，直到有线程空闲。这些特性使得FixedThreadPool能够快速响应外界请求。另外它的任务队列也是没有大小限制的.

**2. CachedThreadPool**
创建:
```java
public static ExecutorService newCachedThreadPool(){  
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, TimeUnit. SECONDS, new SynchronousQueue<Runnable>());  
}  
Executors.newCachedThreadPool().execute(r);
```
说明: CachedThreadPool中没有核心线程，线程最大数为Integer.MAX_VALUE(很大的数)，意味着线程数量没有限制。所以当线程池中的线程都处于活动状态时，会为新任务创建新线程，否则就利用空闲线程（60s空闲时间，过了就会被回收，所以线程池中有0个线程的可能）处理任务。任务队列SynchronousQueue(非常特殊的队列,很多情况下可以理解为无法存储元素)相当于一个空集合，任何任务都会被立即执行。这类线程池比较适合执行大量的耗时较少的任务。

**3. ScheduledThreadPool**
创建:
```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize){  
return new ScheduledThreadPoolExecutor(corePoolSize);  
}  
public ScheduledThreadPoolExecutor(int corePoolSize){  
super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS, new DelayedQueue ());  
}  
//2s后执行r任务
Executors. newScheduledThreadPool(5).scheduleAtFixedRate(r, 2000, TimeUnit.MILLISECONDS);  
//延迟10ms后，每隔2s执行一次r任务
Executors.newScheduledThreadPool(5).scheduleAtFixedRate(r, 10, 2000, TimeUnit.MILLISECONDS);
```
说明: 核心线程数固定，非核心线程（闲置将被立即回收）数没有限制。ScheduledThreadPool主要用于执行定时任务以及有固定周期的重复任务。

**4. SingeleThreadExecutor**
创建:
```java
public static ExecutorService newSingleThreadPool (){  
    return new FinalizableDelegatedExecutorService ( new ThreadPoolExecutor (1, 1, 0, TimeUnit. MILLISECONDS, new LinkedBlockingQueue<Runnable>()) );  
}  
Executors.newSingleThreadPool().execute(r);
```
说明：这种线程池只有一个核心线程，该线程池的意义在于统一所有的外界任务到同一线程中按顺序执行。因此这些任务之间不需要处理线程同步的问题。

## AsyncTask中的线程池
AsyncTask中有两个线程池(SerialExecutor和THREAD_POOL_EXECUTOR)和一个Handle(InternalHandle),其中线程池SerialExecutor用于任务的排队,HREAD_POOL_EXECUTOR用于真正执行任务,InternalHandle用于将执行环境从线程池切换到主线程。

AsyncTask中的线程池THREAD_POOL_EXECUTOR配置规格:
- 核心线程数等于CPU核心数 +1
- 线程池最大线程数为CPU核心数的2倍 +1
- 核心线程无超时机制,非核心线程闲置超时时间为1秒
- 任务队列的容量为128




