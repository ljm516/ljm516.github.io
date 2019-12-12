---
title: java中线程池
date: 2019-10-21 20:58:33
categories: Java
tags: 
- 多线程
---

java中的线程池知识点。

<!--more-->

## 什么是线程池
为了避免系统频繁的创建和销毁线程，可以让线程复用。线程池中总是保持几个活跃线程，当需要使用时，从池子中随便哪一个空闲线程，当完成工作时，并不着急关闭线程，而是将这个线程退回到线程池中，方便其它人使用。

## JDK对线程池的支持
JDK提供了一套Executor框架，帮助开发人员更好的进行线程控制。核心成员如下图所示:
![线程池相关类](线程池相关类.png)

Executors类扮演者线程池工厂角色，通过它可以获取获取指定功能的线程池。Executors提供了以下主要的线程池创建方法，每个方法返回不同特性的线程池:
```java
public static ExecutorService Executors.newFixedThreadPool(int nThreads);
public static ExecutorService Executors.newCachedThreadPool();
public static ExecutorService Executors.newSingleThreadExecutor();
public static ScheduledExecutorService Executors.newScheduledThreadPool(int corePoolSize);
public static ScheduledExecutorService Executors.newSingleThreadScheduledExecutor();
```
- newFixedThreadPool(int nThreads)方法: 该方法返回一个nThreads数量的线程池，该线程池中线程数不变。当有个新的任务需要提交时，线程池如果有空闲的线程，则立即执行；如果没有，则将任务放入任务队列，等待有线程空闲，再处理任务队列里的任务。
- newCachedThreadPool()方法: 该方法返回一个可以根据实际情况调整线程数量的线程池。线程池中线程数不确定，若有线程可以复用，则优先使用可服用的线程。若所有线程均被占用，又有新的任务提交，则创建新的线程执行任务，待任务完成，将线程放回线程池进行复用。
- newSingleThreadExecutor()方法: 该方法返回只有一个线程的线程池。如果多余一个任务被提交到线程池，则将任务放入到任务队列，待线程空闲时，在从任务队列中按FIFO的方法提交任务。
- newScheduledThreadPool(int corePoolSize)方法: 该方法返回一个ScheduledExecutorService对象，该线程可以指定线程数。该线程池用于执行定时任务或周期性的执行任务。
- newSingleThreadScheduledExecutor()方法: 和newScheduledThreadPool(int corePoolSize)方法一样，但是返回的线程池只有一个线程。

## ExecutorService 示例
```java
public class ExecutorsSample {
    private static volatile int nums = 0;
    private static ReentrantLock lock = new ReentrantLock();
    private static CountDownLatch countDownLatch = new CountDownLatch(10);

    public static void main(String[] args) throws InterruptedException {
        submitCallableTask();
    }
    // 使用execute方法执行一个任务
    public static void execute() {
        ExecutorService executorService = Executors.newFixedThreadPool(5);
        for (int i = 0; i < 10; i++) {
            executorService.execute(new RunnableTask());
        }
        executorService.shutdown();
    }
    // 使用submit方法提交一个实现Runnable接口的任务
    public static void submitRunnableTask() throws InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(5);
        List<Future<?>> futureList = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            Future<?> future = executorService.submit(new RunnableTask());
            futureList.add(future);
        }
        countDownLatch.await();
        for (Future<?> future : futureList) {
            try {
                System.out.println("runnableTask future: " + future.get());
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        }
        executorService.shutdown();
    }
    // 使用submit方法提交一个实现Callable接口的任务
    private static void submitCallableTask() throws InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(5);
        List<Future<?>> futureList = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            Future<?> future = executorService.submit(new CallableTask());
            futureList.add(future);
        }
        countDownLatch.await();
        for (Future<?> future : futureList) {
            try {
                System.out.println("callableTask future: " + future.get());
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        }
        executorService.shutdown();
    }
    // 实现runnable接口的任务
    static class RunnableTask implements Runnable {
        @Override
        public void run() {
            try {
                Thread.sleep(1000);
                System.out.println(Thread.currentThread().getName() + " finished,time=" + System.currentTimeMillis());
                countDownLatch.countDown();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    // 实现Callable接口的任务，返回一个Integer的值，进行nums的自增操作
    static class CallableTask implements Callable<Integer> {
        @Override
        public Integer call() {
            try {
                lock.lock();
                for (int i = 0; i < 100; i++) {
                    nums++;
                }
                countDownLatch.countDown();
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
            return nums;
        }
    }
}
```
上面的例子，延时了ExecutorService的execute和submit方法的使用。可以看出，execute方法只能提交一个Runnable的任务，且方法没有返回值；submit方法可以提交一个Runnable或者Callable的任务，且方法具有返回值。

