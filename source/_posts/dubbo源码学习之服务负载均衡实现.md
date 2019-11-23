---
title: dubbo源码学习之服务负载均衡实现
date: 2019-11-20 18:29:23
categories: Java
tags:
- dubbo
---

本文记录源码学习dubbo服务负载均衡实现笔记。

<!--more--> 
Dubbo 提供了4种负载均衡实现，分别是基于权重随机算法的 RandomLoadBalance、基于最少活跃调用数算法的 LeastActiveLoadBalance、基于 hash 一致性的 ConsistentHashLoadBalance，以及基于加权轮询算法的 RoundRobinLoadBalance。默认为random随机调用。

用户可以通过 `<dubbo:servie loadbalance="random">`来指定服务的负载均衡实现。loadbalance的配置可以配置在服务提供者service级别、服务提供者service的方法级别、服务消费者service级别以及服务消费着service的方法级别。
```java
<!--服务提供者service级别-->
<dubbo:service interface="..." loadbalance="random"/>
<!--服务提供者service的方法级别-->
<dubbo:service interface="...">
    <dubbo:method name="..." loadbalance="random"/>
</dubbo:service>

<!--服务消费者service级别-->
<dubbo:reference interface="..." loadbalance="random"/>
<!--服务消费者service的方法级别-->
<dubbo:reference interface="...">
    <dubbo:method name="..." loadbalance="random"/>
</dubbo:reference>
```

## 源码分析AbstractLoadBalance
在进入具体的负载均衡算法学习之前，先学习下它们的父类AbstractLoadBalance，这个类实现了LoadBalance接口，并封装了一些公共逻辑。首先看下AbstractLoadBalance类的入口方法:
```java
public <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    if (CollectionUtils.isEmpty(invokers)) {
        return null;
    }
    if (invokers.size() == 1) {
        return invokers.get(0);
    }
    return doSelect(invokers, url, invocation);
}
```
该方法逻辑很简单，首先判断invokers集合里的元素个数，如果invokers集合里只有一个元素，那么直接返回这个invoker对象；否则的话调用doSelect方法去选择具体的invoker，doSelect方法的实现就要进入其子类进行分析。

AbstractLoadBalance 除了实现了 LoadBalance 接口方法，还封装了一些公共逻辑，比如服务提供者权重计算逻辑。具体实现如下：
```java
protected int getWeight(Invoker<?> invoker, Invocation invocation) {
    int weight = invoker.getUrl().getMethodParameter(invocation.getMethodName(), WEIGHT_KEY, DEFAULT_WEIGHT); // ①
    if (weight > 0) {
        long timestamp = invoker.getUrl().getParameter(REMOTE_TIMESTAMP_KEY, 0L); // ②
        if (timestamp > 0L) {
            int uptime = (int) (System.currentTimeMillis() - timestamp); // ③
            int warmup = invoker.getUrl().getParameter(WARMUP_KEY, DEFAULT_WARMUP); // ④
            if (uptime > 0 && uptime < warmup) {
                weight = calculateWarmupWeight(uptime, warmup, weight); // ⑤
            }
        }
    }
    return weight >= 0 ? weight : 0; // ⑥
}
static int calculateWarmupWeight(int uptime, int warmup, int weight) {
    int ww = (int) ((float) uptime / ((float) warmup / (float) weight));
    return ww < 1 ? 1 : (ww > weight ? weight : ww);
}
```
①从URL参数中获取当前方法配置的权重值，配置的key为weight，缺省配置为100；②从url中获取服务提供者启动时间戳，默认值为0；③计算服务运行耗时=当前时间-服务提供者启动时间；④获取服务配置的warnup时间，默认值为10 * 60 * 1000（10分钟）；⑤如果运时间大于0且warnup时间大于启动耗时，调用calculateWarmupWeight方法计算权重；⑥三目运算获取最终的权重值。

**calculateWarmupWeight方法逻辑**
uptime为服务启动耗时，warnup是服务预热时间，默认值为`10*60*1000=600000ms`，weight是配置的权重值，缺省为100。ww=uptime / (warmup / weight)，warmup和weight是个定值，所以warmup / weight也是定值，那么uptime越大，ww值就越大。根据最终返回的权重的三目运算得知，返回的权重值是小于等于weight的，也就是说ww的越大，最终的权重越接近weight值，也可以说明uptime越大，权重越高。

