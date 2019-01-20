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
package top.ljming.mqconsumer.clients;

import org.apache.commons.lang3.StringUtils;
import org.apache.rocketmq.client.consumer.DefaultMQPullConsumer;
import org.apache.rocketmq.client.consumer.PullResult;
import org.apache.rocketmq.client.exception.MQBrokerException;
import org.apache.rocketmq.client.exception.MQClientException;
import org.apache.rocketmq.common.message.MessageQueue;
import org.apache.rocketmq.remoting.exception.RemotingException;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.beans.factory.annotation.Value;
import top.ljming.mqconsumer.topic.SimpleTopic;

import java.time.OffsetDateTime;
import java.util.HashMap;
import java.util.Map;
import java.util.Set;

/**
 * 描述类的功能.
 *
 * @author ljming
 */
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

## consumser 启动

consumer的启动也是rocketmq的一个客户端的启动，比较注意的几点是：
- offset的加载: cluster的消费模式，从远程broker加载；broadCast模式，从本地文件加载。
- consumerMessageService的类型，根据用户注册的监听器类型有关，分为ConsumeMessageOrderlyService 和 ConsumeMessageConcurrentlyService
- pullMessageService.start(): 拉取消息的服务
- rebalanceService.start(): 负载均衡服务

```java
public synchronized void start() throws MQClientException {
    switch (this.serviceState) {
        case CREATE_JUST:
            log.info("the consumer [{}] start beginning. messageModel={}, isUnitMode={}", this.defaultMQPushConsumer.getConsumerGroup(),
                this.defaultMQPushConsumer.getMessageModel(), this.defaultMQPushConsumer.isUnitMode());
            this.serviceState = ServiceState.START_FAILED;

            this.checkConfig();

            this.copySubscription();

            if (this.defaultMQPushConsumer.getMessageModel() == MessageModel.CLUSTERING) {
                this.defaultMQPushConsumer.changeInstanceNameToPID();
            }
            // 创建一个consumer实例
            this.mQClientFactory = MQClientManager.getInstance().getAndCreateMQClientInstance(this.defaultMQPushConsumer, this.rpcHook);

            this.rebalanceImpl.setConsumerGroup(this.defaultMQPushConsumer.getConsumerGroup());
            this.rebalanceImpl.setMessageModel(this.defaultMQPushConsumer.getMessageModel());
            this.rebalanceImpl.setAllocateMessageQueueStrategy(this.defaultMQPushConsumer.getAllocateMessageQueueStrategy());
            this.rebalanceImpl.setmQClientFactory(this.mQClientFactory);

            this.pullAPIWrapper = new PullAPIWrapper(
                mQClientFactory,
                this.defaultMQPushConsumer.getConsumerGroup(), isUnitMode());
            this.pullAPIWrapper.registerFilterMessageHook(filterMessageHookList);
            // 加载offset
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
            this.offsetStore.load();
            // 消费消息的服务启动
            if (this.getMessageListenerInner() instanceof MessageListenerOrderly) {
                this.consumeOrderly = true;
                this.consumeMessageService =
                    new ConsumeMessageOrderlyService(this, (MessageListenerOrderly) this.getMessageListenerInner());
            } else if (this.getMessageListenerInner() instanceof MessageListenerConcurrently) {
                this.consumeOrderly = false;
                this.consumeMessageService =
                    new ConsumeMessageConcurrentlyService(this, (MessageListenerConcurrently) this.getMessageListenerInner());
            }

            this.consumeMessageService.start();

            boolean registerOK = mQClientFactory.registerConsumer(this.defaultMQPushConsumer.getConsumerGroup(), this);
            if (!registerOK) {
                this.serviceState = ServiceState.CREATE_JUST;
                this.consumeMessageService.shutdown();
                throw new MQClientException("The consumer group[" + this.defaultMQPushConsumer.getConsumerGroup()
                    + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                    null);
            }

            mQClientFactory.start(); // consumer 实例启动
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

    this.updateTopicSubscribeInfoWhenSubscriptionChanged();
    this.mQClientFactory.checkClientInBroker();
    this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
    this.mQClientFactory.rebalanceImmediately();
}
```

## consumer消息消费

这里基于Push模式分析。在consumer启动的时候，说到consumer实例启动时，会启动一个pullMessageService的服务，这个服务主要就是用来拉取消息的。虽然是push模式，但是也是consumer主动发起拉取消息给broker，broker接收到拉取消息的请求，基于长轮询的策略，给予响应。

### PullMessageService: 消息拉取服务

```java
public void run() {
    log.info(this.getServiceName() + " service started");

    while (!this.isStopped()) {
        try {
            PullRequest pullRequest = this.pullRequestQueue.take();
            this.pullMessage(pullRequest);
        } catch (InterruptedException ignored) {
        } catch (Exception e) {
            log.error("Pull Message Service Run Method exception", e);
        }
    }

    log.info(this.getServiceName() + " service end");
}
```

