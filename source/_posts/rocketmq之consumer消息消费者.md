---
title: rocketmq之consumer消息消费者
date: 2019-01-06 22:38:33
categories: 消息队列
tags:
- RocketMQ
- consumer
---

RocketMQ 之消息消费者 consumer 源码解析。

<!--more-->

## 前景回顾

**{% post_link rocketmq之producer消息发送者 RocketMQ学习Part2之Producer %}**
**{% post_link rocketmq之broker源码学习 RocketMQ学习Part1之Broker %}**

## 概述

消费者从broker获取消息并将其提供给应用程序。对于应用程序而言,提供了两种consumer:

- pullConsumer
    pull consumer 从broker主动拉取消息，一旦拉去了批量的消息，应用程序发起消费程序。

- pushConsumer
    push consumer，反过来讲，封装消息提取，消费进度并维护其它内部工作，给出一个 callback 接口，让用户实现，当消息到达时执行。

## 简单的例子

### pullConsumer

```java
public class SimplePullConsumer implements InitializingBean {
    @Value("${mq.address}")
    private String nameSrv;

    private final Map<MessageQueue, Long> OFFSET_TABLE = new HashMap<>(); // 自己维护offset

    private DefaultMQPullConsumer simplePullConsumer = null;

    @Override
    public void afterPropertiesSet() {
        System.out.println("init simpleConsumer");
        startConsumer();
    }

    private void startConsumer() {
        try {
            simplePullConsumer = new DefaultMQPullConsumer("simpleConsumer");
            if (StringUtils.isNotEmpty(nameSrv)) {
                simplePullConsumer.setNamesrvAddr(nameSrv);
            }
            Set<MessageQueue> messageQueueSet = simplePullConsumer.fetchSubscribeMessageQueues(SimpleTopic.SIMPLEPULLTOPIC.getName());
            messageQueueSet.forEach(mq -> {
                try {
                    long offset = simplePullConsumer.fetchConsumeOffset(mq, true);
                    System.out.println("Consumer from the queue : " + mq);
                    while (true) {
                        PullResult pr = simplePullConsumer.pullBlockIfNotFound(mq, null, getMessageQueueOffset(mq), 32);
                        System.out.println("pr: " + pr);
                        putMessageQueueOffset(mq, pr.getNextBeginOffset());
                        switch (pr.getPullStatus()) {
                            case FOUND:
                                break;
                            case NO_NEW_MSG:
                                break;
                            case NO_MATCHED_MSG:
                                break;
                            case OFFSET_ILLEGAL:
                                break;
                            default:
                                break;
                        }
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            });
            simplePullConsumer.start();
            System.out.println("consumer started");
        } catch (Exception e) {
            System.err.println("consumer start error: " + e);
        }
    }

    private long getMessageQueueOffset(MessageQueue mq) {
        Long offset = OFFSET_TABLE.get(mq);
        if (offset != null) {
            return offset;
        }
        return 0;
    }
    private void putMessageQueueOffset(MessageQueue mq, long offset) {
        OFFSET_TABLE.put(mq, offset);
    }
}
```

### pushConsumer

```java
public class Consumer {

    public static void main(String[] args) throws InterruptedException, MQClientException {

        // Instantiate with specified consumer group name.
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("simpleConsumer");
         
        // Specify name server addresses.
        consumer.setNamesrvAddr("localhost:9876");
        
        // Subscribe one more more topics to consume.
        consumer.subscribe("TopicTest", "*");
        // Register callback to execute on arrival of messages fetched from brokers.
        consumer.registerMessageListener(new MessageListenerConcurrently() {

            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
                ConsumeConcurrentlyContext context) {
                System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msgs);
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });

        //Launch the consumer instance.
        consumer.start();

        System.out.printf("Consumer Started.%n");
    }
}
```

## consumer 启动

consumer启动的过程如下图：

![启动过程](consumer启动过程.jpg)

