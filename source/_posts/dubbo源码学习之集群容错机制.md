---
title: dubbo源码学习之集群容错机制
date: 2019-11-23 12:15:56
categories: Java
tags:
- dubbo
---

本文记录源码学习dubbo集群容错机制实现笔记

<!--more-->

集群cluster的作用是将多个服务合并成一个cluster invoker，并将这个invoker暴露给服务消费者。这样一来，服务消费者只需要通过这个invoker进行远程调用即可，至于调用哪个服务提供者，以及调用失败后如何处理等问题，现在都交给集群模块去处理。
![集群工作过程](cluster.png)

集群工作过程可分为两个阶段。第一个阶段是在服务消费者初始化期间，集群cluster实现类为服务消费者创建Cluster invoker 实例，即上图的merge操作。第二阶段是在服务消费者进行远程调用时，具体的Cluster invoker首先会调用Directory的list方法列举invoker列表，Directory的用途是保存invoker。然后通过LoadBalance从invoker列表中选择一个invoker，再调用invoker的invoker方法，完成最后的调用。

Dubbo提供了多种集群实现:
- AvailableCluster 选择第一个可用
- BroadcastCluster 广播
- FailbackCluster  失败自动恢复
- FailfastCluster  快速失败
- FailoverCluster  失败自动切换
- FailsafeCluster  失败安全
- ForkingCluster   并行调用多个服务提供者
- MergeableCluster 

Dubbo的集群实现也是遵循SPI机制，上述所有的Cluster实现类都是实现了Cluster接口。Cluster接口上定义了一个SPI注解，默认扩展为failover。每个集群实现的invoker实现，又交个各自的xxxClusterInvoker实现。这些ClusterInvoker又有共同的父类:AbstractClusterInvoker，下面就从AbstractClusterInvoker开始分析dubbo的集群实现。

## AbstractClusterInvoker的invoker方法
在分析各个集群容错策略前，先来了解一下AbstractClusterInvoker的invoker方法。
```java
public Result invoke(final Invocation invocation) throws RpcException {
    checkWhetherDestroyed();

    // binding attachments into invocation.
    Map<String, String> contextAttachments = RpcContext.getContext().getAttachments();
    if (contextAttachments != null && contextAttachments.size() != 0) {
        ((RpcInvocation) invocation).addAttachments(contextAttachments);
    }

    List<Invoker<T>> invokers = list(invocation);
    LoadBalance loadbalance = initLoadBalance(invokers, invocation);
    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
    return doInvoke(invocation, invokers, loadbalance);
}
```
invokers为服务提供者列表，通过调用list方法，从Directory中获取。然后根据spi机制，获取具体的负载均衡实现，然后执行doInvoker方法。doInvoker方法就在具体的实现类里实现。

## 源码分析AvailableClusterInvoker实现
它实现的策略是选择集群第一个可用的服务提供者。这个策略的缺点是相当于一个服务的主备，同时只有一个服务承载流量，并不能起到负载均衡的作用。

该类的invoker方法实现逻辑很简单:
```java
public class AvailableClusterInvoker<T> extends AbstractClusterInvoker<T> {

    public AvailableClusterInvoker(Directory<T> directory) {
        super(directory);
    }

    @Override
    public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        for (Invoker<T> invoker : invokers) {
            if (invoker.isAvailable()) {
                return invoker.invoke(invocation);
            }
        }
        throw new RpcException("No provider available in " + invokers);
    }
}
```
在doInvoker方法中，循环invoker列表，只要其中一个invoker可用（invoker.isAvailable（）== true）即返回该invoker。如果所有invoker都不可用，则抛出RpcException，表示无可用的provider。
## 源码分析BroadcastClusterInvoker实现
BroadcastClusterInvoker为广播调用实现，就是将调用所有的服务提供者，一个调用失败，并不会熔断，并且如果一个调用失败，则整个调用失败。比较适合刷新缓存的场景。