PullMessageService一旦启动就会从pullRequestQueue中获取PullRequest。pullRequestQueue是一个阻塞队列，只有当pullRequestQueue.take()获取到数据时才会返回，否者当前线程就会一直阻塞，直到拿到数据。

问题：当PullMessageService启动时，pullRequestQueue是空的，也就是说this.pullRequestQueue.take()是拿不到数据，那当前线程岂不是一直都被阻塞了。也没有下面的拉取过程之说了？

继续来看pullMessage的实现。拉取消息的具体实现在DefaultMQPushConsumerImpl类中。

```java
public void pullMessage(final PullRequest pullRequest) {
    final ProcessQueue processQueue = pullRequest.getProcessQueue();
    if (processQueue.isDropped()) {
        log.info("the pull request[{}] is dropped.", pullRequest.toString());
        return;
    }

    pullRequest.getProcessQueue().setLastPullTimestamp(System.currentTimeMillis());

    try {
        this.makeSureStateOK();
    } catch (MQClientException e) {
        log.warn("pullMessage exception, consumer state not ok", e);
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION);
        return;
    }

    if (this.isPause()) {
        log.warn("consumer was paused, execute pull request later. instanceName={}, group={}", this.defaultMQPushConsumer.getInstanceName(), this.defaultMQPushConsumer.getConsumerGroup());
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_SUSPEND);
        return;
    }

    long cachedMessageCount = processQueue.getMsgCount().get();
    long cachedMessageSizeInMiB = processQueue.getMsgSize().get() / (1024 * 1024);

    if (cachedMessageCount > this.defaultMQPushConsumer.getPullThresholdForQueue()) {
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL);
        if ((queueFlowControlTimes++ % 1000) == 0) {
            log.warn(
                "the cached message count exceeds the threshold {}, so do flow control, minOffset={}, maxOffset={}, count={}, size={} MiB, pullRequest={}, flowControlTimes={}",
                this.defaultMQPushConsumer.getPullThresholdForQueue(), processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), cachedMessageCount, cachedMessageSizeInMiB, pullRequest, queueFlowControlTimes);
        }
        return;
    }

    if (cachedMessageSizeInMiB > this.defaultMQPushConsumer.getPullThresholdSizeForQueue()) {
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL);
        if ((queueFlowControlTimes++ % 1000) == 0) {
            log.warn(
                "the cached message size exceeds the threshold {} MiB, so do flow control, minOffset={}, maxOffset={}, count={}, size={} MiB, pullRequest={}, flowControlTimes={}",
                this.defaultMQPushConsumer.getPullThresholdSizeForQueue(), processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), cachedMessageCount, cachedMessageSizeInMiB, pullRequest, queueFlowControlTimes);
        }
        return;
    }

    if (!this.consumeOrderly) {
        if (processQueue.getMaxSpan() > this.defaultMQPushConsumer.getConsumeConcurrentlyMaxSpan()) {
            this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_FLOW_CONTROL);
            if ((queueMaxSpanFlowControlTimes++ % 1000) == 0) {
                log.warn(
                    "the queue's messages, span too long, so do flow control, minOffset={}, maxOffset={}, maxSpan={}, pullRequest={}, flowControlTimes={}",
                    processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), processQueue.getMaxSpan(),
                    pullRequest, queueMaxSpanFlowControlTimes);
            }
            return;
        }
    } else {
        if (processQueue.isLocked()) {
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
            this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION);
            log.info("pull message later because not locked in broker, {}", pullRequest);
            return;
        }
    }

    final SubscriptionData subscriptionData = this.rebalanceImpl.getSubscriptionInner().get(pullRequest.getMessageQueue().getTopic());
    if (null == subscriptionData) {
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION);
        log.warn("find the consumer's subscription failed, {}", pullRequest);
        return;
    }

    final long beginTimestamp = System.currentTimeMillis();

    PullCallback pullCallback = new PullCallback() {
        @Override
        public void onSuccess(PullResult pullResult) {
            if (pullResult != null) {
                pullResult = DefaultMQPushConsumerImpl.this.pullAPIWrapper.processPullResult(pullRequest.getMessageQueue(), pullResult,
                    subscriptionData);

                switch (pullResult.getPullStatus()) {
                    case FOUND:
                        long prevRequestOffset = pullRequest.getNextOffset();
                        pullRequest.setNextOffset(pullResult.getNextBeginOffset());
                        long pullRT = System.currentTimeMillis() - beginTimestamp;
                        DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullRT(pullRequest.getConsumerGroup(),
                            pullRequest.getMessageQueue().getTopic(), pullRT);

                        long firstMsgOffset = Long.MAX_VALUE;
                        if (pullResult.getMsgFoundList() == null || pullResult.getMsgFoundList().isEmpty()) {
                            DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                        } else {
                            firstMsgOffset = pullResult.getMsgFoundList().get(0).getQueueOffset();

                            DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullTPS(pullRequest.getConsumerGroup(),
                                pullRequest.getMessageQueue().getTopic(), pullResult.getMsgFoundList().size());

                            boolean dispatchToConsume = processQueue.putMessage(pullResult.getMsgFoundList());
                            DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(
                                pullResult.getMsgFoundList(),
                                processQueue,
                                pullRequest.getMessageQueue(),
                                dispatchToConsume);

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
                    case NO_NEW_MSG:
                        pullRequest.setNextOffset(pullResult.getNextBeginOffset());

                        DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);

                        DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                        break;
                    case NO_MATCHED_MSG:
                        pullRequest.setNextOffset(pullResult.getNextBeginOffset());

                        DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);

                        DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                        break;
                    case OFFSET_ILLEGAL:
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
        public void onException(Throwable e) {
            if (!pullRequest.getMessageQueue().getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                log.warn("execute the pull request exception", e);
            }

            DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION);
        }
    };

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
        );
    } catch (Exception e) {
        log.error("pullKernelImpl exception", e);
        this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION);
    }
}

```