```java
public synchronized void start() throws MQClientException {
    switch (this.serviceState) {
        case CREATE_JUST:
            log.info("the consumer [{}] start beginning. messageModel={}, isUnitMode={}", this.defaultMQPushConsumer.getConsumerGroup(),
                this.defaultMQPushConsumer.getMessageModel(), this.defaultMQPushConsumer.isUnitMode());
            this.serviceState = ServiceState.START_FAILED;

            this.checkConfig(); // ①

            this.copySubscription(); // ②

            if (this.defaultMQPushConsumer.getMessageModel() == MessageModel.CLUSTERING) {
                this.defaultMQPushConsumer.changeInstanceNameToPID();
            }
            this.mQClientFactory = MQClientManager.getInstance().getAndCreateMQClientInstance(this.defaultMQPushConsumer, this.rpcHook); // ③

            this.rebalanceImpl.setConsumerGroup(this.defaultMQPushConsumer.getConsumerGroup());
            this.rebalanceImpl.setMessageModel(this.defaultMQPushConsumer.getMessageModel());
            this.rebalanceImpl.setAllocateMessageQueueStrategy(this.defaultMQPushConsumer.getAllocateMessageQueueStrategy());
            this.rebalanceImpl.setmQClientFactory(this.mQClientFactory);

            this.pullAPIWrapper = new PullAPIWrapper(mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup(), isUnitMode()); // ④
            this.pullAPIWrapper.registerFilterMessageHook(filterMessageHookList);
            if (this.defaultMQPushConsumer.getOffsetStore() != null) {
                this.offsetStore = this.defaultMQPushConsumer.getOffsetStore();
            } else {
                switch (this.defaultMQPushConsumer.getMessageModel()) {
                    case BROADCASTING:
                        this.offsetStore = new LocalFileOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                        break;
                    case CLUSTERING:
                        this.offsetStore = new RemoteBrokerOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                        break;
                    default:
                        break;
                }
                this.defaultMQPushConsumer.setOffsetStore(this.offsetStore);
            }
            this.offsetStore.load(); // ⑤
            if (this.getMessageListenerInner() instanceof MessageListenerOrderly) {
                this.consumeOrderly = true;
                this.consumeMessageService =
                    new ConsumeMessageOrderlyService(this, (MessageListenerOrderly) this.getMessageListenerInner());
            } else if (this.getMessageListenerInner() instanceof MessageListenerConcurrently) {
                this.consumeOrderly = false;
                this.consumeMessageService =
                    new ConsumeMessageConcurrentlyService(this, (MessageListenerConcurrently) this.getMessageListenerInner());
            } 
            this.consumeMessageService.start(); // ⑥
            boolean registerOK = mQClientFactory.registerConsumer(this.defaultMQPushConsumer.getConsumerGroup(), this); // ⑦
            if (!registerOK) {
                this.serviceState = ServiceState.CREATE_JUST;
                this.consumeMessageService.shutdown();
                throw new MQClientException("The consumer group[" + this.defaultMQPushConsumer.getConsumerGroup()
                    + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                    null);
            }

            mQClientFactory.start(); // ⑧
            log.info("the consumer [{}] start OK.", this.defaultMQPushConsumer.getConsumerGroup());
            this.serviceState = ServiceState.RUNNING;
            break;
        case RUNNING:
        case START_FAILED:
        case SHUTDOWN_ALREADY:
            throw new MQClientException("The PushConsumer service state not OK, maybe started once, "
                + this.serviceState
                + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                null);
        default:
            break;
    }

    this.updateTopicSubscribeInfoWhenSubscriptionChanged(); // ⑨
    this.mQClientFactory.checkClientInBroker(); // ⑩
    this.mQClientFactory.sendHeartbeatToAllBrokerWithLock(); // ⑪
    this.mQClientFactory.rebalanceImmediately(); // ⑫
}
```
①处对consumer配置信息进行检查。
②创建了一个重试主题，命名格式为%RETRY%+consumerGroup，并放入负载均衡服务的订阅。
③创建consumer客户端实例，通过MQClientManger类，和producer一样。
④创建拉取消息的api包装类，在后续的消息拉取中分析。
⑤根据不同的消费模式，创建不同的offsetStore对象，并加载offset信息到内存。BROADCASTING模式则使用本地存储LocalFileOffsetStore，CLUSTERING模式使用远程存储RemoteBrokerOffsetStore。
⑥根据messageListener的类型，创建消息消费服务并启动该服务。MessageListenerOrderly对应`ConsumeMessageOrderlyService`，MessageListenerConcurrently对应`ConsumeMessageConcurrentlyService`。前者为顺序消费监听器，后者为并行消费监听器。
⑦将当前consumer放入到内存，consumerGroup为key，consumer实例为value的consumerTable。
⑧consumer实例启动，有两个关键点，启动了PullMessageService和RebalanceService这两个服务。
⑨从NameSrv更新consumer订阅topic路由信息。
⑩检查broker状态
⑪发送心跳到broker
⑫执行负载均衡

