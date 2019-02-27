---
title: rocketmq之producer消息发送者
date: 2019-01-03 18:00:33
categories: 消息队列
tags:
- RocketMQ
- producer
---

rocketmq之消息发送者 producer 源码解析.

<!--more-->

## 前景回顾

**{% post_link rocketmq之broker源码学习 RocketMQ学习Part1之broker %}**

## producer 概述

RocketMQ 发送消息有三种模式：
- 同步（synchronous）
- 异步 （asynchronous）
- 单向传输（one-way transmission）

producer 支持分布式部署。分布式的producers通过多种负载均衡模式向 broker 集群发送消息。发送过程支持快速故障并具有低延迟。

当发送消息，将会得到 `SendResult` 包含 `SendStatus`。 首先，假设消息的 `isWaitStoreMsgOk=true`(默认true)。
首先，我们假设Message的isWaitStoreMsgOK = true（默认为true）。如果没有，如果没有抛出异常，我们将始终获得SEND_OK。
以下是有关每种状态的说明：

**FLUSH_DISK_TIMEOUT**

如果 broker 设置 `MessageStoreConfig` 的 `FlushDiskType=SYNC_FLUSH`（默认ASYNC_FLUSH）,并且broker没有
在 `MessageStoreConfig` 的 `syncFlushTimeout`（默认5s） 时间内完成 flushing disk，将会得到这个状态。

**FLUSH_SLAVE_TIMEOUT**

如果 broker的角色是 `SYNC_MASTER`(默认是ASYNC_MASTER)，slave broker没有完成在`MessageStoreConfig` 的 `syncFlushTimeout`（默认5s） 时间内完成
master的同步，将会得到这个状态。

**SLAVE_NOT_AVAILABLE**

如果broker的角色是 `SYNC_MASTER`（默认是 ASYNC_MASTER），但是没有配置 slave broker，将会得到这个状态。

**SEND_OK**

`SEND_OK` 不意味着是可用的。为了确保没有消息丢失，应该启用 `SYNC_MASTER` 或者 `SYNC_FLUSH`。

## 一个简单的Demo

```java
public class SyncProducer {

    public static void main(String[] args) throws Exception {

        // 实例化producer，并给出group name
        DefaultMQProducer producer = new DefaultMQProducer("sync_group");

        // 指定name server 地址
        producer.setNamesrvAddr("localhost:9876");
//      producer.setRetryTimesWhenSendFailed(3);
        // 启动实例
        producer.start();

        JSONObject msgBody = new JSONObject();
        msgBody.put("id", 7);
        msgBody.put("name", "ljming");
        msgBody.put("learning", "rocketmq");

        Message message = new Message("simple_topic", "*",
                msgBody.toJSONString().getBytes(RemotingHelper.DEFAULT_CHARSET));

        // 调用send方法，将消息传送到一个消息服务器（broker）
        SendResult result = producer.send(message);
        System.out.printf("%s%n", result);
        producer.shutdown();
    }
}
```

下面基于同步发送模式解析producer从启动到发送消息的过程。

## prodcuer.start()

producer启动过程如下图所示:

![启动过程](producer启动.jpg)

- 第①步对producer的groupName进行校验，就是一些合法性的校验。
- 第②步通过MQClientManager类创建了MQClientInstance，表示一个消息队列客户端实例。
- 第③步创建的客户端实例放入缓存，一个以producerGroupName为Key，MQClientInstance为value的名为producerTable的map中。
- 第④步调用MQClientInstance对象的start方法，启动消息队列客户端实例。
- 第⑤步调用MQClientAPIImpl对象的start方法，start方法中启动了RemotingClient。MQClientAPIImpl类中主要是对客户端和服务端通信内容进行封装，而客户端给服务端发送请求具体实现在RemotingClient中。
- 第⑥步启动了多个定时任务。
- 第⑦步启动拉取消息的服务--PullMessageService。
- 第⑧步启动负载均衡服务--RebalanceService，主要是对Consumer客户端的负载均衡。
- 第⑨步，producer启动完成，向所有的broker发送心跳。