在提交pullRequest时，有一个流量控制，PullRequst对象有个ProcessQueue对象，ProcessQueue对象保存了对应MessageQueue消息处理状态的快照。ProcessQueue对象里主要的内容是一个 TreeMap 和一个读写锁。 TreeMap 里以 MessageQueue 的 Offset 作为 Key，以消息内容为 Value，保存了 所有从 MessageQueue 获取到，但是还未被处理的消息; 读写锁控制着多个线程对 TreeMap 对象的并发访问 。

consumer客户端在提交拉取请求前都会做以下几个判断:

![消息流量控制](flow_controller.jpg)

当上述判断被流控了，当前PullRequest对象会被扔到PullMessageService服务的一个定时任务里，任务延迟时间根据不同的流控时间而不同。

```java
public void executePullRequestLater(final PullRequest pullRequest, final long timeDelay) {
    if (!isStopped()) {
        this.scheduledExecutorService.schedule(new Runnable() {
            @Override
            public void run() {
                PullMessageService.this.executePullRequestImmediately(pullRequest);
            }
        }, timeDelay, TimeUnit.MILLISECONDS);
    } else {
        log.warn("PullMessageServiceScheduledThread has shutdown");
    }
}
```

PullCallback 对象在PullRequest执行完成，返回时进行回调。拉取的核心实现在pullAPIWrapper中实现，调用了 pullAPIWrapper.pullKernelImpl方法。在这个方法中，做了以下操作：

- 获取要拉取消息的broker地址
- 构建PullMessageRequestHeader对象

然后通过调用MQClientAPIImpl对象的PullMessage方法向broker发起拉取消息请求。MQClientAPIImpl封装了客户端和服务器端通信的api。

在MQClientAPIImpl.pullMessage中，根据Communication不同，进行不同的实现。

```java
public PullResult pullMessage(
    final String addr,
    final PullMessageRequestHeader requestHeader,
    final long timeoutMillis,
    final CommunicationMode communicationMode,
    final PullCallback pullCallback
) throws RemotingException, MQBrokerException, InterruptedException {
    RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.PULL_MESSAGE, requestHeader);

    switch (communicationMode) {
        case ONEWAY:
            assert false;
            return null;
        case ASYNC:
            this.pullMessageAsync(addr, request, timeoutMillis, pullCallback);
            return null;
        case SYNC:
            return this.pullMessageSync(addr, request, timeoutMillis);
        default:
            assert false;
            break;
    }

    return null;
}
```

在拉取消息的实现中，push模式的consumer，采用的ASYNC的通信方式，pull模式的consumer，采用的SYNC的通信方式。由于consumer是基于push模式分析的，这里看ASYNC通信方式的实现。ASYNC的拉取方式，会传入一个PullCallback类，有两个抽象方法，onSuccess和onException，针对拉取消息放回结果进行不同的调用不同的实现方法。

```java
PullCallback pullCallback = new PullCallback() {
    @Override
    public void onSuccess(PullResult pullResult) {
        if (pullResult != null) {
            pullResult = DefaultMQPushConsumerImpl.this.pullAPIWrapper.processPullResult(pullRequest.getMessageQueue(), pullResult,
                subscriptionData);

            switch (pullResult.getPullStatus()) {
                case FOUND:
                    long prevRequestOffset = pullRequest.getNextOffset();
                    pullRequest.setNextOffset(pullResult.getNextBeginOffset());
                    long pullRT = System.currentTimeMillis() - beginTimestamp;
                    DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullRT(pullRequest.getConsumerGroup(),
                        pullRequest.getMessageQueue().getTopic(), pullRT);

                    long firstMsgOffset = Long.MAX_VALUE;
                    if (pullResult.getMsgFoundList() == null || pullResult.getMsgFoundList().isEmpty()) {
                        DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                    } else {
                        firstMsgOffset = pullResult.getMsgFoundList().get(0).getQueueOffset();

                        DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullTPS(pullRequest.getConsumerGroup(),
                            pullRequest.getMessageQueue().getTopic(), pullResult.getMsgFoundList().size());

                        boolean dispatchToConsume = processQueue.putMessage(pullResult.getMsgFoundList());
                        DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(
                            pullResult.getMsgFoundList(),
                            processQueue,
                            pullRequest.getMessageQueue(),
                            dispatchToConsume);

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
                case NO_NEW_MSG:
                    pullRequest.setNextOffset(pullResult.getNextBeginOffset());

                    DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);

                    DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                    break;
                case NO_MATCHED_MSG:
                    pullRequest.setNextOffset(pullResult.getNextBeginOffset());

                    DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);

                    DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                    break;
                case OFFSET_ILLEGAL:
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
    public void onException(Throwable e) {
        if (!pullRequest.getMessageQueue().getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
            log.warn("execute the pull request exception", e);
        }

        DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION);
    }
}
```