以上所有步骤就是consumer启动全过程。

全都是文字的分享是没有生命的，来张图😂：

![启动过程](consumer链路.jpg)

## consumer消息消费

这里基于Push模式分析consumer消费消息的过程。consumer启动的时候，启动了一个PullMessageService服务，这个服务就是用来向broker发起消息拉取请求的。这里大家都会有个疑问，不是说是Push模式么，为什么是使用PullMessageService（拉取消息服务）？其实在RocketMQ不管是Push和Pull模式，都是使用这个服务向broker获取消息。Push模式基于长轮询的模式向broker获取消息，就是当consumer向broker发起消息拉取请求，当broker接收到这个请求后，获取检查是否有新消息可以返回，如果没有，broker不会立刻返回结果，而是把这个请求挂起，等待5s，如果5s后还是没有新消息，然后返回结果。如果在请求挂起的时间内，broker接收到了新的消息，broker会立马唤醒这个请求，不用等待5s，然后返回结果。这个策略在后面详细分析。

### PullMessageService: 消息拉取服务

#### 第一个PullRequest哪里来？
```java
public void run() {
    log.info(this.getServiceName() + " service started");

    while (!this.isStopped()) {
        try {
            PullRequest pullRequest = this.pullRequestQueue.take(); // ①
            this.pullMessage(pullRequest); // ②
        } catch (InterruptedException ignored) {
        } catch (Exception e) {
            log.error("Pull Message Service Run Method exception", e);
        }
    }

    log.info(this.getServiceName() + " service end");
}
```
PullMessageService的run方法实现，只要服务处于运行状态，就会从pullRequestQueue中获取PullRequest，pullRequestQueue的定义：`LinkedBlockingQueue<PullRequest> pullRequestQueue = new LinkedBlockingQueue<PullRequest>();`。pullRequestQueue是一个阻塞队列，只有当pullRequestQueue.take()获取到数据时才会返回，否者当前线程就会一直阻塞，直到拿到数据。然后执行②，进行消息拉取。

但是当PullMessageService启动时，pullRequestQueue是空链表，也就是说this.pullRequestQueue.take()是拿不到数据，那当前线程岂不是一直都被阻塞了，②过程就一直无法执行了。