至此producer实例启动完成，有个疑问，为什么producer的启动，附带还启动了两个和consumer相关的服务呢？（PullMessageService和RebalanceService）

## producer 发送消息

![producer发送消息过程](producer发送消息.jpg)

整个发送过程，核心的两个步骤是②和③。

### ②获取TopicPublishInfo
首先看看TopicPublishInfo有什么信息。

TopicPublicInfo的属性：
```
 - boolean orderTopic = false; // 是否为顺序的
 - boolean haveTopicRouterInfo = false; // 是否可用
 - List<MessageQueue> messageQueueList = new ArrayList<MessageQueue>(); // MessageQueue 集合
 - ThreadLocalIndex sendWhichQueue = new ThreadLocalIndex(); // 当前要发送到那个MessageQueue，返回下标值
 - TopicRouteData topicRouteData; // 路由数据

    MessageQueue的属性：
    - String topic; // 所属topic
    - String brokerName; // 所属broker
    - int queueId; // 队列编号
    
    TopicRouteData的属性：
    - String orderTopicConf; // 顺序topic的配置
    - List<QueueData> queueDatas; // 队列数据
    - List<BrokerData> brokerDatas; // broker 数据
    - HashMap<String, List<String>> filterServerTable; // 过滤服务缓存，key为brokerAddr，value为过滤的server集合

        QueueData的属性：
        - String brokerName; // broker
        - int readQueueNums; // 读队列个数 
        - int writeQueueNums; // 写队列个数
        - int perm; // 权限
        - int topicSynFlag; // 同步标记

        BrokerData的属性：
        - String cluster; // 所属集群
        - String brokerName; // broker名称
        - HashMap<Long, String> brokerAddrs;  // broker 地址映射,key为brokerId，value为broker address
```
上面的类结构说明，TopicPublishInfo以topic维度描述，一个topic可以分布在多个broker上，一个topic具有多个MessageQueue。一个队列元数据（QueueData）基于Broker描述，描述信息包括所在的broker、读写队列个数、权限、同步标识。

源码分析`tryToFindTopicPublishInfo(final String topic)`：获取topic的路由信息，决定消息发送到具体的broker的MessageQueue上。
```java
private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
    TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);  //①
    if (null == topicPublishInfo || !topicPublishInfo.ok()) {
        this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic); // ②
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
    }

    if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {  // ③
        return topicPublishInfo;
    } else {
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer); // ④
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
        return topicPublishInfo;
    }
}
```
①处从缓存topicPublishInfoTable中获取，首次获取肯定为空。然后就会new一个TopicPublishInfo对象，放入缓存，②然后更新NameSrv的TopicPublishInfo。如果②中获取的TopicPublishInfo可用，直接返回，否者进入④逻辑。②和④的区别在于前者是通过具体的topic去nameSrv获取TopicPublishInfo，后者是通过默认的topic去nameSrv获取TopicPublishInfo。也可以说明，在tryToFindTopicPublishInfo实现中，首先使用指定的topic查找，如果没有找到，使用默认的topic查找路由信息。