拉取成功，PullResult的PullStatus属性有以下几种值：

- FOUND: 表示成功拉取到消息
- NO_NEW_MSG: 表示当前messageQueue中没有新消息
- NO_MATCHED_MSG: 表示当前MessageQueue中没有满足条件的消息
- OFFSET_ILLEGAL: 表示传入的offset值非法

**FOUND**

- pullRequest设置offset为下一个offset，构建下次拉取消息的请求对象
- consumerStatsManager时间窗口内的拉取次数+1
- 如果拉取到的消息为空，立即执行下一次拉取请求
- consumerStatsManager TPS统计
- 将拉取到的消息添加到ProcessQueue中
- 提交一个消息消费任务
- 判断拉取时间间隔，进行下一次的消息拉取

**NO_NEW_MSG || NO_MATCHED_MSG**

取下一个offset的数据，立即执行pullRequest

**OFFSET_ILLEGAL**

丢弃当前processQueue，启动一个延时任务：跟新MessageQueue的offset，持久化offset，在负载均衡中丢弃当前processQueue。

### consumeMessageService 服务消费消息

前面讲到当PullStatus为FOUND，且拉取到的消息不为空，则提交了一个消息消费的任务。

```java
DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(pullResult.getMsgFoundList(), processQueue, pullRequest.getMessageQueue(), dispatchToConsume);
```

这里的实现就得看ConsumeMessageService.submitConsumerRequest方法

```java
public void submitConsumeRequest(
    final List<MessageExt> msgs,
    final ProcessQueue processQueue,
    final MessageQueue messageQueue,
    final boolean dispatchToConsume) {
    final int consumeBatchSize = this.defaultMQPushConsumer.getConsumeMessageBatchMaxSize();
    if (msgs.size() <= consumeBatchSize) {
        ConsumeRequest consumeRequest = new ConsumeRequest(msgs, processQueue, messageQueue);
        try {
            this.consumeExecutor.submit(consumeRequest);
        } catch (RejectedExecutionException e) {
            this.submitConsumeRequestLater(consumeRequest);
        }
    } else {
        for (int total = 0; total < msgs.size(); ) {
            List<MessageExt> msgThis = new ArrayList<MessageExt>(consumeBatchSize);
            for (int i = 0; i < consumeBatchSize; i++, total++) {
                if (total < msgs.size()) {
                    msgThis.add(msgs.get(total));
                } else {
                    break;
                }
            }

            ConsumeRequest consumeRequest = new ConsumeRequest(msgThis, processQueue, messageQueue);
            try {
                this.consumeExecutor.submit(consumeRequest);
            } catch (RejectedExecutionException e) {
                for (; total < msgs.size(); total++) {
                    msgThis.add(msgs.get(total));
                }

                this.submitConsumeRequestLater(consumeRequest);
            }
        }
    }
}
```

这里的实现，在线程池中提交了一个ConsumerRequest的任务，该ConsumerRequest对象实现了Runnable接口，那么就看下ConsumerRequest的run方法：