来看下doInvoker的实现逻辑:
```java
public class BroadcastClusterInvoker<T> extends AbstractClusterInvoker<T> {

    private static final Logger logger = LoggerFactory.getLogger(BroadcastClusterInvoker.class);

    public BroadcastClusterInvoker(Directory<T> directory) {
        super(directory);
    }

    @Override
    @SuppressWarnings({"unchecked", "rawtypes"})
    public Result doInvoke(final Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        checkInvokers(invokers, invocation);
        RpcContext.getContext().setInvokers((List) invokers);
        RpcException exception = null;
        Result result = null;
        for (Invoker<T> invoker : invokers) {
            try {
                result = invoker.invoke(invocation);
            } catch (RpcException e) {
                exception = e;
                logger.warn(e.getMessage(), e);
            } catch (Throwable e) {
                exception = new RpcException(e.getMessage(), e);
                logger.warn(e.getMessage(), e);
            }
        }
        if (exception != null) {
            throw exception;
        }
        return result;
    }
}
```
具体是逻辑主要是for循环体内，循环每个invoker，调用其invoker方法。如果调用当前invoker调用发生了异常，并不会立即返回。等到所有的调用完成，如果检测到发生了exception，则抛出异常，表示本次调用失败。

## 源码分析FailbackClusterInvoker实现
Failback表示调用失败后，返回成功，但会在后台定时重试，重试次数。通常用于消息通知，但消费者重启后，重试任务丢失。

```java
protected Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    Invoker<T> invoker = null;
    try {
        checkInvokers(invokers, invocation);
        invoker = select(loadbalance, invocation, invokers, null); // ①
        return invoker.invoke(invocation);
    } catch (Throwable e) {
        // log
        addFailed(loadbalance, invocation, invokers, invoker); // ②
        return AsyncRpcResult.newDefaultAsyncResult(null, null, invocation); // ignore
    }
}
```
①首先根据loadBalance选择invoker，服务的负责均衡实现在上一篇已经分析了，然后执行invoker方法；②如果调用发生了异常，则将本次调用信息执行重试逻辑。

**addFailed方法实现**
```java
private void addFailed(LoadBalance loadbalance, Invocation invocation, List<Invoker<T>> invokers, Invoker<T> lastInvoker) {
    if (failTimer == null) {
        synchronized (this) {
            if (failTimer == null) {
                failTimer = new HashedWheelTimer(
                        new NamedThreadFactory("failback-cluster-timer", true),
                        1,
                        TimeUnit.SECONDS, 32, failbackTasks); // ①
            }
        }
    }
    RetryTimerTask retryTimerTask = new RetryTimerTask(loadbalance, invocation, invokers, lastInvoker, retries, RETRY_FAILED_PERIOD); // ②
    try {
        failTimer.newTimeout(retryTimerTask, RETRY_FAILED_PERIOD, TimeUnit.SECONDS); // ③
    } catch (Throwable e) {
        logger.error("Failback background works error,invocation->" + invocation + ", exception: " + e.getMessage());
    }
}
```
①首先判断failTimer是否为null，如果为null，则new一个HashedWheelTimer定时器；②创建一个RetryTimerTask；③使用HashedWheelTimer提交重试任务（延时5s执行）。

**来看看RetryTimerTask类的定义**
```java
private class RetryTimerTask implements TimerTask {
    private final Invocation invocation;
    private final LoadBalance loadbalance;
    private final List<Invoker<T>> invokers;
    private final int retries;
    private final long tick;
    private Invoker<T> lastInvoker;
    private int retryTimes = 0;

    RetryTimerTask(LoadBalance loadbalance, Invocation invocation, List<Invoker<T>> invokers, Invoker<T> lastInvoker, int retries, long tick) {
        this.loadbalance = loadbalance;
        this.invocation = invocation;
        this.invokers = invokers;
        this.retries = retries;
        this.tick = tick;
        this.lastInvoker=lastInvoker;
    }

    @Override
    public void run(Timeout timeout) {
        try {
            Invoker<T> retryInvoker = select(loadbalance, invocation, invokers, Collections.singletonList(lastInvoker)); // ①
            lastInvoker = retryInvoker; // ②
            retryInvoker.invoke(invocation); // ③
        } catch (Throwable e) {
            logger.error("Failed retry to invoke method " + invocation.getMethodName() + ", waiting again.", e);
            if ((++retryTimes) >= retries) { // ④
                logger.error("Failed retry times exceed threshold (" + retries + "), We have to abandon, invocation->" + invocation);
            } else {
                rePut(timeout); // ⑥
            }
        }
    }

    private void rePut(Timeout timeout) {
        if (timeout == null) {
            return;
        }

        Timer timer = timeout.timer();
        if (timer.isStop() || timeout.isCancelled()) {
            return;
        }
        // 再次提交定时任务
        timer.newTimeout(timeout.task(), tick, TimeUnit.SECONDS);
    }
}
```
RetryTimerTask的各个属性都包含了本次调用的信息。①根据loadBalance选择invoker；②将选择出来的invoker赋值给lastInvoker；③执行invoker；④如果再次失败，判断失败次数是否大于等于配置的retries值，如果满足调节，则打印失败日志；⑤第④步中条件不满足，则再次将提交定时任务，再走一遍run方法的逻辑。