## ScheduledExecutorService 示例

### scheduleAtFixedRate
```java
public static void scheduleAtFixedRate() {
    ScheduledExecutorService scheduledExecutorService = Executors.newSingleThreadScheduledExecutor();
    scheduledExecutorService.scheduleAtFixedRate(new RunnableTask(), 1000, 2000, TimeUnit.MILLISECONDS);
}
static class RunnableTask implements Runnable {
    @Override
    public void run() {
        try {
            Thread.sleep(1000);
            System.out.println(Thread.currentThread().getName() + " finished,time=" + System.currentTimeMillis() / 1000);
            countDownLatch.countDown();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
上面的代码，创建了一个单线程ScheduledExecutorService线程池，然后使用scheduleAtFixedRate方法提交了RunnableTask，延迟1s执行，每2s执行一次。然后RunnableTask进行休眠1s。看输出:
```
current time: 1575985038
pool-1-thread-1 finished,time=1575985040
pool-1-thread-1 finished,time=1575985042
pool-1-thread-1 finished,time=1575985044
pool-1-thread-1 finished,time=1575985046
pool-1-thread-1 finished,time=1575985048
pool-1-thread-1 finished,time=1575985050
pool-1-thread-1 finished,time=1575985052
.....
```
当前时间和第一行时差为2s，是因为RunnableTask先休眠了1s，然后进行输出。所以可以看出，从提交任务开始，先延迟了1s执行，然后每次任务完成时输出的时间间隔都为2s。那如果RunnableTask执行时间大于执行周期呢，如上面的例子，如果在RunnableTask休眠5s，周期还是2s，会不会出现任务重叠的问题呢？我们将休眠时间Thread.sleep(5000);看输出:
```
pool-1-thread-1 finished,time=1575985390
pool-1-thread-1 finished,time=1575985395
pool-1-thread-1 finished,time=1575985400
pool-1-thread-1 finished,time=1575985405
pool-1-thread-1 finished,time=1575985410
pool-1-thread-1 finished,time=1575985415
pool-1-thread-1 finished,time=1575985420
.....
```
可以看出，任务执行的周期变成了5s，也就是说如果执行时间大于执行周期，那么任务就会在上一个任务完成后立即执行。

### scheduleWithFixedDelay
```java
private static void scheduleWithFixedDelay() {
    ScheduledExecutorService scheduledExecutorService = Executors.newSingleThreadScheduledExecutor();
    scheduledExecutorService.scheduleWithFixedDelay(new RunnableTask(), 1000, 2000, TimeUnit.MILLISECONDS);
}
```
scheduleWithFixedDelay方式是在上一个任务执行完成后，在经过一个delay的时间执行。看上面例子的输出:
```
pool-1-thread-1 finished,time=1575985792
pool-1-thread-1 finished,time=1575985795
pool-1-thread-1 finished,time=1575985798
pool-1-thread-1 finished,time=1575985801
pool-1-thread-1 finished,time=1575985804
pool-1-thread-1 finished,time=1575985807
.....
```
每次输出的时间间隔是3秒，这是因为RunnableTask执行了1秒(休眠)，也就是任务执行时间为1s，说明了下一个任务开始上一个任务结束在过2秒提交执行的。相应的，如果执行时间为5s，输出如下:
```
pool-1-thread-1 finished,time=1575986015
pool-1-thread-1 finished,time=1575986022
pool-1-thread-1 finished,time=1575986029
pool-1-thread-1 finished,time=1575986036
```
可以看出，每次的间隔是7s（2+5）。

#### scheduleAtFixedRate和scheduleWithFixedDelay的区别
这两者的区别，官方文档解释如下:
- scheduleAtFixedRate: 创建一个周期性任务。任务开始于一个给定的初始延迟时间，后续的任务按照给定的周期进行；后续一个任务将会在initialDelay+period时执行，第二个任务将在initialDelay+2*period，依次类推。
- scheduleWithFixedDelay: 创建并执行一个周期性任务。任务开始于一个给定的初始延迟时间，后续任务将会按给定的延时进行:即上一个任务的结束时间到下一个任务开始的时间差。

## 线程池内部实现
对于核心的几个线程池，不管是newFixedThreadPool()方法、newSingleThreadExecutor()方法还是newCacheThreadPool()方法，虽然创建的线程池有着不同的特性，但其内部实现均使用了ThreadPoolExecutor类。
```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>());
}
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>()));
}
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS, new SynchronousQueue<Runnable>());
}
```
从上面的实现可以看出，都是对ThreadPoolExecutor的封装，来看下它的最重要的构造函数:
```java
public ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory, RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.acc = System.getSecurityManager() == null ?
            null :
            AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```
构造函数各个参数含义:
- corePoolSize: 指定线程池中线程数量
- maximumPoolSize: 指定了线程中最大的线程数
- keepAliveTime: 当线程池线程数量超过corePoolSize时，多余的空闲线程的存活时间。即超过corePoolSize的空闲线程，在多长时间内会被销毁
- unit: keepAliveTime的单位
- workQueue: 任务队列，被提交但尚未被执行的任务
- threadFactory: 线程工厂，用于创建线程，一般用默认的即可
- handler: 拒绝策略。当任务太多来不及处理时，如何拒绝任务

其它参数都比较简单易懂，下面主要详细介绍workQueue和handler这两个参数。

### 任务队列-workQueue
workQueue指被提交但未执行的任务队列，是一个BlockingQueue接口的对象，仅用于存放任务对象。根据队列功能的分类，在ThreadPoolExecutor类的构造函数中可使用以下几种BlockingQueue接口:
- 直接提交的队列: 该功能由SynchronousQueue对象提供。它是一个特殊的BlockingQueue，它没有容量。如果使用SynchronousQueue，则提交的任务不会被真实的保存，而总是将新任务提交给线程执行，如果没有空闲的进程，则尝试创建新的进程，如果进程数量已经达到最大值，则执行拒绝策略。因此，如果使用SynchronousQueue队列，通常要设置很大的maximumPoolSize值，否则很容易就会执行拒绝策略。
- 有界的任务队列: 有界的任务队列可以使用ArrayBlockingQueue类实现。其构造函数 必须带一个容量参数，表示该对列的最大容量。当使用有界的任务队列时，若有新的任务需要执行，如果线程池的实际线程数小于corePoolSize，则会优先创建新的线程，若大于corePoolSize，则会将新任务加入到等待队列。若队列已满，则在总线程数不大于maximumPoolSize的前提下，创建新的线程执行任务。若大于maximumPoolSize，则执行拒绝策略。流程如图所示:
![有界队列任务流转](有界队列任务流转.png)
- 无界的任务队列: 无界任务队列可以通过LinkedBlockingQueue类实现。和有界队列相比，除非系统资源耗尽，否则无界的任务队列不存在任务入队失败的情况。当有新的任务提交时，如果线程数小于corePoolSize时，线程池会生成新的线程执行任务，但当系统的线程数达到corePoolSize后，就不会继续增加了。若后续有新的任务加入，而又没有空闲的线程资源，则任务直接进入队列等待。若任务创建和处理的速度差异很大，无界队列会保持快速增长，直到耗尽系统资源。
- 优先任务队列: 优先任务队列是带有执行优先级的队列。它通过PriorityBlockingQueue类实现，可以控制任务的执行先后顺序，它是一个特殊的无界队列。无论是有界的ArrayBlockingQueue或者是无界的LinkedBlockingQueue都是按照先进先出算法处理任务的。而PriorityBlockingQueue则可以根据任务自身的优先级顺序先后执行（优先级高的先执行）。

回顾newFixedThreadPool()方法的是实现，它返回一个corePoolSize和maximumPoolSize等量的，并且使用了LinkedBlockingQueue任务队列的线程池。因为对于固定大小的线程池，不存在线程数量的动态变化，因此corePoolSize和maximumPoolSize可以相等。同时，使用无界队列存放无法立即执行的任务，当任务提交非常频繁的时候，队列可能迅速膨胀，从而耗尽系统资源。

newSingleThreadExecutor()方法返回的是单线程的线程池，是newFixedThreadPool()的退化，只是简单的将线程池数量设置为1。

newCacheThreadPool()方法返回corePoolSize为0，maximumPoolSize为无穷大的线程池，这意味着如果没有任务，该线程池内无线程，而当任务提交时，该线程池会使用空闲的线程执行任务，若无空闲线程，则将队伍加入SynchronousQueue队列，而该队列是一种直接提交的队列，它总是迫使线程池增加新的线程执行任务。当任务执行完毕后，由于corePoolSize为0，因此空闲线程在指定时间(60s)被回收。

### 拒绝策略-handler
线程池的拒绝策略，当阻塞队列满了，且没有空闲的工作线程，如果继续提交任务，必须采取一种策略处理该任务，线程池提供了4中策略:
- CallerRunsPolicy: 调用者执行策略
- AbortPolicy: 抛出RejectedExecutionException
- DiscardPolicy: 丢弃策略
- DiscardOldestPolicy: 丢弃最老的任务。

#### CallerRunsPolicy
谁提交的任务，谁去执行。

```java
public static class CallerRunsPolicy implements RejectedExecutionHandler {
    /**
     * Creates a {@code CallerRunsPolicy}.
     */
    public CallerRunsPolicy() { }