```java
public void run() {
    if (this.processQueue.isDropped()) {
        log.info("the message queue not be able to consume, because it's dropped. group={} {}", ConsumeMessageConcurrentlyService.this.consumerGroup, this.messageQueue);
        return;
    }

    MessageListenerConcurrently listener = ConsumeMessageConcurrentlyService.this.messageListener;
    ConsumeConcurrentlyContext context = new ConsumeConcurrentlyContext(messageQueue);
    ConsumeConcurrentlyStatus status = null;

    ConsumeMessageContext consumeMessageContext = null;
    if (ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.hasHook()) {
        consumeMessageContext = new ConsumeMessageContext();
        consumeMessageContext.setConsumerGroup(defaultMQPushConsumer.getConsumerGroup());
        consumeMessageContext.setProps(new HashMap<String, String>());
        consumeMessageContext.setMq(messageQueue);
        consumeMessageContext.setMsgList(msgs);
        consumeMessageContext.setSuccess(false);
        ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.executeHookBefore(consumeMessageContext);
    }

    long beginTimestamp = System.currentTimeMillis();
    boolean hasException = false;
    ConsumeReturnType returnType = ConsumeReturnType.SUCCESS;
    try {
        ConsumeMessageConcurrentlyService.this.resetRetryTopic(msgs);
        if (msgs != null && !msgs.isEmpty()) {
            for (MessageExt msg : msgs) {
                MessageAccessor.setConsumeStartTimeStamp(msg, String.valueOf(System.currentTimeMillis()));
            }
        }
        status = listener.consumeMessage(Collections.unmodifiableList(msgs), context); // 用户自定义的消息监听器，进行消息消费，需用用户自己实现
    } catch (Throwable e) {
        log.warn("consumeMessage exception: {} Group: {} Msgs: {} MQ: {}",
            RemotingHelper.exceptionSimpleDesc(e),
            ConsumeMessageConcurrentlyService.this.consumerGroup,
            msgs,
            messageQueue);
        hasException = true;
    }
    long consumeRT = System.currentTimeMillis() - beginTimestamp;
    if (null == status) {
        if (hasException) {
            returnType = ConsumeReturnType.EXCEPTION;
        } else {
            returnType = ConsumeReturnType.RETURNNULL;
        }
    } else if (consumeRT >= defaultMQPushConsumer.getConsumeTimeout() * 60 * 1000) {
        returnType = ConsumeReturnType.TIME_OUT;
    } else if (ConsumeConcurrentlyStatus.RECONSUME_LATER == status) {
        returnType = ConsumeReturnType.FAILED;
    } else if (ConsumeConcurrentlyStatus.CONSUME_SUCCESS == status) {
        returnType = ConsumeReturnType.SUCCESS;
    }

    if (ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.hasHook()) {
        consumeMessageContext.getProps().put(MixAll.CONSUME_CONTEXT_TYPE, returnType.name());
    }

    if (null == status) {
        log.warn("consumeMessage return null, Group: {} Msgs: {} MQ: {}",
            ConsumeMessageConcurrentlyService.this.consumerGroup,
            msgs,
            messageQueue);
        status = ConsumeConcurrentlyStatus.RECONSUME_LATER;
    }

    if (ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.hasHook()) {
        consumeMessageContext.setStatus(status.toString());
        consumeMessageContext.setSuccess(ConsumeConcurrentlyStatus.CONSUME_SUCCESS == status);
        ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.executeHookAfter(consumeMessageContext);
    }

    ConsumeMessageConcurrentlyService.this.getConsumerStatsManager()
        .incConsumeRT(ConsumeMessageConcurrentlyService.this.consumerGroup, messageQueue.getTopic(), consumeRT);

    if (!processQueue.isDropped()) {
        ConsumeMessageConcurrentlyService.this.processConsumeResult(status, context, this);
    } else {
        log.warn("processQueue is dropped without process consume result. messageQueue={}, msgs={}", messageQueue, msgs);
    }
}
```

用户自定对消息消费完会返回一个状态：
- CONSUME_SUCCESS: 消费成功
- RECONSUME_LATER: 稍后重新消费

### consumeMessageService 服务消息消费重试机制

上面说到，consumer端消费消息，会返回一个状态--ConsumeConcurrentlyStatus（并发模式下）。当返回的状态是RECONSUME_LATER时，则表示该消息需要稍后重新消费。消息消费完之后，会对返回的status进行处理，处理的逻辑在ConsumeMessageConcurrentlyService.processConsumerResult:

![处理消费结果](process_consumer_result.jpg)

从图中可以看出，status=CONSUME_SUCCESS时，会计算出成功的消息条数和失败的消息条数，然后通过ConsumerStatsManager统计消息消费和失败的TPS；

当status=RECONSUME_LATER; ackIndex=-1; 这时就需要将失败的消息重新消费。

![重新消费](consumer_send_back.jpg)

根据不同的消费状态，设置的不同ackIndex的值。当从代码中可知，当status=CONSUME_SUCCESS时，是不会给broker发送back消息的。当消息需要被重新消费时，才会想broker发送back消息。如果发送给broker的back消息的msg失败，则会将这些消息扔到一个集合，然后重新发送一个ConsumeRequest。

给broker发送消息重发请求是，发送的REQUEST_CODE=CONSUMER_SEND_MSG_BACK=36。broker处理这个请求的处理器是SendMessageProcessor。在broker源码分析中，broker初始化是，会注册一系列的处理器。对于RequestCode=CONSUMER_SEND_MSG_BACK的请求，对应的处理器是SendMessageProcessor。
```java
this.remotingServer.registerProcessor(RequestCode.CONSUMER_SEND_MSG_BACK, sendProcessor, this.sendMessageExecutor);
```

接下来， 就看看broker接收到这个请求，如何处理的。这个实现在SendMessageProcessor.consumerSendMsgBack()方法中

