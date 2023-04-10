# 线程池

线程池：管理线程执行和调度的池子。 

* 管理线程，避免增加创建线程和销毁线程的资源损耗。因为线程其实也是一个对象，创建一个对象，需要经过类加载过程，销毁一个对象，需要走GC垃圾回收流程，都是需要资源开销的。
* 提高响应速度，如果任务到达了，相对于从线程池拿线程，重新去创建一条线程执行，速度肯定慢很多。
* 重复利用，线程用完，再放回池子，可以达到重复利用的效果，节省资源。

## ThreadPoolExecutor

ThreadPoolExecutor是调度线程的执行器，一般会使用`Executors`提供的静态方法来创建线程池，但是这些方法容易出现OOM，后面会分析原因。 

先了解ThreadPoolExecutor
```
//ThreadPoolExecutor构造函数
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

* int corePoolSize：核心线程数
* int maximumPoolSize：最大线程数（核心线程数 + 非核心线程数）
* long keepAliveTime：非核心线程闲置下来最多存活的时间
* TimeUnit unit：非核心线程保持存活的时间单位
* BlockingQueue<Runnable> workQueue：线程池任务队列
  * ArrayBlockingQueue：一个基于数组结构的有界阻塞队列
  * LinkedBlockingQueue：一个基于链表结构的阻塞队列
  * SynchronousQueue：不存储元素的阻塞队列
  * PriorityBlockingQueue：有优先级的无界阻塞队列
* ThreadFactory threadFactory：线程的创建工厂
* RejectedExecutionHandler handler：线程池饱和拒绝策略
  * AbortPolicy：直接抛出异常，默认策略
  * CallerRunsPolicy：用调用者所在线程来运行任务
  * DiscardOldestPolicy：丢弃任务队列里最老的任务
  * DiscardPolicy：不处理，丢弃当前任务

## Executors

查看Executors提供的一些创建线程池的方法
### newFixedThreadPool

一个固定大小的线程池
```
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>(),
                                  threadFactory);
}
```

* 核心线程数与最大线程数一样
* LinkedBlockingQueue作为任务队列，它是无界队列，不会触发拒绝策略，容易造成OOM。

### newSingleThreadExecutor

单个线程的线程池
```
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

* 核心线程数和最大线程数都是1
* 同样使用LinkedBlockingQueue队列，容易造成OOM。

### newCachedThreadPool

根据线程使用情况创建线程的线程池
```
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

* 核心线程数为0
* 最大线程数是Integer.MAX_VALUE，可以无限创建线程，会造成OOM
* 使用SynchronousQueue不存储元素的阻塞队列
* 非核心线程存活的时间为60秒，超过时间会释放
* 有空闲线程就使用空闲线程执行任务，无空闲线程就创建线程

### newScheduledThreadPool

一个可调度的线程池
```
//ScheduledExecutorService继承ExecutorService
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
```
 
ScheduledThreadPoolExecutor继承ThreadPoolExecutor，实现ScheduledExecutor接口
```
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE,
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
          new DelayedWorkQueue());
}
```

* 自定义核心线程数数
* 最大线程数是Integer.MAX_VALUE，同样会造成OOM
* DelayedWorkQueue是一个类似PriorityQueue优先级阻塞队列，是线程安全的
* 可执行周期性任务，封装ScheduledFutureTask类，作为调度任务的实现

 
> 常用的阻塞队列主要有以下几种：

* ArrayBlockingQueue：ArrayBlockingQueue（有界队列）是一个用数组实现的有界阻塞队列，按FIFO排序量。
* LinkedBlockingQueue：LinkedBlockingQueue（可设置容量队列）是基于链表结构的阻塞队列，按FIFO排序任务，容量可以选择进行设置，不设置的话，将是一个无边界的阻塞队列，最大长度为Integer.MAX_VALUE，吞吐量通常要高于ArrayBlockingQuene；newFixedThreadPool线程池使用了这个队列
* DelayQueue：DelayQueue（延迟队列）是一个任务定时周期的延迟执行的队列。根据指定的执行时间从小到大排序，否则根据插入到队列的先后排序。newScheduledThreadPool线程池使用了这个队列。
* PriorityBlockingQueue：PriorityBlockingQueue（优先级队列）是具有优先级的无界阻塞队列
* SynchronousQueue：SynchronousQueue（同步队列）是一个不存储元素的阻塞队列，每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于LinkedBlockingQuene，newCachedThreadPool线程池使用了这个队列。