    /**
     * Executes task r in the caller's thread, unless the executor
     * has been shut down, in which case the task is discarded.
     *
     * @param r the runnable task requested to be executed
     * @param e the executor attempting to execute this task
     */
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            r.run();
        }
    }
}
```
rejectedExecution的实现，直接执行runable的run方法来执行这个任务。

#### AbortPolicy
中止任务

```java
public static class AbortPolicy implements RejectedExecutionHandler {
    /**
     * Creates an {@code AbortPolicy}.
     */
    public AbortPolicy() { }

    /**
     * Always throws RejectedExecutionException.
     *
     * @param r the runnable task requested to be executed
     * @param e the executor attempting to execute this task
     * @throws RejectedExecutionException always
     */
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        throw new RejectedExecutionException("Task " + r.toString() +
                                             " rejected from " +
                                             e.toString());
    }
}
```
直接抛出 RejectedExecutionException

#### DiscardPolicy
丢弃策略

```java
public static class DiscardPolicy implements RejectedExecutionHandler {
    /**
     * Creates a {@code DiscardPolicy}.
     */
    public DiscardPolicy() { }

    /**
     * Does nothing, which has the effect of discarding task r.
     *
     * @param r the runnable task requested to be executed
     * @param e the executor attempting to execute this task
     */
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    }
}
```
不做任何处理，直接丢弃

#### DiscardOldestPolicy
丢弃阻塞队列中靠最前的任务，并执行当前任务

```java
public static class DiscardOldestPolicy implements RejectedExecutionHandler {
    /**
     * Creates a {@code DiscardOldestPolicy} for the given executor.
     */
    public DiscardOldestPolicy() { }

    /**
     * Obtains and ignores the next task that the executor
     * would otherwise execute, if one is immediately available,
     * and then retries execution of task r, unless the executor
     * is shut down, in which case task r is instead discarded.
     *
     * @param r the runnable task requested to be executed
     * @param e the executor attempting to execute this task
     */
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            e.getQueue().poll();
            e.execute(r);
        }
    }
}
```
调用BlockQueue的poll方法，将最前列的任务丢弃，然后执行当前任务。