整个权重计算过程，主要用于保证当服务运行时长小于预热时间（warnup）时，对服务进行降权，避免让服务在启动之初就处于高负载状态。

## 基于权重随机算法的 RandomLoadBalance

### 源码分析RandomLoadBalance实现
```java
public class RandomLoadBalance extends AbstractLoadBalance {

    public static final String NAME = "random";

    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        int length = invokers.size(); // ①
        boolean sameWeight = true; // ②
        int[] weights = new int[length]; // ③
        int firstWeight = getWeight(invokers.get(0), invocation); // ④
        weights[0] = firstWeight;
        int totalWeight = firstWeight; // ⑤
        for (int i = 1; i < length; i++) {
            int weight = getWeight(invokers.get(i), invocation); // ⑥
            weights[i] = weight;
            totalWeight += weight;
            if (sameWeight && weight != firstWeight) { // ⑦
                sameWeight = false;
            }
        }
        if (totalWeight > 0 && !sameWeight) { // ⑧
            int offset = ThreadLocalRandom.current().nextInt(totalWeight); // ⑨
            // Return a invoker based on the random value.
            for (int i = 0; i < length; i++) { // ⑩
                offset -= weights[i];
                if (offset < 0) {
                    return invokers.get(i);
                }
            }
        }
        return invokers.get(ThreadLocalRandom.current().nextInt(length)); // ⑪
    }
}
```
①可用invoker的个数；②定义一个标志，判断所有的invoker配置的权重是否相同；③创建一个数组，存放每个invoker的权重值，用以后续使用；④调用getWeight方法获取第一个invoker的权重，用与判断sameWeight；⑤定义totalWeight，为所有invoker权重之和；⑥循环获取每个invoker，并计算其权重；⑦判断sameWeight标志逻辑；⑧如果总权重值大于0并且每个invoker的权重都不一样，则进行随机算法选择invoker；⑨获取totalWeight内的随机整数；⑩基于随机数获取invoker，基于第⑨步的随机数offset，循环每个invoker的权重，offset减去本次invoker的权重之后，结果小于0，则返回本次inovker对象；⑪如果所有的invoker权重相同，随机获取其中一个。

## 基于最少活跃调用数算法的LeastActiveLoadBalance
LeastActiveLoadBalance的特点:
- 相同活跃数的随机，活跃数指调用前后计数差
- 慢的提供者收到更少的请求，因为越慢的提供者的调用前后计数差会越大