```java
private RemotingCommand consumerSendMsgBack(final ChannelHandlerContext ctx, final RemotingCommand request)
    throws RemotingCommandException {
    final RemotingCommand response = RemotingCommand.createResponseCommand(null);
    final ConsumerSendMsgBackRequestHeader requestHeader =
        (ConsumerSendMsgBackRequestHeader)request.decodeCommandCustomHeader(ConsumerSendMsgBackRequestHeader.class);

    if (this.hasConsumeMessageHook() && !UtilAll.isBlank(requestHeader.getOriginMsgId())) {

        ConsumeMessageContext context = new ConsumeMessageContext();
        context.setConsumerGroup(requestHeader.getGroup());
        context.setTopic(requestHeader.getOriginTopic());
        context.setCommercialRcvStats(BrokerStatsManager.StatsType.SEND_BACK);
        context.setCommercialRcvTimes(1);
        context.setCommercialOwner(request.getExtFields().get(BrokerStatsManager.COMMERCIAL_OWNER));

        this.executeConsumeMessageHookAfter(context);
    }

    SubscriptionGroupConfig subscriptionGroupConfig =
        this.brokerController.getSubscriptionGroupManager().findSubscriptionGroupConfig(requestHeader.getGroup());
    if (null == subscriptionGroupConfig) {
        response.setCode(ResponseCode.SUBSCRIPTION_GROUP_NOT_EXIST);
        response.setRemark("subscription group not exist, " + requestHeader.getGroup() + " "
            + FAQUrl.suggestTodo(FAQUrl.SUBSCRIPTION_GROUP_NOT_EXIST));
        return response;
    }

    if (!PermName.isWriteable(this.brokerController.getBrokerConfig().getBrokerPermission())) {
        response.setCode(ResponseCode.NO_PERMISSION);
        response.setRemark("the broker[" + this.brokerController.getBrokerConfig().getBrokerIP1() + "] sending message is forbidden");
        return response;
    }

    if (subscriptionGroupConfig.getRetryQueueNums() <= 0) {
        response.setCode(ResponseCode.SUCCESS);
        response.setRemark(null);
        return response;
    }

    String newTopic = MixAll.getRetryTopic(requestHeader.getGroup());
    int queueIdInt = Math.abs(this.random.nextInt() % 99999999) % subscriptionGroupConfig.getRetryQueueNums();

    int topicSysFlag = 0;
    if (requestHeader.isUnitMode()) {
        topicSysFlag = TopicSysFlag.buildSysFlag(false, true);
    }

    TopicConfig topicConfig = this.brokerController.getTopicConfigManager().createTopicInSendMessageBackMethod(
        newTopic,
        subscriptionGroupConfig.getRetryQueueNums(),
        PermName.PERM_WRITE | PermName.PERM_READ, topicSysFlag);
    if (null == topicConfig) {
        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark("topic[" + newTopic + "] not exist");
        return response;
    }

    if (!PermName.isWriteable(topicConfig.getPerm())) {
        response.setCode(ResponseCode.NO_PERMISSION);
        response.setRemark(String.format("the topic[%s] sending message is forbidden", newTopic));
        return response;
    }

    MessageExt msgExt = this.brokerController.getMessageStore().lookMessageByOffset(requestHeader.getOffset());
    if (null == msgExt) {
        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark("look message by offset failed, " + requestHeader.getOffset());
        return response;
    }

    final String retryTopic = msgExt.getProperty(MessageConst.PROPERTY_RETRY_TOPIC);
    if (null == retryTopic) {
        MessageAccessor.putProperty(msgExt, MessageConst.PROPERTY_RETRY_TOPIC, msgExt.getTopic());
    }
    msgExt.setWaitStoreMsgOK(false);

    int delayLevel = requestHeader.getDelayLevel();

    int maxReconsumeTimes = subscriptionGroupConfig.getRetryMaxTimes();
    if (request.getVersion() >= MQVersion.Version.V3_4_9.ordinal()) {
        maxReconsumeTimes = requestHeader.getMaxReconsumeTimes();
    }

    if (msgExt.getReconsumeTimes() >= maxReconsumeTimes
        || delayLevel < 0) {  // 如果最大重试次数超过默认次数或者延迟级别小于0，直接入死信队列
        newTopic = MixAll.getDLQTopic(requestHeader.getGroup());
        queueIdInt = Math.abs(this.random.nextInt() % 99999999) % DLQ_NUMS_PER_GROUP;

        topicConfig = this.brokerController.getTopicConfigManager().createTopicInSendMessageBackMethod(newTopic,
            DLQ_NUMS_PER_GROUP,
            PermName.PERM_WRITE, 0
        );
        if (null == topicConfig) {
            response.setCode(ResponseCode.SYSTEM_ERROR);
            response.setRemark("topic[" + newTopic + "] not exist");
            return response;
        }
    } else {
        if (0 == delayLevel) { //延迟级别为0，表示broker端控制重试延迟时间 
            delayLevel = 3 + msgExt.getReconsumeTimes(); // 设delayLevel=3，即第一次重试是10s最后
        }

        msgExt.setDelayTimeLevel(delayLevel);
    }

    MessageExtBrokerInner msgInner = new MessageExtBrokerInner();
    msgInner.setTopic(newTopic);
    msgInner.setBody(msgExt.getBody());
    msgInner.setFlag(msgExt.getFlag());
    MessageAccessor.setProperties(msgInner, msgExt.getProperties());
    msgInner.setPropertiesString(MessageDecoder.messageProperties2String(msgExt.getProperties()));
    msgInner.setTagsCode(MessageExtBrokerInner.tagsString2tagsCode(null, msgExt.getTags()));

    msgInner.setQueueId(queueIdInt);
    msgInner.setSysFlag(msgExt.getSysFlag());
    msgInner.setBornTimestamp(msgExt.getBornTimestamp());
    msgInner.setBornHost(msgExt.getBornHost());
    msgInner.setStoreHost(this.getStoreHost());
    msgInner.setReconsumeTimes(msgExt.getReconsumeTimes() + 1);

    String originMsgId = MessageAccessor.getOriginMessageId(msgExt);
    MessageAccessor.setOriginMessageId(msgInner, UtilAll.isBlank(originMsgId) ? msgExt.getMsgId() : originMsgId);

    PutMessageResult putMessageResult = this.brokerController.getMessageStore().putMessage(msgInner);
    if (putMessageResult != null) {
        switch (putMessageResult.getPutMessageStatus()) {
            case PUT_OK:
                String backTopic = msgExt.getTopic();
                String correctTopic = msgExt.getProperty(MessageConst.PROPERTY_RETRY_TOPIC);
                if (correctTopic != null) {
                    backTopic = correctTopic;
                }

                this.brokerController.getBrokerStatsManager().incSendBackNums(requestHeader.getGroup(), backTopic);

                response.setCode(ResponseCode.SUCCESS);
                response.setRemark(null);

                return response;
            default:
                break;
        }

        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark(putMessageResult.getPutMessageStatus().name());
        return response;
    }

    response.setCode(ResponseCode.SYSTEM_ERROR);
    response.setRemark("putMessageResult is null");
    return response;
}
```
消息重试会创建一个新的主题-"%RETRY%+consumerGroup", 然后根据新的主题获取TopicConfig，如果TopicConfig为空，则返回异常信息。