第②步中为了不再选择调用失败了的invoker。select方法接收了之前所有选择过的的invoker，再进行选择的时候，就会将这些排出掉。

## 源码分析FailfastClusterInvoker实现
FailFast快速失败，服务调用失败后立马抛出异常，不进行重试。适用于幂等操作，比如新增记录
```java
public class FailfastClusterInvoker<T> extends AbstractClusterInvoker<T> {

    public FailfastClusterInvoker(Directory<T> directory) {
        super(directory);
    }

    @Override
    public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        checkInvokers(invokers, invocation);
        Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
        try {
            return invoker.invoke(invocation);
        } catch (Throwable e) {
            if (e instanceof RpcException && ((RpcException) e).isBiz()) { // biz exception.
                throw (RpcException) e;
            }
            // throw RpcException
        }
    }
}
```
实现逻辑:通过select方法选择invoker，然后进行调用，如果调用失败，则立即抛出异常。

## 源码分析FailoverClusterInvoker实现
Failover是dubbo集群实现的缺省默认配置，在调用失败时，会自动切换invoker进行重试。

```java
public class FailoverClusterInvoker<T> extends AbstractClusterInvoker<T> {

    private static final Logger logger = LoggerFactory.getLogger(FailoverClusterInvoker.class);

    public FailoverClusterInvoker(Directory<T> directory) {
        super(directory);
    }

    @Override
    @SuppressWarnings({"unchecked", "rawtypes"})
    public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        List<Invoker<T>> copyInvokers = invokers;
        checkInvokers(copyInvokers, invocation);
        String methodName = RpcUtils.getMethodName(invocation);
        int len = getUrl().getMethodParameter(methodName, RETRIES_KEY, DEFAULT_RETRIES) + 1; // 执行调用的次数，配置的retries+1，retries默认为2，所以len默认值为3
        if (len <= 0) {
            len = 1;
        }
        // retry loop.
        RpcException le = null; // last exception.
        List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyInvokers.size()); // invoked invokers.
        Set<String> providers = new HashSet<String>(len);
        for (int i = 0; i < len; i++) {
            if (i > 0) { // 表示进行了重试
                checkWhetherDestroyed();
                // 再次调用list方法回去invoker列表，这样做的好处是如果某个服务挂了，可以通过该方法拿到最新的ivoker列表
                copyInvokers = list(invocation); 
                // 再次检测invoker列表
                checkInvokers(copyInvokers, invocation);
            }
            // 负载均衡选择对应的invoker
            Invoker<T> invoker = select(loadbalance, invocation, copyInvokers, invoked);
            invoked.add(invoker);
            RpcContext.getContext().setInvokers((List) invoked);
            try {
                Result result = invoker.invoke(invocation);
                if (le != null && logger.isWarnEnabled()) {
                    // log
                }
                return result;
            } catch (RpcException e) {
                if (e.isBiz()) { // biz exception.
                    throw e;
                }
                le = e;
            } catch (Throwable e) {
                le = new RpcException(e.getMessage(), e);
            } finally {
                providers.add(invoker.getUrl().getAddress());
            }
        }
        // throw new RpcException
    }
}
```
这里的实现首先获取重试次数，然后根据重试次数进行循环调用，失败进行重试。这里需要划重点的是：每次重试都会调用list方法获取invoker列表信息，这样的做的好处，如果某个服务挂了，可以获取到最新的invoker列表信息。