源码分析`updateTopicRouteInfoFromNameServer(final String topic, boolean isDefault, DefaultMQProducer defaultMQProducer)`
```java
public boolean updateTopicRouteInfoFromNameServer(final String topic, boolean isDefault, DefaultMQProducer defaultMQProducer) {
    try {
        if (this.lockNamesrv.tryLock(LOCK_TIMEOUT_MILLIS, TimeUnit.MILLISECONDS)) { // ①
            try {
                TopicRouteData topicRouteData;
                if (isDefault && defaultMQProducer != null) {
                    topicRouteData = this.mQClientAPIImpl.getDefaultTopicRouteInfoFromNameServer(defaultMQProducer.getCreateTopicKey(),
                        1000 * 3); // ②
                    if (topicRouteData != null) {
                        for (QueueData data : topicRouteData.getQueueDatas()) {
                            int queueNums = Math.min(defaultMQProducer.getDefaultTopicQueueNums(), data.getReadQueueNums());
                            data.setReadQueueNums(queueNums);
                            data.setWriteQueueNums(queueNums);
                        }
                    }
                } else {
                        topicRouteData = this.mQClientAPIImpl.getTopicRouteInfoFromNameServer(topic, 1000 * 3); // ③
                }
                if (topicRouteData != null) {
                    TopicRouteData old = this.topicRouteTable.get(topic);
                    boolean changed = topicRouteDataIsChange(old, topicRouteData); // ④
                    if (!changed) {
                        changed = this.isNeedUpdateTopicRouteInfo(topic); // ⑤
                    } else {
                        log.info("the topic[{}] route info changed, old[{}] ,new[{}]", topic, old, topicRouteData);
                    }

                    if (changed) { // ⑥
                        TopicRouteData cloneTopicRouteData = topicRouteData.cloneTopicRouteData();

                        for (BrokerData bd : topicRouteData.getBrokerDatas()) {
                            this.brokerAddrTable.put(bd.getBrokerName(), bd.getBrokerAddrs());
                        }

                        // Update Pub info ⑦
                        {
                            TopicPublishInfo publishInfo = topicRouteData2TopicPublishInfo(topic, topicRouteData);
                            publishInfo.setHaveTopicRouterInfo(true);
                            Iterator<Entry<String, MQProducerInner>> it = this.producerTable.entrySet().iterator();
                            while (it.hasNext()) {
                                Entry<String, MQProducerInner> entry = it.next();
                                MQProducerInner impl = entry.getValue();
                                if (impl != null) {
                                    impl.updateTopicPublishInfo(topic, publishInfo);
                                }
                            }
                        }

                        // Update sub info ⑧
                        {
                            Set<MessageQueue> subscribeInfo = topicRouteData2TopicSubscribeInfo(topic, topicRouteData);
                            Iterator<Entry<String, MQConsumerInner>> it = this.consumerTable.entrySet().iterator();
                            while (it.hasNext()) {
                                Entry<String, MQConsumerInner> entry = it.next();
                                MQConsumerInner impl = entry.getValue();
                                if (impl != null) {
                                    impl.updateTopicSubscribeInfo(topic, subscribeInfo);
                                }
                            }
                        }
                        log.info("topicRouteTable.put. Topic = {}, TopicRouteData[{}]", topic, cloneTopicRouteData);
                        this.topicRouteTable.put(topic, cloneTopicRouteData);
                        return true;
                    }
                } else {
                    log.warn("updateTopicRouteInfoFromNameServer, getTopicRouteInfoFromNameServer return null, Topic: {}", topic);
                }
            } catch (Exception e) {
                if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX) && !topic.equals(MixAll.AUTO_CREATE_TOPIC_KEY_TOPIC)) {
                    log.warn("updateTopicRouteInfoFromNameServer Exception", e);
                }
            } finally {
                this.lockNamesrv.unlock();
            }
        } else {
            log.warn("updateTopicRouteInfoFromNameServer tryLock timeout {}ms", LOCK_TIMEOUT_MILLIS);
        }
    } catch (InterruptedException e) {
        log.warn("updateTopicRouteInfoFromNameServer Exception", e);
    }

    return false;
}
```
①处为了避免重复从NameSrv获取配置信息，是有个ReentranLock，超时时间为3000ms。②和③的区别在于前者通过默认的topic查找路由信息，后者通过指定的topic查找路由信息。④和⑤为了确定本地缓存的topic路由信息和nameSrv获取到的信息是否发生了变化。⑥如果有变化，则进行⑦和⑧的操作，前者对topic发布的信息进行本地缓存更新，后者对topic订阅的信息进行本地缓存更新。

### ③选择消息要发送到哪个MessageQueue
选择要发送的MessageQueue之前，会根据发送消息模式定义消息发送失败尝试次数，同步模式默认进行三次发送。其它两个模式只发送一次，不进行重试。下面对获取要发送到的MessageQueue源码进行分析。

源码分析`selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName)`

具体实现是调用了MQFaultStrategy这个类的`selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName)`方法。