delayLevel表示延迟级别，如果msg的重试次数超过最大的重试次数（maxReconsumeTimes，默认16次）或者 delayLevel < 0, 则消息不会进行重试，直接扔到死信队列。但是在实际业务场景下，不会重试那么多次，一般重试3次没有成功，就不需要再重试了，用户可以自定义处理该条消息。

delayLevel值一般为0（表示broker端控制重试次数），-1（表示不重试，直接入死信队列），> 0（客户端控制重试次数）；

![重试次数和延迟控制](reconsumer_delay.jpg)

然后会构建 MessageExtBrokerInner 对象之后，会将消息入盘。入盘的逻辑在MessageStore.putMessage方法实现。

在入盘的时候，针对重试的消息，RocketMq有个名为`SCHEDULE_TOPIC_XXXX`的主题，对应一个ScheduleMessageService服务处理这些消息。这个服务在MessageStore启动的时候启动的。

**ScheduleMessageService.start()**
```java
public void start() {

    for (Map.Entry<Integer, Long> entry : this.delayLevelTable.entrySet()) {
        Integer level = entry.getKey();
        Long timeDelay = entry.getValue(); // 延迟执行时间
        Long offset = this.offsetTable.get(level);
        if (null == offset) {
            offset = 0L;
        }

        if (timeDelay != null) {
            this.timer.schedule(new DeliverDelayedMessageTimerTask(level, offset), FIRST_DELAY_TIME);
        }
    }

    this.timer.scheduleAtFixedRate(new TimerTask() {

        @Override
        public void run() {
            try {
                ScheduleMessageService.this.persist();
            } catch (Throwable e) {
                log.error("scheduleAtFixedRate flush exception", e);
            }
        }
    }, 10000, this.defaultMessageStore.getMessageStoreConfig().getFlushDelayOffsetInterval());
}

```

ScheduleMessageService启动一个定时任务，提交了DeliverDelayedMessageTimerTask---延迟发送消息定时任务。在DeliverDelayedMessageTimerTask中，会将这条消息重新写入磁盘，这次的的topic是上面提到的--"%RETRY%+consumerGroup"。DeliverDelayedMessageTimerTask的run方法调用executeOnTimeup方法，进行消息重试。