## 源码分析FailsafeClusterInvoker实现
Failsafe是一种失败安全的ClusterInvoker。所谓的失败安全是值，调用过程中出现异常时，仅仅打印异常，不会抛出异常。适用于写入审计入职等操作。
```java
public class FailsafeClusterInvoker<T> extends AbstractClusterInvoker<T> {
    private static final Logger logger = LoggerFactory.getLogger(FailsafeClusterInvoker.class);

    public FailsafeClusterInvoker(Directory<T> directory) {
        super(directory);
    }

    @Override
    public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        try {
            checkInvokers(invokers, invocation);
            Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
            return invoker.invoke(invocation);
        } catch (Throwable e) {
            logger.error("Failsafe ignore exception: " + e.getMessage(), e);
            return AsyncRpcResult.newDefaultAsyncResult(null, null, invocation); // ignore
        }
    }
}
```
这个方法逻辑很简单，首先根据loadBalance选择invoker，然后执行invoker方法，如果调用发生了异常，打印异常日志。

## 源码分析ForkingClusterInvoker实现
Forking会在运行时通过线程池创建过个线程，并发调用多个服务器。只要有一个服务提供者成功返回结果，则返回结果。适用于一些对实时性要求较高的读操作，但是比较消耗资源。

```java
public class ForkingClusterInvoker<T> extends AbstractClusterInvoker<T> {
    private final ExecutorService executor = Executors.newCachedThreadPool(
            new NamedInternalThreadFactory("forking-cluster-timer", true));

    public ForkingClusterInvoker(Directory<T> directory) {
        super(directory);
    }

    @Override
    @SuppressWarnings({"unchecked", "rawtypes"})
    public Result doInvoke(final Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        try {
            checkInvokers(invokers, invocation);
            final List<Invoker<T>> selected;
            // 获取forks的配置，缺省配置为2
            final int forks = getUrl().getParameter(FORKS_KEY, DEFAULT_FORKS);
            // 获取timeout配置，缺省配置为1000
            final int timeout = getUrl().getParameter(TIMEOUT_KEY, DEFAULT_TIMEOUT);
            // 如果配置的forks不合法，这将invoker列表赋值给selected
            if (forks <= 0 || forks >= invokers.size()) {
                selected = invokers;
            } else {
                selected = new ArrayList<>();
                // 循环forks次数，选择forks个数的invoker，放入selected列表
                for (int i = 0; i < forks; i++) {
                    // loadBalance选择invoker
                    Invoker<T> invoker = select(loadbalance, invocation, invokers, selected);
                    // 去重
                    if (!selected.contains(invoker)) {
                        selected.add(invoker);
                    }
                }
            }
            RpcContext.getContext().setInvokers((List) selected);
            final AtomicInteger count = new AtomicInteger();
            final BlockingQueue<Object> ref = new LinkedBlockingQueue<>();
            // 循环selected列表
            for (final Invoker<T> invoker : selected) {
                // 为每个invoker创建一个执行线程
                executor.execute(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            // 执行远程调用
                            Result result = invoker.invoke(invocation);
                            // 将结果放入阻塞队列中
                            ref.offer(result);
                        } catch (Throwable e) {
                            int value = count.incrementAndGet();
                            // 只有value大于等于selected的size时，才将异常放入阻塞队列
                            if (value >= selected.size()) {
                                ref.offer(e);
                            }
                        }
                    }
                });
            }
            try {
                // 取出结果
                Object ret = ref.poll(timeout, TimeUnit.MILLISECONDS);
                if (ret instanceof Throwable) { // 如果取出的结果是异常，则抛出异常
                    Throwable e = (Throwable) ret;
                    // throw new RpcException
                }
                return (Result) ret;
            } catch (InterruptedException e) {
                // throw new RpcException
            }
        } finally {
            // clear attachments which is binding to current thread.
            RpcContext.getContext().clearAttachments();
        }
    }
}
```
这个方法大致分为两部分：第一部分选择出需要执行远程调用的invoker；第二部分为选择的invoker创建线程，执行任务；选择invoker的过程中，主要是根据forks的配置，选择需要选择的invoker各个数；线程池任务执行时，执行远程调用，会将结果放入阻塞队列，但是如果调用发生了异常，并不会立即将异常放入阻塞队列，而是判断失败的次数时候等于选择的invoker个数。这样做是因为在并行调用多个服务的情况下，只要有一个服务提供者能够成功返回结果，而其它全部失败。此时仍应该返回成功的结果，而非抛出异常，在`value >= selected.size()`时将异常放入阻塞队列，可以保证异常对象不会出现在正常结果的前面，这样可以从则塞队列中优先取出正常的结果。