```java
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
    if (this.sendLatencyFaultEnable) { // ①
        try {
            int index = tpInfo.getSendWhichQueue().getAndIncrement();
            for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {  // ③
                int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();
                if (pos < 0)
                    pos = 0;
                MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
                if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) { // ④
                    if (null == lastBrokerName ||  mq.getBrokerName().equals(lastBrokerName))
                        return mq;
                }
            }

            final String notBestBroker = latencyFaultTolerance.pickOneAtLeast(); // ⑤
            int writeQueueNums = tpInfo.getQueueIdByBroker(notBestBroker);
            if (writeQueueNums > 0) {
                final MessageQueue mq = tpInfo.selectOneMessageQueue();
                if (notBestBroker != null) {
                    mq.setBrokerName(notBestBroker);
                    mq.setQueueId(tpInfo.getSendWhichQueue().getAndIncrement() % writeQueueNums);
                }
                return mq;
            } else {
                latencyFaultTolerance.remove(notBestBroker);
            }
        } catch (Exception e) {
            log.error("Error occurred when selecting message queue", e);
        }

        return tpInfo.selectOneMessageQueue();
    }

    return tpInfo.selectOneMessageQueue(lastBrokerName); // ②
}
```    

①判断是否启用发送延迟故障，否的话直接进行②逻辑。②的实现，直接从messageQueueList中选取一个MessageQueue返回，也不关心当前Broker是否可用（比如当前时间Broker挂了）。

如果①启动了发送延迟故障，那么就会考虑broker的可用性。这里面的if分支，首先从ThreadLocal中获取本次要发送的MessageQueue的下标，然后③进行循环MessageQueueList，这么做的理由是确保所选的messageQueue的broker是可用的，如果本次不可用，循环出最终可用的那个messageQueue。

④进行判断当前MessageQueue的Broker是否可用，需要理解发送消息延迟机制。

**延迟策略**
```java
private boolean sendLatencyFaultEnable = false;
private long[] latencyMax = {50L, 100L, 550L, 1000L, 2000L, 3000L, 15000L}; // 最大延迟时间
private long[] notAvailableDuration = {0L, 0L, 30000L, 60000L, 120000L, 180000L, 600000L}; // 不可用时间
// 更新缓存中不可用broker
public void updateFaultItem(final String brokerName, final long currentLatency, boolean isolation) {
    if (this.sendLatencyFaultEnable) {
        long duration = computeNotAvailableDuration(isolation ? 30000 : currentLatency);
        this.latencyFaultTolerance.updateFaultItem(brokerName, currentLatency, duration);
    }
}
// 根据发送消息延迟时间，计算出broker不可用时长
private long computeNotAvailableDuration(final long currentLatency) {
    for (int i = latencyMax.length - 1; i >= 0; i--) {
        if (currentLatency >= latencyMax[i])
            return this.notAvailableDuration[i];
    }
    return 0;
}
```
在计算出broker不可用时长后，放入缓存中的FaultItem设置的startTimestamp=当前时间+notAvailableDuration。在判断当前broker是否可用的条件就是当前时间是否大于startTimestamp。

⑤表示当前所有的broker都处于不可用状态，那么这里的处理逻辑是从保存不可用broker信息的缓存中，根据时间排序，startTimestamp最小的那个将会被选择。这是获取到的broker也不是一定不可用，也许在这个期间，broker被运维修复了。

以上所有步骤就是选择消息将要发送到那个MessageQueue的全部过程。

### ④发送消息的核心实现

经过②③的关键两步，确定了当前这条消息要发送到那个broker上。接下来就是消息发送的具体实现。Producer和broker基于长连接的模式，将消息发送到broker上，Producer给broker发送消息的请求码为`RequestCode.SEND_MESSAGE=10`。这里需要说明的是：SYNC模式需要等待broker接收消息，存储完成，如果Boker是SYNC_MASTER，还需要等待主从完成，才会返回消息。ASYNC模式不会等待broker返回结果，但是通过CallBack自我实现消息成功或失败通知。Oneway模式只管发送消息，不管broker接收成功与否（有点类似UDP的请求）。

### ⑤进行发送延迟故障更新

消息在发送过程，会记录发送开始时间以及结束时间，两者的差值就是当前消息发送的延迟时间（②③④过程花费的时间）。然后会根据延迟时间，算出broker不可用时间。

![延迟策略](延迟策略.jpg)