```java
public void executeOnTimeup() {
    /** 
    * 通过topicName_queueId为key，从磁盘中获取consumerQueue文件内容
    * queueId 通过delayLevel获取，queueId=delayLevel-1
    * /
    ConsumeQueue cq =
        ScheduleMessageService.this.defaultMessageStore.findConsumeQueue(SCHEDULE_TOPIC,
            delayLevel2QueueId(delayLevel));

    long failScheduleOffset = offset;

    if (cq != null) {
        SelectMappedBufferResult bufferCQ = cq.getIndexBuffer(this.offset);
        if (bufferCQ != null) {
            try {
                long nextOffset = offset;
                int i = 0;
                ConsumeQueueExt.CqExtUnit cqExtUnit = new ConsumeQueueExt.CqExtUnit();
                for (; i < bufferCQ.getSize(); i += ConsumeQueue.CQ_STORE_UNIT_SIZE) {
                    long offsetPy = bufferCQ.getByteBuffer().getLong();
                    int sizePy = bufferCQ.getByteBuffer().getInt();
                    long tagsCode = bufferCQ.getByteBuffer().getLong();

                    if (cq.isExtAddr(tagsCode)) {
                        if (cq.getExt(tagsCode, cqExtUnit)) {
                            tagsCode = cqExtUnit.getTagsCode();
                        } else {
                            //can't find ext content.So re compute tags code.
                            log.error("[BUG] can't find consume queue extend file content!addr={}, offsetPy={}, sizePy={}",
                                tagsCode, offsetPy, sizePy);
                            long msgStoreTime = defaultMessageStore.getCommitLog().pickupStoreTimestamp(offsetPy, sizePy);
                            tagsCode = computeDeliverTimestamp(delayLevel, msgStoreTime);
                        }
                    }

                    long now = System.currentTimeMillis();
                    long deliverTimestamp = this.correctDeliverTimestamp(now, tagsCode);

                    nextOffset = offset + (i / ConsumeQueue.CQ_STORE_UNIT_SIZE);

                    // 延迟时间是否完成
                    long countdown = deliverTimestamp - now;

                    if (countdown <= 0) {
                        // 从commitLog文件加载出message内容
                        MessageExt msgExt =
                            ScheduleMessageService.this.defaultMessageStore.lookMessageByOffset(
                                offsetPy, sizePy);

                        if (msgExt != null) {
                            try {
                                // message封装，topic切换为%RETRY%+consumerGroup
                                MessageExtBrokerInner msgInner = this.messageTimeup(msgExt);
                                // 再次写入commitLog文件，但是此时的topic变化了
                                PutMessageResult putMessageResult =
                                    ScheduleMessageService.this.defaultMessageStore
                                        .putMessage(msgInner);

                                if (putMessageResult != null
                                    && putMessageResult.getPutMessageStatus() == PutMessageStatus.PUT_OK) {
                                    continue;
                                } else {
                                    // XXX: warn and notify me
                                    log.error(
                                        "ScheduleMessageService, a message time up, but reput it failed, topic: {} msgId {}",
                                        msgExt.getTopic(), msgExt.getMsgId());
                                    ScheduleMessageService.this.timer.schedule(
                                        new DeliverDelayedMessageTimerTask(this.delayLevel,
                                            nextOffset), DELAY_FOR_A_PERIOD);
                                    ScheduleMessageService.this.updateOffset(this.delayLevel,
                                        nextOffset);
                                    return;
                                }
                            } catch (Exception e) {
                                /*
                                 * XXX: warn and notify me



                                 */
                                log.error(
                                    "ScheduleMessageService, messageTimeup execute error, drop it. msgExt="
                                        + msgExt + ", nextOffset=" + nextOffset + ",offsetPy="
                                        + offsetPy + ",sizePy=" + sizePy, e);
                            }
                        }
                    } else {
                        ScheduleMessageService.this.timer.schedule(
                            new DeliverDelayedMessageTimerTask(this.delayLevel, nextOffset),
                            countdown);
                        ScheduleMessageService.this.updateOffset(this.delayLevel, nextOffset);
                        return;
                    }
                } // end of for

                nextOffset = offset + (i / ConsumeQueue.CQ_STORE_UNIT_SIZE);
                ScheduleMessageService.this.timer.schedule(new DeliverDelayedMessageTimerTask(
                    this.delayLevel, nextOffset), DELAY_FOR_A_WHILE);
                ScheduleMessageService.this.updateOffset(this.delayLevel, nextOffset);
                return;
            } finally {

                bufferCQ.release();
            }
        } // end of if (bufferCQ != null)
        else {

            long cqMinOffset = cq.getMinOffsetInQueue();
            if (offset < cqMinOffset) {
                failScheduleOffset = cqMinOffset;
                log.error("schedule CQ offset invalid. offset=" + offset + ", cqMinOffset="
                    + cqMinOffset + ", queueId=" + cq.getQueueId());
            }
        }
    } // end of if (cq != null)

    ScheduleMessageService.this.timer.schedule(new DeliverDelayedMessageTimerTask(this.delayLevel,
        failScheduleOffset), DELAY_FOR_A_WHILE);
}
```