这里就需要引入RebalanceService服务来讲。consumer服务启动的之后启动了RebalanceService，并且在consumer启动最后，立马执行了负载均衡的操作：`this.mQClientFactory.rebalanceImmediately()`，这个方法唤醒了RebalanceService服务：
```java
public void rebalanceImmediately() {
    this.rebalanceService.wakeup();
}
```
RebalanceService被唤醒之后，执行了`doRebalance()`操作。
```java
// RebalanceService.java
public void run() {
    log.info(this.getServiceName() + " service started");

    while (!this.isStopped()) {
        this.waitForRunning(waitInterval);
        this.mqClientFactory.doRebalance(); // ①
    }

    log.info(this.getServiceName() + " service end");
}
// MQClientInstance.java
public void doRebalance() {
    for (Map.Entry<String, MQConsumerInner> entry : this.consumerTable.entrySet()) { // ②
        MQConsumerInner impl = entry.getValue();
        if (impl != null) {
            try {
                impl.doRebalance(); 
            } catch (Throwable e) {
                log.error("doRebalance exception", e);
            }
        }
    }
}
// RebalanceImpl.java
public void doRebalance(final boolean isOrder) {
    Map<String, SubscriptionData> subTable = this.getSubscriptionInner();
    if (subTable != null) {
        for (final Map.Entry<String, SubscriptionData> entry : subTable.entrySet()) { 
            final String topic = entry.getKey();
            try {
                this.rebalanceByTopic(topic, isOrder); // ③
            } catch (Throwable e) {
                if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                    log.warn("rebalanceByTopic Exception", e);
                }
            }
        }
    }
    this.truncateMessageQueueNotMyTopic();
}
```
①负载均衡服务被唤醒，执行负载均衡操作。
②循环consumerTable里的所有consumer实例执行负载均衡。
③循环consumer订阅的topic信息，根据每个topic进行负载均衡操作。

在`rebalanceByTopic(String topic, boolean isOrder)`实现中，会调用`updateProcessQueueTableInRebalance(String topic, Set<MessageQueue> mqSet, boolean isOrder)`方法。这个方法实现内部，对ProcessQueueTable进行处理，
ProcessQueueTable是一个以MessageQueue为key，ProcessQueue为value的Map。ProcessQueue是队列消费快照，每个MessageQueue对应一个ProcessQueue。对processQueueTable处理的结果是，根据key和value生成PullRequest的集合，最后在对PullRequestList进行分发。

```java
public void dispatchPullRequest(List<PullRequest> pullRequestList) {
    for (PullRequest pullRequest : pullRequestList) {
        this.defaultMQPushConsumerImpl.executePullRequestImmediately(pullRequest);
        log.info("doRebalance, {}, add a new pull request {}", consumerGroup, pullRequest);
    }
}
```
在`this.defaultMQPushConsumerImpl.executePullRequestImmediately(pullRequest)`实现内部，直接调用了PullMessageService的executePullRequestImmediately方法，这个方法只是把PullRequest对象放到了阻塞队列`pullRequestQueue`中。讲到这里就衔接上了上面讲的，PullMessageService从阻塞队列`pullRequestQueue`取出PullRequest对象，然后进行消息拉取操作。

#### 执行消息拉取
上面讲到PullMessageService从阻塞队列成功获取到了PullRequest对象，接下来就是执行拉取消息的操作。

首先会通过consumerGroupName获取Consumer对象，从consumerTable缓存获取。拉取消息的具体实现在DefaultMQPushConsumerImpl类中，下面就源码分析拉取消息的实现`DefaultMQPushConsumerImpl.pullMessage(PullRequest pullRequest)`:

```java
public void pullMessage(final PullRequest pullRequest) {
    final ProcessQueue processQueue = pullRequest.getProcessQueue(); // ①
    if (processQueue.isDropped()) {
        log.info("the pull request[{}] is dropped.", pullRequest.toString());
        return;
    }

    pullRequest.getProcessQueue().setLastPullTimestamp(System.currentTimeMillis()); // ②

    try {
        this.makeSureStateOK(); // ③
    } catch (MQClientException e) {
        log.warn("pullMessage exception, consumer state not ok", e);
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION); // ④
        return;
    }

    if (this.isPause()) {
        log.warn("consumer was paused, execute pull request later. instanceName={}, group={}", this.defaultMQPushConsumer.getInstanceName(), this.defaultMQPushConsumer.getConsumerGroup());
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_SUSPEND);  // ⑤
        return;
    }

    long cachedMessageCount = processQueue.getMsgCount().get(); // ⑥
    long cachedMessageSizeInMiB = processQueue.getMsgSize().get() / (1024 * 1024); // ⑦

    if (cachedMessageCount > this.defaultMQPushConsumer.getPullThresholdForQueue()) {
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL); // ⑧
        if ((queueFlowControlTimes++ % 1000) == 0) {
            log.warn(
                "the cached message count exceeds the threshold {}, so do flow control, minOffset={}, maxOffset={}, count={}, size={} MiB, pullRequest={}, flowControlTimes={}",
                this.defaultMQPushConsumer.getPullThresholdForQueue(), processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), cachedMessageCount, cachedMessageSizeInMiB, pullRequest, queueFlowControlTimes);
        }
        return;
    }

    if (cachedMessageSizeInMiB > this.defaultMQPushConsumer.getPullThresholdSizeForQueue()) {
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL); // ⑨
        if ((queueFlowControlTimes++ % 1000) == 0) {
            log.warn(
                "the cached message size exceeds the threshold {} MiB, so do flow control, minOffset={}, maxOffset={}, count={}, size={} MiB, pullRequest={}, flowControlTimes={}",
                this.defaultMQPushConsumer.getPullThresholdSizeForQueue(), processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), cachedMessageCount, cachedMessageSizeInMiB, pullRequest, queueFlowControlTimes);
        }
        return;
    }

    if (!this.consumeOrderly) { // ⑩
        if (processQueue.getMaxSpan() > this.defaultMQPushConsumer.getConsumeConcurrentlyMaxSpan()) {
            this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL); // ⑪
            if ((queueMaxSpanFlowControlTimes++ % 1000) == 0) {
                log.warn(
                    "the queue's messages, span too long, so do flow control, minOffset={}, maxOffset={}, maxSpan={}, pullRequest={}, flowControlTimes={}",
                    processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), processQueue.getMaxSpan(),
                    pullRequest, queueMaxSpanFlowControlTimes);
            }
            return;
        }
    } else {
        if (processQueue.isLocked()) { // ⑫
            if (!pullRequest.isLockedFirst()) {
                final long offset = this.rebalanceImpl.computePullFromWhere(pullRequest.getMessageQueue());
                boolean brokerBusy = offset < pullRequest.getNextOffset();
                log.info("the first time to pull message, so fix offset from broker. pullRequest: {} NewOffset: {} brokerBusy: {}",
                    pullRequest, offset, brokerBusy);
                if (brokerBusy) {
                    log.info("[NOTIFYME]the first time to pull message, but pull request offset larger than broker consume offset. pullRequest: {} NewOffset: {}",
                        pullRequest, offset);
                }

                pullRequest.setLockedFirst(true);
                pullRequest.setNextOffset(offset);
            }
        } else {
            this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION); // ⑬
            log.info("pull message later because not locked in broker, {}", pullRequest);
            return;
        }
    }

    final SubscriptionData subscriptionData = this.rebalanceImpl.getSubscriptionInner().get(pullRequest.getMessageQueue().getTopic()); // ⑭
    if (null == subscriptionData) {
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION); // ⑮
        log.warn("find the consumer's subscription failed, {}", pullRequest);
        return;
    }

    final long beginTimestamp = System.currentTimeMillis();

    PullCallback pullCallback = new PullCallback() { /**
        该出实现代码较长，也较为重要，抽出来在下面分析
    */};

    boolean commitOffsetEnable = false;
    long commitOffsetValue = 0L;
    if (MessageModel.CLUSTERING == this.defaultMQPushConsumer.getMessageModel()) {
        commitOffsetValue = this.offsetStore.readOffset(pullRequest.getMessageQueue(), ReadOffsetType.READ_FROM_MEMORY);
        if (commitOffsetValue > 0) {
            commitOffsetEnable = true;
        }
    }

    String subExpression = null;
    boolean classFilter = false;
    SubscriptionData sd = this.rebalanceImpl.getSubscriptionInner().get(pullRequest.getMessageQueue().getTopic());
    if (sd != null) {
        if (this.defaultMQPushConsumer.isPostSubscriptionWhenPull() && !sd.isClassFilterMode()) {
            subExpression = sd.getSubString();
        }

        classFilter = sd.isClassFilterMode();
    }

    int sysFlag = PullSysFlag.buildSysFlag(
        commitOffsetEnable, // commitOffset
        true, // suspend
        subExpression != null, // subscription
        classFilter // class filter
    );
    try {
        this.pullAPIWrapper.pullKernelImpl(
            pullRequest.getMessageQueue(),
            subExpression,
            subscriptionData.getExpressionType(),
            subscriptionData.getSubVersion(),
            pullRequest.getNextOffset(),
            this.defaultMQPushConsumer.getPullBatchSize(),
            sysFlag,
            commitOffsetValue,
            BROKER_SUSPEND_MAX_TIME_MILLIS,
            CONSUMER_TIMEOUT_MILLIS_WHEN_SUSPEND,
            CommunicationMode.ASYNC,
            pullCallback
        ); // ⑯
    } catch (Exception e) {
        log.error("pullKernelImpl exception", e);
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION);
    }
}
```
①取出当前PullRequest的ProcessQueue对象。
②记录当前进行消息拉取的时间，为下面的延迟执行拉取消息做好时间标志。
③确保当前consumer是处于运行状态。
④出现异常，延迟执行拉取操作，延迟时间`PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION=3000ms`。
⑤当前consumer被暂停，延迟执行拉取操作，延迟时间`PULL_TIME_DELAY_MILLS_WHEN_SUSPEND=1000ms`。
⑥和⑦分别获取当前ProcessQueue缓存的消息条数和缓存的消息大小，这两个指标主要是对消息的拉取进行流量控制。
⑧和⑨就是对⑥和⑦的实现，当cachedMessageCount或者cachedMessageSizeInMiB大于设定的阈值，就会延迟执行拉取操作，延迟时间`PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL=50`。
⑩判断当前消费模式是否为顺序的，否的话执行，判断当前ProcessQueue的最大offset跨度是否超过consumer的默认值，超过执行⑪，延迟执行消息拉取，延迟时间`PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL=50`。
⑥-⑪的过程，都是对消息拉取流量的控制，超过了阈值都会进行延迟拉取，除了延迟拉取，也进行了额外的控制，每种流控方式都有一个统计值，如果统计值达到一定的范围，则当前拉取操作直接返回。
⑫和⑬是对顺序消费模式的一个流控处理。
⑭获取topic订阅信息，如果subscriptionData为空，执行⑮操作，延迟执行拉取操作，延迟时间`PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION=3000ms`。
⑯是消息拉取的核心实现。