### 源码分析LeastActiveLoadBalance实现
最小活跃数理解: 活跃数调用数越小，表明该服务提供者服务效率高，单位时间内可处理更多的请求。所以应当将请求分配给该服务提供者。在具体具体实现中，每个服务提供者对应一个活跃数active。初始情况下，所有服务提供者活跃数都为0，没收到一个请求，activie加1，完成请求后减1。在服务运行一段时间后，性能好的服务提供者处理请求的速度更快，因此活跃数下降的也越快，此时这样的服务提供者能够优先获取到新的服务请求，这就是最小活跃数负载均衡算法的基础思想。 [dubbo负载均衡](http://dubbo.apache.org/zh-cn/docs/source_code_guide/loadbalance.html)

```java
public class LeastActiveLoadBalance extends AbstractLoadBalance {

    public static final String NAME = "leastactive";

    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        // invoker的数量
        int length = invokers.size();
        // 最小活跃调用的invoker个数
        int leastActive = -1;
        // 具有相同最小活跃调用次数的invoker个数
        int leastCount = 0;
        // 具有相同最小活跃调用次数的invoker的下标
        int[] leastIndexes = new int[length];
        // 每个invoker的权重
        int[] weights = new int[length];
        // 所有最小活跃调用invoker的通过warmup计算后的权重之和
        int totalWeight = 0;
        // 第一个最小活跃调用invoker的权重
        int firstWeight = 0;
        // 每个最小活跃调用invoker是否具有相同的权重
        boolean sameWeight = true;

        // 过滤出所有的最少活跃调用数的invoker
        for (int i = 0; i < length; i++) {
            Invoker<T> invoker = invokers.get(i);
            // 获取当前invoker的活跃数
            int active = RpcStatus.getStatus(invoker.getUrl(), invocation.getMethodName()).getActive();
            // 获取invoker的权重，默认100
            int afterWarmup = getWeight(invoker, invocation);
            // 保存起来，后续使用
            weights[i] = afterWarmup;
            // 如果是第一个invoker或者invoker的活跃数小于当前最小活跃数。
            if (leastActive == -1 || active < leastActive) {
                // 将当前invoker的活跃数赋值给leastActive
                leastActive = active;
                // 设置最小活跃调用数为1
                leastCount = 1;
                // 将第一个活跃调用的invoker下标放入leastIndexs的首位
                leastIndexes[0] = i;
                // 设置totalWeight值
                totalWeight = afterWarmup;
                // 记录第一个最小活跃调用的权重
                firstWeight = afterWarmup;
                // 是否具有相同的权重（因为这里只要一个invoker，所以为true）
                sameWeight = true;
                // 如果当前invoker的活跃数等于leastActive，则计算
            } else if (active == leastActive) {
                // 将当前invoker的下标存入leastIndexes中
                leastIndexes[leastCount++] = i;
                // 计算所有最小活跃调用的权重之和
                totalWeight += afterWarmup;
                // 判断是否具有相同的权重（所有的最少活跃调用的invoker）
                if (sameWeight && i > 0
                        && afterWarmup != firstWeight) {
                    sameWeight = false;
                }
            }
        }
        // 如果最小活跃数的invoker只有一个，直接返回该invoker即可
        if (leastCount == 1) {
            return invokers.get(leastIndexes[0]);
        }
        // 如果所有的最小活跃调用数的invoker权重不相等，且权重之和大于0
        if (!sameWeight && totalWeight > 0) {
            // 基于totalWhight获取一个随机数
            int offsetWeight = ThreadLocalRandom.current().nextInt(totalWeight);
            // 基于随机数返回一个invoker，和RandomLoadBalance逻辑一致
            for (int i = 0; i < leastCount; i++) {
                int leastIndex = leastIndexes[i];
                offsetWeight -= weights[leastIndex];
                if (offsetWeight < 0) {
                    return invokers.get(leastIndex);
                }
            }
        }
        // 如果所有的最小活跃调用invoker具有相同的权重，随机返回其中的一个
        return invokers.get(leastIndexes[ThreadLocalRandom.current().nextInt(leastCount)]);
    }
}
```
LeastActiveLoadBalance类上有一段注释是这么介绍的:
> 筛选出最小活跃调用的invoker，然后计算它们的权重和数量。如果只有一个invoker，直接返回；如果有多个invoker，且权重不同，根据权重总和随机获取；如果有多个invoker，且权重相同，随机获取一个。

LeastActiveLoadBalance 很好理解，源码中每一行都有注释，解释其意思，它和RandomLoadBalance不同是，在进行随机前，会根据invoker的active数，选举出最小活跃调用的invoker，然后基于所有的最小活跃调用invoker进行随机权重选择。

## 基于 hash 一致性的 ConsistentHashLoadBalance
一致性hash算法工作原理: 首先根据ip或者其它信息为缓存节点生成一个hash，并将这个hash投射到[0, 2^23 - 1]的圆环上。当有查询或者写入请求时，则为缓存项目的key生成一个hash值。然后查找第一个大于或等于该hash值的缓存节点，并到这个节点查询或写入缓存项。如果当前节点挂了，则在下一次查询或写入时，为缓存项查找另一个大于其hash值的缓存节点即可。如下图所示：每个缓存节点都存储在大于等于其hash值的缓存节点，如果这个阶段挂了，则寻找下一个节点。
![hash环](hash环.png)

但是出现了一个问题，如果节点分布在hash环上不均匀，则会造成数据倾斜的问题。为了解决个问题，通过引入虚拟节点，让节点在圆环上分布散开，到达节点均衡分布在hash环上的效果。[dubbo负载均衡](http://dubbo.apache.org/zh-cn/docs/source_code_guide/loadbalance.html)

### 源码分析ConsistentHashLoadBalance

```java
public class ConsistentHashLoadBalance extends AbstractLoadBalance {
    public static final String NAME = "consistenthash";

    public static final String HASH_NODES = "hash.nodes"; // hash 节点的名称

    public static final String HASH_ARGUMENTS = "hash.arguments"; // 进行hash的参数名

    private final ConcurrentMap<String, ConsistentHashSelector<?>> selectors = new ConcurrentHashMap<String, ConsistentHashSelector<?>>();

    @SuppressWarnings("unchecked")
    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        String methodName = RpcUtils.getMethodName(invocation); // ①
        String key = invokers.get(0).getUrl().getServiceKey() + "." + methodName; // ②
        int identityHashCode = System.identityHashCode(invokers); // ③
        ConsistentHashSelector<T> selector = (ConsistentHashSelector<T>) selectors.get(key); // ④
        if (selector == null || selector.identityHashCode != identityHashCode) {
            selectors.put(key, new ConsistentHashSelector<T>(invokers, methodName, identityHashCode));
            selector = (ConsistentHashSelector<T>) selectors.get(key);
        }
        return selector.select(invocation); // ⑤
    }
```
doSelect方法逻辑：①首先获取invoker的方法名；②生成selectors的key=接口的名称+方法名；③为invokers生成唯一的hashcode值；根据key获取对应的ConsistentHashSelector，如果没有生成一个；⑤执行ConsistentHashSelector的select方法，获取对应的invoker。

#### ConsistentHashSelector
ConsistentHashSelector是hash一致性负载均衡算法的核心实现。

```java
private static final class ConsistentHashSelector<T> {
    // 使用TreeMap存储invoker虚拟节点
    private final TreeMap<Long, Invoker<T>> virtualInvokers;

    private final int returnplicaNumber;

    private final int identityHashCode;

    private final int[] argumentIndex;

    ConsistentHashSelector(List<Invoker<T>> invokers, String methodName, int identityHashCode) {
        this.virtualInvokers = new TreeMap<Long, Invoker<T>>();
        this.identityHashCode = identityHashCode;
        URL url = invokers.get(0).getUrl();
        // 虚拟节点个数，默认160个
        this.replicaNumber = url.getMethodParameter(methodName, HASH_NODES, 160); // ①
        // 获取参与hash计算的参数下标值，默认对第一个参数进行hash运算
        String[] index = COMMA_SPLIT_PATTERN.split(url.getMethodParameter(methodName, HASH_ARGUMENTS, "0")); // ②
        argumentIndex = new int[index.length];
        for (int i = 0; i < index.length; i++) { // ③
            argumentIndex[i] = Integer.parseInt(index[i]);
        }
        for (Invoker<T> invoker : invokers) { // ④
            String address = invoker.getUrl().getAddress();
            for (int i = 0; i < replicaNumber / 4; i++) {
                // 对 address + i 进行md5运算，得到一个长度为16的字节数组
                byte[] digest = md5(address + i); // ⑤
                // 对digest部分字节进行4次hash计算，得到四个不同的long类型正整数
                for (int h = 0; h < 4; h++) { // ⑥
                    // h = 0 时，取 digest 中下标为 0 ~ 3 的4个字节进行位运算
                    // h = 1 时，取 digest 中下标为 4 ~ 7 的4个字节进行位运算
                    // h = 2, h = 3 时过程同上
                    long m = hash(digest, h); // ⑦
                    // 将 hash 到 invoker 的映射关系存储到 virtualInvokers 中，
                    // virtualInvokers 需要提供高效的查询操作，因此选用 TreeMap 作为存储结构
                    virtualInvokers.put(m, invoker);
                }
            }
        }
    }

    public Invoker<T> select(Invocation invocation) {
        String key = toKey(invocation.getArguments()); 
        byte[] digest = md5(key);
        return selectForKey(hash(digest, 0));
    }

    private String toKey(Object[] args) {
        StringBuilder buf = new StringBuilder();
        for (int i : argumentIndex) {
            if (i >= 0 && i < args.length) {
                buf.append(args[i]);
            }
        }
        return buf.toString();
    }

    private Invoker<T> selectForKey(long hash) {
        Map.Entry<Long, Invoker<T>> entry = virtualInvokers.ceilingEntry(hash);
        if (entry == null) {
            entry = virtualInvokers.firstEntry();
        }
        return entry.getValue();
    }

    private long hash(byte[] digest, int number) {
        return (((long) (digest[3 + number * 4] & 0xFF) << 24)
                | ((long) (digest[2 + number * 4] & 0xFF) << 16)
                | ((long) (digest[1 + number * 4] & 0xFF) << 8)
                | (digest[number * 4] & 0xFF))
                & 0xFFFFFFFFL;
    }

    private byte[] md5(String value) {
        MessageDigest md5;
        try {
            md5 = MessageDigest.getInstance("MD5");
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
        md5.reset();
        byte[] bytes = value.getBytes(StandardCharsets.UTF_8);
        md5.update(bytes);
        return md5.digest();
    }
}
```
先看ConsistentHashSelector的构造函数。
- virtualInvokers 是一个TreeMap，用于存储hash环上的虚拟节点。
- replicaNumber 虚拟节点个数
- identityHashCode 由invokers生成的唯一标识
- argumentIndex 存储方法的参数下标

①中从url中获取hash.nodes的配置，缺省配置为160个，赋值给replicaNumber；②获取配置进行hash的参数下标，缺省配置为第一个参数；③存储每个参数的下标，留作后用；④循环每个invoker，填充virtualInvokers；⑤根据address经过md5方法生成长度为16的byte数组；⑥循环4次，每次取byte数组的4个数进行hash算法，得到一个long的值；⑦以第⑥生成的值为key，invoker为value，存入treeMap中。

再看ConsistentHashSelector的select方法。
首先会通过配置的需要hash的参数的下标生成一个key（如果多个，则拼接），通过md5方法生成生成长度16的byte数组，然后取前四个元素进行hash，等到一个long类型的值。再调用selectForKey方法获取invoker。selectForKey方法，首先根据hash值从TreeMap中找到第一个比该hash值大的key，返回这个key的value，如果没找到，则返回treemap的第一个元素。

整个流程，需要理解的是，任何参数经过hash方法之后，得到的值一定是处于一个范围之内的，即会落在hash环上。然后ConsistentHashSelector初始化时，将invoker的虚拟节点都散列在hash环上，所以通过参数下标进行hash方法时，得到的值也会落在hash环的某个地方，然后使用TreeMap的特性，顺时针寻找第一个大于该值的节点即可。

## 基于加权轮询算法的 RoundRobinLoadBalance
先了解什么是加权轮询。轮询是指将请求轮流分配给每台服务器，比如：有A、B、C三台服务器，将第一个请求分配给A，将第二个请求分配给B，第三个请求分配给C，第四个请求再给A。这种过程就叫轮询。轮询是一种无状态的负载均衡算法，适用于每台服务器性能相近的场景。但是现实场景下，每台服务器服务性能不能保证均相等，如果把等量的请求分配给性能差的机器，这显然是不合理的。因此，需要对轮询过程进行加权，以调控每台服务的负载。经过加权后，每台服务器能够得到的请求数比例，接近或等于他们的权重比。

```java
public class RoundRobinLoadBalance extends AbstractLoadBalance {
    public static final String NAME = "roundrobin";
    // 清除过期数据的周期
    private static final int RECYCLE_PERIOD = 60000;
    // 封装每个invoker的权重
    protected static class WeightedRoundRobin {
        private int weight; // invoker配置的权重值
        private AtomicLong current = new AtomicLong(0); // 并发场景下当前invoker会被同时选中，表示该节点被所有线程选中的权重之和
        private long lastUpdate; // 最后一次更新的时间，用户后续缓存超时的判断
        public int getWeight() {
            return weight;
        }
        public void setWeight(int weight) {
            this.weight = weight;
            current.set(0);
        }
        public long increaseCurrent() {
            return current.addAndGet(weight);
        }
        public void sel(int total) {
            current.addAndGet(-1 * total);
        }
        public long getLastUpdate() {
            return lastUpdate;
        }
        public void setLastUpdate(long lastUpdate) {
            this.lastUpdate = lastUpdate;
        }
    }

    private ConcurrentMap<String, ConcurrentMap<String, WeightedRoundRobin>> methodWeightMap = new ConcurrentHashMap<String, ConcurrentMap<String, WeightedRoundRobin>>(); // 存储method权重容器，key=接口名 + "." + 方法名
    private AtomicBoolean updateLock = new AtomicBoolean(); // 更新锁

    // 算法实现
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName(); // 构建methodWeightMap的key，接口名+方法名
        ConcurrentMap<String, WeightedRoundRobin> map = methodWeightMap.get(key);  // 存储每个invoker节点的权重
        if (map == null) {
            methodWeightMap.putIfAbsent(key, new ConcurrentHashMap<String, WeightedRoundRobin>());
            map = methodWeightMap.get(key);
        }
        int totalWeight = 0; // 所有节点的权重之和
        long maxCurrent = Long.MIN_VALUE;
        long now = System.currentTimeMillis(); // 记录当前更新时间
        Invoker<T> selectedInvoker = null; // 最终选的invoker
        WeightedRoundRobin selectedWRR = null; // 被选中的invoker的映射权重对象
        for (Invoker<T> invoker : invokers) { // 循环每个invoker
            String identifyString = invoker.getUrl().toIdentityString(); // 根据invoker的url生成存储每个invoker节点的权重map的key
            WeightedRoundRobin weightedRoundRobin = map.get(identifyString);
            int weight = getWeight(invoker, invocation); // 计算出当前invoker的权重

            // 存入map中       
            if (weightedRoundRobin == null) {
                weightedRoundRobin = new WeightedRoundRobin();
                weightedRoundRobin.setWeight(weight);
                map.putIfAbsent(identifyString, weightedRoundRobin);
            }
            // 如果当前invoker的权重发生变化，更新map中的当前invoker的权重值
            if (weight != weightedRoundRobin.getWeight()) {
                //weight changed
                weightedRoundRobin.setWeight(weight);
            }
            // cur=cur+weight，每次都对cur进行累加
            long cur = weightedRoundRobin.increaseCurrent();
            // 设置当前invoker权重更新时间  
            weightedRoundRobin.setLastUpdate(now);
            if (cur > maxCurrent) { // 控制cur值在Long.MIN_VALUE之内
                maxCurrent = cur;
                selectedInvoker = invoker; // 将当前invoker赋值给selectedInvoker
                selectedWRR = weightedRoundRobin; // 将当前invoker的weightedRoundRobin赋值给selectedWRR
            }
            totalWeight += weight; // totalWeight 累加
        }
        // 更新缓存数据，获取updataLock锁并且invoker的size不等于map的size（说明数据不一致）
        if (!updateLock.get() && invokers.size() != map.size()) {
            if (updateLock.compareAndSet(false, true)) { // 通过CAS操作，获取锁
                try {
                    // copy -> modify -> update reference
                    ConcurrentMap<String, WeightedRoundRobin> newMap = new ConcurrentHashMap<String, WeightedRoundRobin>();
                    newMap.putAll(map); // 定于newMap为最新的数据
                    Iterator<Entry<String, WeightedRoundRobin>> it = newMap.entrySet().iterator();
                    while (it.hasNext()) { // 循环map里的每个元素
                        Entry<String, WeightedRoundRobin> item = it.next();
                        // 如果元素的最后更新时间和当前时间差大于默认的过期时间（60s），就会被已移除
                        if (now - item.getValue().getLastUpdate() > RECYCLE_PERIOD) {
                            it.remove();
                        }
                    }
                    // 更新methodWeight里的对应key的value引用（更新）
                    methodWeightMap.put(key, newMap);
                } finally {
                    // 释放锁，设置为false，确保下一个线程可以获取
                    updateLock.set(false);
                }
            }
        }
        // 最终要选择的invoker，将选中的invoker的current值减掉totalWeight
        if (selectedInvoker != null) {
            selectedWRR.sel(totalWeight);
            return selectedInvoker;
        }
        // should not happen here
        return invokers.get(0);
    }
}
```
以上对dubbo的RoundRobinLoadBalance实现做了个初步的解释，基本上都写上了注释，每行代码的含义，但是要真正理解RoundRobin还是要知道轮训负载均衡的本质。
