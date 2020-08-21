---
layout: post
category: 计算机基础
title: Java回望(5)-多线程
tagline: by Snail
tags: 
  - Java
published: true
date: 2020-05-15 21:32:42 +08:00
---	

让我们继续Java回望系列，今天来回顾下平时使用的多线程方案和其中的一些知识点。
<!--more-->

## ThreadPoolExecutor
作为一个常用的线程池类，可以通过构造函数中的参数来理解它。具体参数列举如下：
- int corePoolSize：线程池中一直有的线程个数，即使空闲也不会被回收（可以通过设置"允许核心线程超时"参数来使它超时回收）
- int maximumPoolSize：线程池中可持有的最多线程数
- long keepAliveTime：超过corePoolSize线程数的线程在被销毁之前等待新任务到达的最长时间
- TimeUnit unit：keepAliveTime参数的单位
- BlockingQueue<Runnable> workQueue：线程池的等待队列，被execute方法提交的任务进入这一队列
- ThreadFactory threadFactory：线程工厂，可自定义线程创建的过程
- RejectedExecutionHandler handler：拒绝处理器，负责在workQueue满的时候处理新提交的任务

## ExecutorService
juc包里一个十分方便的线程池工具，可以由Executors的静态方法来创建。常用的方法分别列举如下
- newCachedThreadPool：创建无界的线程池。适用于：响应时间要求高、数据量可控的场景。有OOM风险。
```
//创建CachedThreadPool
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```
- newFixedThreadPool：创建固定线程数的线程池，适用于线程资源有限，数据量较小或不可控场景。OOM风险较小。
```
//创建FixedThreadPool
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```
除此之外，ScheduledExecutorService被用来：
在创建时，还可以传入ThreadFactory来自定义线程创建时的一些属性。比如通过实现newThread方法，可以定义thread的名字格式等信息。创建完成后，通过调用submit(Runnable)来向线程池提交任务。

点开newCachedThreadPool和newFixedThreadPool的实现方法看，实际上会发现其底层均为实例化了ThreadPoolExecutor对象，通过不同的参数来实例化。

调用ExecutorService.submit(Callable<T> callable)方法会返回Future<T>，Future中会包含callable的返回结果，可以同时提交多个任务并将future加到list中，然后在list统一处理并发处理的结果。
```
List<Future<List<String>>> futureList = new ArrayList<>(2);
futureList.add(executors.submit(() -> method1()));
futureList.add(executors.submit(() -> method2()));

for (Future<List<String>> future : futureList) {
    future.get();//do something here
}
```

### 队列
每当新的任务提交给线程池时，它会被放进线程池实例化时指定的队列中。常用的队列有如下几种：
- SynchronousQueue：
    - SynchronousQueue没有容量，是无缓冲等待队列，是一个不存储元素的阻塞队列
    - 会直接将任务交给消费者，必须等队列中的添加元素被消费后才能继续添加新的元素。   
    - newCachedThreadPool使用了这一队列
- LinkedBlockingQueue：
    - 当前执行的线程数量达到corePoolSize的数量时，剩余的元素会在阻塞队列里等待。（所以在使用此阻塞队列时maximumPoolSizes就相当于无效了），每个线程完全独立于其他线程。
    - 生产者和消费者使用独立的锁来控制数据的同步，即在高并发的情况下可以并行操作队列中的数据。
    - newFixedThreadPool使用了这一队列
- ArrayBlockingQueue：
    - ArrayBlockingQueue是一个有界缓存等待队列，可以指定缓存队列的大小
    - 当正在执行的线程数等于corePoolSize时，多余的元素缓存在ArrayBlockingQueue队列中等待有空闲的线程时继续执行
    - 当ArrayBlockingQueue已满时，加入ArrayBlockingQueue失败，会开启新的线程去执行
    - 当线程数已经达到最大的maximumPoolSizes时，再有新的元素尝试加入ArrayBlockingQueue时会报错

### 拒绝策略
4种拒绝策略：
- AbortPolicy（抛出一个异常，默认的拒绝策略）：在无法接受任务时，抛出一个RejectedExecutionException异常。
- DiscardPolicy（直接丢弃任务）：在无法接受任务时，丢弃任务。适用于不重要的业务场景。
- DiscardOldestPolicy（丢弃队列里最老的任务，将当前这个任务继续提交给线程池）：丢弃队列中最老的任务，并将新任务添加到队列中。
- CallerRunsPolicy（交给线程池调用所在的线程进行处理）：由提交任务的那个线程来处理该任务。

### callable vs runnable
平时使用的ExecutorService.submit方法里，常常会定义一个lambda表达式代表的匿名函数，这个时候实际上是实现了一个callable接口。和callable相似的还有一个接口是runnable，他俩的关系如下：

区别:
- callable可以抛异常, runnable不能
- callable可以有返回值, runnable不能

相同点: 
- 两者都是接口；
- 两者都可用来编写多线程程序；
- 两者都需要调用Thread.start()启动线程；

### 如何检测线程（池）状态
对于线程状态，我们可以通过以下几种方式观测：
- 使用JConsole查看线程状态
- 扩展ThreadPoolExecutor，重写beforeExecute和afterExecute，在这两个方法里分别做一些任务执行前和任务执行后的相关监控逻辑，还有个terminated方法，是在线程池关闭后回调

对于线程池状态，我们可以利用ThreadPoolExecutor的一些方法来查看线程池的指标。对于executorService我们可以强制类型转换ThreadPoolExecutor tpe = ((ThreadPoolExecutor) executorService);接着利用如下方法：
- tpe.getQueue().size();当前排队线程数
- tpe.getActiveCount();当前活动线程数
- tpe.getCompletedTaskCount();执行完成线程数
- tpe.getTaskCount();总线程数

## 如何选定线程数
大概可以从如下几点考虑：
- CPU数量，对于CPU密集型程序，可以以CPU数量为基准设置线程数
- 内存大小，每个线程都需要内存开销，因此也要结合实际的内存大小来设置线程数
- 业务逻辑，还是要根据实际的业务场景进行多次调试，选取出最适合的线程数

参考：
- [优雅的使用Java线程池](https://zhuanlan.zhihu.com/p/60986630)
- [线程池的三种队列区别：SynchronousQueue、LinkedBlockingQueue 和ArrayBlockingQueue](https://blog.csdn.net/qq_26881739/article/details/80983495)
- [callable和runnable的区别](https://www.cnblogs.com/zjulanjian/p/11132605.html)
- [Java并发编程之线程池任务监控](https://blog.csdn.net/a1282379904/article/details/77894368)
- [多线程的那点事儿（1）－－如何选择线程数](https://blog.csdn.net/Nocky/article/details/8487578)