消息的核心实现中，在PullAPIWrapper类中对消息拉取请求进行封装，然后在MQClientAPIImpl类中和Broker发起请求，进行消息的拉取，请求码为:`RequestCode.PULL_MESSAGE=11`。

Push模式的消息拉取，会有个PullCallBack对象，当消息拉取完成会回调这个类的onSuccess方法，若发送异常，回调onException方法。以下源码分析在拉取消息后PullCallBack对象的两个方法的实现:

```java
PullCallback pullCallback = new PullCallback() {
    @Override
    public void onSuccess(PullResult pullResult) {
        if (pullResult != null) {
            pullResult = DefaultMQPushConsumerImpl.this.pullAPIWrapper.processPullResult(pullRequest.getMessageQueue(), pullResult,
                subscriptionData);

            switch (pullResult.getPullStatus()) { 
                case FOUND: // ①
                    long prevRequestOffset = pullRequest.getNextOffset();
                    pullRequest.setNextOffset(pullResult.getNextBeginOffset());
                    long pullRT = System.currentTimeMillis() - beginTimestamp; // ②
DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullRT(pullRequest.getConsumerGroup(),
                        pullRequest.getMessageQueue().getTopic(), pullRT); // ③

                    long firstMsgOffset = Long.MAX_VALUE;
                    if (pullResult.getMsgFoundList() == null || pullResult.getMsgFoundList().isEmpty()) {
                        DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest); // ④
                    } else {
                        firstMsgOffset = pullResult.getMsgFoundList().get(0).getQueueOffset();

                        DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullTPS(pullRequest.getConsumerGroup(),
                            pullRequest.getMessageQueue().getTopic(), pullResult.getMsgFoundList().size());

                        boolean dispatchToConsume = processQueue.putMessage(pullResult.getMsgFoundList());
                        DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(
                            pullResult.getMsgFoundList(),
                            processQueue,
                            pullRequest.getMessageQueue(),
                            dispatchToConsume); // ⑤

                        if (DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval() > 0) {
                            DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest,
                                DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval());
                        } else {
                            DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                        }
                    }

                    if (pullResult.getNextBeginOffset() < prevRequestOffset
                        || firstMsgOffset < prevRequestOffset) {
                        log.warn(
                            "[BUG] pull message result maybe data wrong, nextBeginOffset: {} firstMsgOffset: {} prevRequestOffset: {}",
                            pullResult.getNextBeginOffset(),
                            firstMsgOffset,
                            prevRequestOffset);
                    }

                    break;
                case NO_NEW_MSG: // ⑥
                    pullRequest.setNextOffset(pullResult.getNextBeginOffset());

                    DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);

                    DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                    break;
                case NO_MATCHED_MSG: // ⑦
                    pullRequest.setNextOffset(pullResult.getNextBeginOffset());

                    DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);

                    DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                    break;
                case OFFSET_ILLEGAL: // ⑧
                    log.warn("the pull request offset illegal, {} {}",
                        pullRequest.toString(), pullResult.toString());
                    pullRequest.setNextOffset(pullResult.getNextBeginOffset());

                    pullRequest.getProcessQueue().setDropped(true);
                    DefaultMQPushConsumerImpl.this.executeTaskLater(new Runnable() {

                        @Override
                        public void run() {
                            try {
                                DefaultMQPushConsumerImpl.this.offsetStore.updateOffset(pullRequest.getMessageQueue(),
                                    pullRequest.getNextOffset(), false);

                                DefaultMQPushConsumerImpl.this.offsetStore.persist(pullRequest.getMessageQueue());

                                DefaultMQPushConsumerImpl.this.rebalanceImpl.removeProcessQueue(pullRequest.getMessageQueue());

                                log.warn("fix the pull request offset, {}", pullRequest);
                            } catch (Throwable e) {
                                log.error("executeTaskLater Exception", e);
                            }
                        }
                    }, 10000);
                    break;
                default:
                    break;
            }
        }
    }

    @Override
    public void onException(Throwable e) { // ⑨
        if (!pullRequest.getMessageQueue().getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
            log.warn("execute the pull request exception", e);
        }

        DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION);
    }
};
```
①当前pullStatus为FOUND，意味着，拉取到了消息。
②和③对拉取的时间窗口进行了统计。
④如果拉取到的消息列表为空，对当前的PullRequest立马从新执行一次。
⑤发起消息消费请求。
⑥和⑦的status，都对offset进行调整，立马执行拉一次的拉取请求。
⑧对于`OFFSET_ILLEGAL`的状态，将会提交一个定时任务，从远程存储更新offset信息，毕竟当前的ProcessQueue从所属的MessageQueue删除。

以上就是consumer基于push模式向broker拉取消息的过程。消息的消费以及消费失败重试，在下一篇讲解。