---
title: rocketmq之broker源码学习
date: 2018-12-28 16:28:48
categories: 消息队列
tags:
- RocketMQ
- broker
---

rocketmq之 broker 源码解析.

<!--more-->

## broker 概述
broker 是RocketMQ系统的主要组成部分。

broker接收存储从producer发送过来的消息，并准备处理来自消费者的拉取请求。它还存储与消息相关的元数据，包括consumer group，消耗进度 offset 和 topic/queue 信息。

根据角色可以将 broker 分为两类：master、slave。 master broker 提供RW（读写）权限，slave broker 只读。master 又可以分为 `ASYNC_MASTER`，`SYNC_MASTER`。如果不能容忍消息的丢失，建议部署 `SYNC_MASTER` 加 `SLAVE`。如果对消息的丢失是可以容忍的，
但是你想broker总是可用的，你可以部署`ASYNC_MASTER` 加 `SLAVE`。

为了部署没有单点故障的高可用集群，应该部署多个 broker 集合。一个 broker 集合包含一个 brokerId 为0的master和若干个brokerId非0的slave。
在一个集合里的所有broker具有相同的brokerName。在严重的场景下，一个broker集合至少有两个broker，每个topic都存在于两个或更多broker里。

**broker 默认配置**

![broker默认配置](broker_conf.jpg)

属性名 | 默认值 | 描述
---|---|---
listenPort | 10911 | 客户端监听端口
nameserverAddr | null | NameServer 地址
brokerIP1 | 网络接口的InetAddress | 如果有多个地址，应配置
brokerName | null | broker 名称
brokerClusterName | DefaultCluster | broker 所属集群
brokerId | 0 | brokerId， 0 表示master，正整数表示slave
storePathCommitLog | $HOME/store/commitlog/ | commitlog 存储文件路径
storePathConsumerQueue | $HOME/store/consumequeue/ | 消费队列存储路径
mapedFileSizeCommitLog | 1024`*`1024`*`1024(1G) | commitlog 文件大小
deleteWhen | 04 | 何时删除超出保留时间的提交日志
fileReserverdTime | 48 | 在删除之前保留commitlog的小时数
brokerRole | ASYNC_MASTER | SYNC_MASTER/ASYNC_MASTER/SLVAE
flushDiskType | ASYNC_FLUSH | {SYNC_FLUSH/ASYNC_FLUSH}.在确认生产者之前，SYNC_FLUSH模式的broker将每个消息刷新到磁盘上。另一方面，ASYNC_FLUSH模式的broker利用了组提交，实现了更好的性能。

对于flushDiskType而言，建议使用 `ASYNC_FLUSH`, `SYNC_FLUSH` 代价很大而且会导致性能的流失。如果想要可靠，建议使用 `SYNC_MASTER` 加 `SLAVE`。

## broker 启动过程

broker的启动，由BrokerStartup处理，直接执行该类的main方法。broker 启动过程大致分为3步。

- 解析命令行参数，获取用户启动broker传过来的参数
- BrokerController 初始化
- BrokerController 启动

BrokerStartup.main()
```java
public static void main(String[] args) {
    start(createBrokerController(args));
}

public static BrokerController start(BrokerController controller) {
    try {
        controller.start();
        String tip = "The broker[" + controller.getBrokerConfig().getBrokerName() + ", "
            + controller.getBrokerAddr() + "] boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();

        if (null != controller.getBrokerConfig().getNamesrvAddr()) {
            tip += " and name server is " + controller.getBrokerConfig().getNamesrvAddr();
        }

        log.info(tip);
        return controller;
    } catch (Throwable e) {
        e.printStackTrace();
        System.exit(-1);
    }

    return null;
}
```

从上述代码看出，broker启动的核心是BrokerController。下面就从BrokerController的initialize和start方法剖析broker启动过程。

### BrokerController.initialize()

第一步，加载落盘的文件信息。
```java
result = result && this.topicConfigManager.load(); // 加载topic信息

result = result && this.consumerOffsetManager.load(); // 加载consumerOffset信息
result = result && this.subscriptionGroupManager.load(); // 加载订阅组信息

if (result) {
    try {
        this.messageStore =
            new DefaultMessageStore(this.messageStoreConfig, this.brokerStatsManager, this.messageArrivingListener,
                this.brokerConfig); 
        this.brokerStats = new BrokerStats((DefaultMessageStore) this.messageStore);
        //load plugin
        MessageStorePluginContext context = new MessageStorePluginContext(messageStoreConfig, brokerStatsManager, messageArrivingListener, brokerConfig);
        this.messageStore = MessageStoreFactory.build(context, this.messageStore);
    } catch (IOException e) {
        result = false;
        e.printStackTrace();
    }
}

result = result && this.messageStore.load(); // 加载磁盘里的消息，包括commitLog、consumerQueue、checkPoint、index等信息
```

broker落盘的数据保存目录结构，存储路径为 ${home}/store

![store文件夹](store_path.png)

第二步，实例化remotingServer和五个线程池。这五个线程池分别是：

- sendMessageExecutor: 处理消息发送的线程池
- pullMessageExecutor: 处理消息拉取的线程池
- adminBrokerExecutor: 管理员Broker线程池？？ 还没搞清楚这是干嘛的
- clientManageExecutor: 客户端管理线程池
- consumerManageExecutor: 消费管理线程池

第三步，注册时间处理单元--registerProcessor()

将以下处理器注册到remotingServer中。

- SendMessageProcessor: 处理消息的处理器，对应的线程池是 sendMessageExecutor
- PullMessageProcessor: 拉取消息的处理器，对应的线程池是 pullMessageExecutor
- QueryMessageProcessor: 获取消息的处理器，对应的线程池是 pullMessageExecutor
- ClientManageProcessor: 客户端管理的处理器，对应的线程池是 clientManageExecutor
- ConsumerManageProcessor: 消费管理的处理器，对应的线程池是 consumerManageExecutor
- EndTransactionProcessor: 事务处理器，对应的是线程池 sendMessageExecutor
- AdminBrokerProcessor: 管理broker的处理器，对应的线程池是 adminBrokerExecutor

注册处理器的实现：

```java
public void registerProcessor(int requestCode, NettyRequestProcessor processor, ExecutorService executor) {
    ExecutorService executorThis = executor;
    if (null == executor) {
        executorThis = this.publicExecutor;
    }

    Pair<NettyRequestProcessor, ExecutorService> pair = new Pair<NettyRequestProcessor, ExecutorService>(processor, executorThis);
    this.processorTable.put(requestCode, pair);
}
```

Pair对象类型与hashmap，key-value的数据格式，这里 key 为处理器对象，value 为对应的线程池。对应的processor处理不同的请求，不同的请求，有不同的请求码，最后，会将Pair对象放到processorTable（hashMap）中，key 为 requestCode，value 是 Pair。

到这里，broker 初始化完成。接下来就是 broker 的启动。

### Broker.start()

broker 的启动，具体实现在 BrokerController.start() 里。

```java
public void start() throws Exception {
    if (this.messageStore != null) {
        this.messageStore.start();
    }

    if (this.remotingServer != null) {
        this.remotingServer.start(); // remotingServer启动
    }

    if (this.fastRemotingServer != null) {
        this.fastRemotingServer.start();
    }

    if (this.brokerOuterAPI != null) {
        this.brokerOuterAPI.start(); // broker 和外界发生信息交流的api服务启动
    }

    if (this.pullRequestHoldService != null) {
        this.pullRequestHoldService.start(); // 处理拉取消息请求的服务启动
    }

    if (this.clientHousekeepingService != null) {
        this.clientHousekeepingService.start();
    }

    if (this.filterServerManager != null) {
        this.filterServerManager.start();
    }

    this.registerBrokerAll(true, false);

    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            try {
                BrokerController.this.registerBrokerAll(true, false);
            } catch (Throwable e) {
                log.error("registerBrokerAll Exception", e);
            }
        }
    }, 1000 * 10, 1000 * 30, TimeUnit.MILLISECONDS);

    if (this.brokerStatsManager != null) {
        this.brokerStatsManager.start();
    }

    if (this.brokerFastFailure != null) {
        this.brokerFastFailure.start();
    }
}
```

#### messageStore.start() 分析

messageStore 有两种实现，DefaultMessageStore 和 AbstractPluginMessageStore。这里看看 DefaultMessageStore。

**DefaultMessageStore.start()**

```java
public void start() throws Exception {
    this.flushConsumeQueueService.start();
    this.commitLog.start();
    this.storeStatsService.start();

    if (this.scheduleMessageService != null && SLAVE != messageStoreConfig.getBrokerRole()) {
        this.scheduleMessageService.start();
    }

    if (this.getMessageStoreConfig().isDuplicationEnable()) {
        this.reputMessageService.setReputFromOffset(this.commitLog.getConfirmOffset());
    } else {
        this.reputMessageService.setReputFromOffset(this.commitLog.getMaxOffset());
    }
    this.reputMessageService.start();

    this.haService.start();

    this.createTempFile(); // 创建临时文件 abort
    this.addScheduleTask(); // 添加定时任务
    this.shutdown = false;
}
```

**flushConsumerQueueService.start(): 启动消费队列刷盘服务**

默认每1000ms执行一次刷盘，每次默认从consumerQueue里取出两页数据刷入磁盘。 

**commitLog.start(): 启动commitLog刷盘服务**

这里的实现是基于FlushCommitLogService服务，它有两种子类，如果transientStorePoolEnable是开启的，并且刷盘方式是`ASYNC_FLUSH`，则具体实现是CommitRealTimeService服务，否则的具体是实现在FlushRealTimeService。CommitRealTimeService 和 FlushRealTimeService的区别: 一是刷盘间隔，前者为200ms, 后者是500ms；二是彻底刷盘时间间隔，前者200ms,后者为1000*10(10s)；三是刷入的目标，前者是将数据提交到FileChannel,后者是将数据写入到磁盘。

**storeStatsService.start(): 启动存储统计的服务**

默认每1000ms执行一次。

**scheduleMessageService.start(): 定时任务消息服务**

延时消息服务相关

**reputMessageService.start()**

从物理队列解析消息重新发送到逻辑队列

**haService.start(): 高可用服务**

同步双写，异步复制功能

### Broker 接收消息

在broker的初始化过程中，rocketmq 将SendMessageProcessor注册到了remotingServer，对应的线程池是 sendMessageExecutor，对应的RequestCode是SEND_MESSAGE=10;具体接收消息的逻辑在SendMessageProcessor.processRequest()方法中。

#### SendMessageProcessor.processRequest(ChannelHandlerContext ctx, RemotingCommand request);

```java
public RemotingCommand processRequest(ChannelHandlerContext ctx,
                    RemotingCommand request) throws RemotingCommandException {
    SendMessageContext mqtraceContext;
    switch (request.getCode()) {
        case RequestCode.CONSUMER_SEND_MSG_BACK:
            return this.consumerSendMsgBack(ctx, request);
        default:
            SendMessageRequestHeader requestHeader = parseRequestHeader(request); // 解析请求头
            if (requestHeader == null) {
                return null;
            }

            mqtraceContext = buildMsgContext(ctx, requestHeader);
            this.executeSendMessageHookBefore(ctx, request, mqtraceContext);

            RemotingCommand response;
            if (requestHeader.isBatch()) { // 如果是批量请求
                response = this.sendBatchMessage(ctx, request, mqtraceContext, requestHeader);
            } else {
                response = this.sendMessage(ctx, request, mqtraceContext, requestHeader); // 接收消息
            }

            this.executeSendMessageHookAfter(response, mqtraceContext);
            return response;
    }
}
```

#### message 落盘

```java
private RemotingCommand sendMessage(final ChannelHandlerContext ctx,
                                final RemotingCommand request,
                                final SendMessageContext sendMessageContext,
                                final SendMessageRequestHeader requestHeader) throws RemotingCommandException {

    final RemotingCommand response = RemotingCommand.createResponseCommand(SendMessageResponseHeader.class);
    final SendMessageResponseHeader responseHeader = (SendMessageResponseHeader)response.readCustomHeader();
    /**
    省略
    */
    } else {
        putMessageResult = this.brokerController.getMessageStore().putMessage(msgInner); // 落盘
    }

    return handlePutMessageResult(putMessageResult, response, request, msgInner, responseHeader, sendMessageContext, ctx, queueIdInt);

}
```

落盘的具体实现，在DefaultMessageStore中，是将数据写入到commitLog文件中。写入时，每次都取mappedFileQueue的mappedFiles的最后一个mappedFile，如果为空或者已经满了，就会新建一个mappedFile对象。

### Broker 处理消息拉取

在broker的初始化过程中，rocketmq 将PullMessageExecutor注册到了remotingServer，对应的线程池是 pullMessageExecutor，对应的RequestCode是PULL_MESSAGE=11;具体处理消息拉取的逻辑在PullMessageExecutor.processRequest()方法中。

```java
private RemotingCommand processRequest(final Channel channel, RemotingCommand request, boolean brokerAllowSuspend)
    throws RemotingCommandException {
    RemotingCommand response = RemotingCommand.createResponseCommand(PullMessageResponseHeader.class);
    final PullMessageResponseHeader responseHeader = (PullMessageResponseHeader) response.readCustomHeader();
    final PullMessageRequestHeader requestHeader =
        (PullMessageRequestHeader) request.decodeCommandCustomHeader(PullMessageRequestHeader.class);

    response.setOpaque(request.getOpaque());

    log.debug("receive PullMessage request command, {}", request);

    /**
    各种有效性校验
    */

    // 构建消息过滤器
    MessageFilter messageFilter;
    if (this.brokerController.getBrokerConfig().isFilterSupportRetry()) {
        messageFilter = new ExpressionForRetryMessageFilter(subscriptionData, consumerFilterData,
            this.brokerController.getConsumerFilterManager());
    } else {
        messageFilter = new ExpressionMessageFilter(subscriptionData, consumerFilterData,
            this.brokerController.getConsumerFilterManager());
    }

    // 从consumerQueue中获取消息
    final GetMessageResult getMessageResult =
        this.brokerController.getMessageStore().getMessage(requestHeader.getConsumerGroup(), requestHeader.getTopic(),
            requestHeader.getQueueId(), requestHeader.getQueueOffset(), requestHeader.getMaxMsgNums(), messageFilter);
    if (getMessageResult != null) {
        response.setRemark(getMessageResult.getStatus().name());
        responseHeader.setNextBeginOffset(getMessageResult.getNextBeginOffset());
        responseHeader.setMinOffset(getMessageResult.getMinOffset());
        responseHeader.setMaxOffset(getMessageResult.getMaxOffset());

        if (getMessageResult.isSuggestPullingFromSlave()) {
            responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getWhichBrokerWhenConsumeSlowly());
        } else {
            responseHeader.setSuggestWhichBrokerId(MixAll.MASTER_ID);
        }

        switch (this.brokerController.getMessageStoreConfig().getBrokerRole()) {
            case ASYNC_MASTER:
            case SYNC_MASTER:
                break;
            case SLAVE:
                if (!this.brokerController.getBrokerConfig().isSlaveReadEnable()) {
                    response.setCode(ResponseCode.PULL_RETRY_IMMEDIATELY);
                    responseHeader.setSuggestWhichBrokerId(MixAll.MASTER_ID);
                }
                break;
        }

        if (this.brokerController.getBrokerConfig().isSlaveReadEnable()) {
            // consume too slow ,redirect to another machine
            if (getMessageResult.isSuggestPullingFromSlave()) {
                responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getWhichBrokerWhenConsumeSlowly());
            }
            // consume ok
            else {
                responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getBrokerId());
            }
        } else {
            responseHeader.setSuggestWhichBrokerId(MixAll.MASTER_ID);
        }

        switch (getMessageResult.getStatus()) {
            case FOUND:
                response.setCode(ResponseCode.SUCCESS);
                break;
            case MESSAGE_WAS_REMOVING:
                response.setCode(ResponseCode.PULL_RETRY_IMMEDIATELY);
                break;
            case NO_MATCHED_LOGIC_QUEUE:
            case NO_MESSAGE_IN_QUEUE:
                if (0 != requestHeader.getQueueOffset()) {
                    response.setCode(ResponseCode.PULL_OFFSET_MOVED);

                    // XXX: warn and notify me
                    log.info("the broker store no queue data, fix the request offset {} to {}, Topic: {} QueueId: {} Consumer Group: {}",
                        requestHeader.getQueueOffset(),
                        getMessageResult.getNextBeginOffset(),
                        requestHeader.getTopic(),
                        requestHeader.getQueueId(),
                        requestHeader.getConsumerGroup()
                    );
                } else {
                    response.setCode(ResponseCode.PULL_NOT_FOUND);
                }
                break;
            case NO_MATCHED_MESSAGE:
                response.setCode(ResponseCode.PULL_RETRY_IMMEDIATELY);
                break;
            case OFFSET_FOUND_NULL:
                response.setCode(ResponseCode.PULL_NOT_FOUND);
                break;
            case OFFSET_OVERFLOW_BADLY:
                response.setCode(ResponseCode.PULL_OFFSET_MOVED);
                // XXX: warn and notify me
                log.info("the request offset: {} over flow badly, broker max offset: {}, consumer: {}",
                    requestHeader.getQueueOffset(), getMessageResult.getMaxOffset(), channel.remoteAddress());
                break;
            case OFFSET_OVERFLOW_ONE:
                response.setCode(ResponseCode.PULL_NOT_FOUND);
                break;
            case OFFSET_TOO_SMALL:
                response.setCode(ResponseCode.PULL_OFFSET_MOVED);
                log.info("the request offset too small. group={}, topic={}, requestOffset={}, brokerMinOffset={}, clientIp={}",
                    requestHeader.getConsumerGroup(), requestHeader.getTopic(), requestHeader.getQueueOffset(),
                    getMessageResult.getMinOffset(), channel.remoteAddress());
                break;
            default:
                assert false;
                break;
        }

        if (this.hasConsumeMessageHook()) {
            ConsumeMessageContext context = new ConsumeMessageContext();
            context.setConsumerGroup(requestHeader.getConsumerGroup());
            context.setTopic(requestHeader.getTopic());
            context.setQueueId(requestHeader.getQueueId());

            String owner = request.getExtFields().get(BrokerStatsManager.COMMERCIAL_OWNER);

            switch (response.getCode()) {
                case ResponseCode.SUCCESS:
                    int commercialBaseCount = brokerController.getBrokerConfig().getCommercialBaseCount();
                    int incValue = getMessageResult.getMsgCount4Commercial() * commercialBaseCount;

                    context.setCommercialRcvStats(BrokerStatsManager.StatsType.RCV_SUCCESS);
                    context.setCommercialRcvTimes(incValue);
                    context.setCommercialRcvSize(getMessageResult.getBufferTotalSize());
                    context.setCommercialOwner(owner);

                    break;
                case ResponseCode.PULL_NOT_FOUND:
                    if (!brokerAllowSuspend) {

                        context.setCommercialRcvStats(BrokerStatsManager.StatsType.RCV_EPOLLS);
                        context.setCommercialRcvTimes(1);
                        context.setCommercialOwner(owner);

                    }
                    break;
                case ResponseCode.PULL_RETRY_IMMEDIATELY:
                case ResponseCode.PULL_OFFSET_MOVED:
                    context.setCommercialRcvStats(BrokerStatsManager.StatsType.RCV_EPOLLS);
                    context.setCommercialRcvTimes(1);
                    context.setCommercialOwner(owner);
                    break;
                default:
                    assert false;
                    break;
            }

            this.executeConsumeMessageHookBefore(context);
        }

        switch (response.getCode()) {
            case ResponseCode.SUCCESS:

                this.brokerController.getBrokerStatsManager().incGroupGetNums(requestHeader.getConsumerGroup(), requestHeader.getTopic(),
                    getMessageResult.getMessageCount());

                this.brokerController.getBrokerStatsManager().incGroupGetSize(requestHeader.getConsumerGroup(), requestHeader.getTopic(),
                    getMessageResult.getBufferTotalSize());

                this.brokerController.getBrokerStatsManager().incBrokerGetNums(getMessageResult.getMessageCount());
                if (this.brokerController.getBrokerConfig().isTransferMsgByHeap()) {
                    final long beginTimeMills = this.brokerController.getMessageStore().now();
                    final byte[] r = this.readGetMessageResult(getMessageResult, requestHeader.getConsumerGroup(), requestHeader.getTopic(), requestHeader.getQueueId());
                    this.brokerController.getBrokerStatsManager().incGroupGetLatency(requestHeader.getConsumerGroup(),
                        requestHeader.getTopic(), requestHeader.getQueueId(),
                        (int) (this.brokerController.getMessageStore().now() - beginTimeMills));
                    response.setBody(r);
                } else {
                    try {
                        FileRegion fileRegion =
                            new ManyMessageTransfer(response.encodeHeader(getMessageResult.getBufferTotalSize()), getMessageResult);
                        channel.writeAndFlush(fileRegion).addListener(new ChannelFutureListener() {
                            @Override
                            public void operationComplete(ChannelFuture future) throws Exception {
                                getMessageResult.release();
                                if (!future.isSuccess()) {
                                    log.error("transfer many message by pagecache failed, {}", channel.remoteAddress(), future.cause());
                                }
                            }
                        });
                    } catch (Throwable e) {
                        log.error("transfer many message by pagecache exception", e);
                        getMessageResult.release();
                    }

                    response = null;
                }
                break;

        //当前没有消息，将本次拉取消息的请求挂起。在PullRequestHoldService中重新发起拉取请求。
            case ResponseCode.PULL_NOT_FOUND: 

                if (brokerAllowSuspend && hasSuspendFlag) {
                    long pollingTimeMills = suspendTimeoutMillisLong;
                    if (!this.brokerController.getBrokerConfig().isLongPollingEnable()) {
                        pollingTimeMills = this.brokerController.getBrokerConfig().getShortPollingTimeMills();
                    }

                    String topic = requestHeader.getTopic();
                    long offset = requestHeader.getQueueOffset();
                    int queueId = requestHeader.getQueueId();
                    PullRequest pullRequest = new PullRequest(request, channel, pollingTimeMills,
                        this.brokerController.getMessageStore().now(), offset, subscriptionData, messageFilter);
                    this.brokerController.getPullRequestHoldService().suspendPullRequest(topic, queueId, pullRequest);
                    response = null;
                    break;
                }

            case ResponseCode.PULL_RETRY_IMMEDIATELY:
                break;
            case ResponseCode.PULL_OFFSET_MOVED:
                if (this.brokerController.getMessageStoreConfig().getBrokerRole() != BrokerRole.SLAVE
                    || this.brokerController.getMessageStoreConfig().isOffsetCheckInSlave()) {
                    MessageQueue mq = new MessageQueue();
                    mq.setTopic(requestHeader.getTopic());
                    mq.setQueueId(requestHeader.getQueueId());
                    mq.setBrokerName(this.brokerController.getBrokerConfig().getBrokerName());

                    OffsetMovedEvent event = new OffsetMovedEvent();
                    event.setConsumerGroup(requestHeader.getConsumerGroup());
                    event.setMessageQueue(mq);
                    event.setOffsetRequest(requestHeader.getQueueOffset());
                    event.setOffsetNew(getMessageResult.getNextBeginOffset());
                    this.generateOffsetMovedEvent(event);
                    log.warn(
                        "PULL_OFFSET_MOVED:correction offset. topic={}, groupId={}, requestOffset={}, newOffset={}, suggestBrokerId={}",
                        requestHeader.getTopic(), requestHeader.getConsumerGroup(), event.getOffsetRequest(), event.getOffsetNew(),
                        responseHeader.getSuggestWhichBrokerId());
                } else {
                    responseHeader.setSuggestWhichBrokerId(subscriptionGroupConfig.getBrokerId());
                    response.setCode(ResponseCode.PULL_RETRY_IMMEDIATELY);
                    log.warn("PULL_OFFSET_MOVED:none correction. topic={}, groupId={}, requestOffset={}, suggestBrokerId={}",
                        requestHeader.getTopic(), requestHeader.getConsumerGroup(), requestHeader.getQueueOffset(),
                        responseHeader.getSuggestWhichBrokerId());
                }

                break;
            default:
                assert false;
        }
    } else {
        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark("store getMessage return null");
    }

    boolean storeOffsetEnable = brokerAllowSuspend;
    storeOffsetEnable = storeOffsetEnable && hasCommitOffsetFlag;
    storeOffsetEnable = storeOffsetEnable
        && this.brokerController.getMessageStoreConfig().getBrokerRole() != BrokerRole.SLAVE;
    if (storeOffsetEnable) {
        this.brokerController.getConsumerOffsetManager().commitOffset(RemotingHelper.parseChannelRemoteAddr(channel),
            requestHeader.getConsumerGroup(), requestHeader.getTopic(), requestHeader.getQueueId(), requestHeader.getCommitOffset());
    }
    return response;
}
```

#### 拉取请求的挂起以及处理

broker处理消息的拉取，是基于长轮询的策略。结合上面说到消息的拉取，如果当前拉取请求没有拿到数据，当前请求将会被挂起。此时，PullRequestHoldService服务会将当前这个请求存入内存--key为`topic@queueId`，value为PullRequest的map中。当PullRequestHoldService启动后，会从缓存中取出被挂起的请求，基于长轮询的策略，请求会被挂起1000ms*5，这样可以避免客户端向服务端发送无效的请求，_当broker检测到有新的消息到达时，会立即发出通知_（调用PullRequestHoldService的notifyMessageArriving方法）。然后PullRequestHoldService服务将这个请求再次发送给PullMessageProcessor处理器，进行处理。

```java
public void run() {
    log.info("{} service started", this.getServiceName());
    while (!this.isStopped()) {
        try {
            if (this.brokerController.getBrokerConfig().isLongPollingEnable()) {
                this.waitForRunning(5 * 1000); // 长轮询，等待5秒
            } else {
                this.waitForRunning(this.brokerController.getBrokerConfig().getShortPollingTimeMills());
            }

            long beginLockTimestamp = this.systemClock.now();
            this.checkHoldRequest(); // 当前时候可以将请求重发
            long costTime = this.systemClock.now() - beginLockTimestamp;
            if (costTime > 5 * 1000) {
                log.info("[NOTIFYME] check hold request cost {} ms.", costTime);
            }
        } catch (Throwable e) {
            log.warn(this.getServiceName() + " service has exception. ", e);
        }
    }

    log.info("{} service end", this.getServiceName());
}
private void checkHoldRequest() {
    for (String key : this.pullRequestTable.keySet()) {
        String[] kArray = key.split(TOPIC_QUEUEID_SEPARATOR);
        if (2 == kArray.length) {
            String topic = kArray[0];
            int queueId = Integer.parseInt(kArray[1]);
            final long offset = this.brokerController.getMessageStore().getMaxOffsetInQueue(topic, queueId);
            try {
                // 通知新消息到达
                this.notifyMessageArriving(topic, queueId, offset);
            } catch (Throwable e) {
                log.error("check hold request failed. topic={}, queueId={}", topic, queueId, e);
            }
        }
    }
}
public void notifyMessageArriving(final String topic, final int queueId, final long maxOffset, final Long tagsCode,
    long msgStoreTime, byte[] filterBitMap, Map<String, String> properties) {
    String key = this.buildKey(topic, queueId);
    ManyPullRequest mpr = this.pullRequestTable.get(key);
    if (mpr != null) {
        List<PullRequest> requestList = mpr.cloneListAndClear();
        if (requestList != null) {
            List<PullRequest> replayList = new ArrayList<PullRequest>();

            for (PullRequest request : requestList) {
                long newestOffset = maxOffset;
                if (newestOffset <= request.getPullFromThisOffset()) {
                    newestOffset = this.brokerController.getMessageStore().getMaxOffsetInQueue(topic, queueId);
                }

                if (newestOffset > request.getPullFromThisOffset()) {
                    boolean match = request.getMessageFilter().isMatchedByConsumeQueue(tagsCode,
                        new ConsumeQueueExt.CqExtUnit(tagsCode, msgStoreTime, filterBitMap));
                    // match by bit map, need eval again when properties is not null.
                    if (match && properties != null) {
                        match = request.getMessageFilter().isMatchedByCommitLog(null, properties);
                    }

                    if (match) {
                        try { // 有新消息，将请求重新发送给PullMessageProcessor，进行处理
                            this.brokerController.getPullMessageProcessor().executeRequestWhenWakeup(request.getClientChannel(),
                                request.getRequestCommand());
                        } catch (Throwable e) {
                            log.error("execute request when wakeup failed.", e);
                        }
                        continue;
                    }
                }
                // 已经过了挂起的时间，请求重发
                if (System.currentTimeMillis() >= (request.getSuspendTimestamp() + request.getTimeoutMillis())) {
                    try {
                        this.brokerController.getPullMessageProcessor().executeRequestWhenWakeup(request.getClientChannel(),
                            request.getRequestCommand());
                    } catch (Throwable e) {
                        log.error("execute request when wakeup failed.", e);
                    }
                    continue;
                }

                replayList.add(request);
            }

            if (!replayList.isEmpty()) {
                mpr.addPullRequest(replayList);
            }
        }
    }
}
```

上面说到，第一次没有拉取到消息，当前请求会被挂起，等待5s后在进行请求。那么在等待过程中，如果有新的消息到达，是否需要等待线程被重新唤醒再执行呢？答案是，不需要，如果有新的消息到达，PullRequestHoldService会接收到通知，会立即将当前请求立即执行。Broker在初始化DefaultMessageStore对象时，会给DefaultMessageStore对象的属性messageArrivingListener实例化一个监听器（NotifyMessageArrivingListener）。DefaultMessageStore启动是，会启动一个reputMessageService服务，该服务每一毫秒执行一次（几乎是实时的），执行一个doput操作。

```java
private void doReput() {
    for (boolean doNext = true; this.isCommitLogAvailable() && doNext; ) {

        if (DefaultMessageStore.this.getMessageStoreConfig().isDuplicationEnable()
            && this.reputFromOffset >= DefaultMessageStore.this.getConfirmOffset()) {
            break;
        }

        SelectMappedBufferResult result = DefaultMessageStore.this.commitLog.getData(reputFromOffset);
        if (result != null) {
            try {
                this.reputFromOffset = result.getStartOffset();

                for (int readSize = 0; readSize < result.getSize() && doNext; ) {
                    DispatchRequest dispatchRequest =
                        DefaultMessageStore.this.commitLog.checkMessageAndReturnSize(result.getByteBuffer(), false, false);
                    int size = dispatchRequest.getMsgSize();

                    if (dispatchRequest.isSuccess()) {
                        if (size > 0) {
                            DefaultMessageStore.this.doDispatch(dispatchRequest);

                            if (BrokerRole.SLAVE != DefaultMessageStore.this.getMessageStoreConfig().getBrokerRole()
                                && DefaultMessageStore.this.brokerConfig.isLongPollingEnable()) { // 出发监听器，表示有新的消息到达
                                DefaultMessageStore.this.messageArrivingListener.arriving(dispatchRequest.getTopic(),
                                    dispatchRequest.getQueueId(), dispatchRequest.getConsumeQueueOffset() + 1,
                                    dispatchRequest.getTagsCode(), dispatchRequest.getStoreTimestamp(),
                                    dispatchRequest.getBitMap(), dispatchRequest.getPropertiesMap());
                            }

                            this.reputFromOffset += size;
                            readSize += size;
                            if (DefaultMessageStore.this.getMessageStoreConfig().getBrokerRole() == BrokerRole.SLAVE) {
                                DefaultMessageStore.this.storeStatsService
                                    .getSinglePutMessageTopicTimesTotal(dispatchRequest.getTopic()).incrementAndGet();
                                DefaultMessageStore.this.storeStatsService
                                    .getSinglePutMessageTopicSizeTotal(dispatchRequest.getTopic())
                                    .addAndGet(dispatchRequest.getMsgSize());
                            }
                        } else if (size == 0) {
                            this.reputFromOffset = DefaultMessageStore.this.commitLog.rollNextFile(this.reputFromOffset);
                            readSize = result.getSize();
                        }
                    } else if (!dispatchRequest.isSuccess()) {

                        if (size > 0) {
                            log.error("[BUG]read total count not equals msg total size. reputFromOffset={}", reputFromOffset);
                            this.reputFromOffset += size;
                        } else {
                            doNext = false;
                            if (DefaultMessageStore.this.brokerConfig.getBrokerId() == MixAll.MASTER_ID) {
                                log.error("[BUG]the master dispatch message to consume queue error, COMMITLOG OFFSET: {}",
                                    this.reputFromOffset);

                                this.reputFromOffset += result.getSize() - readSize;
                            }
                        }
                    }
                }
            } finally {
                result.release();
            }
        } else {
            doNext = false;
        }
    }
}

```


综上，就是broker从初始化->启动->处理消息的发送->消息拉取等过程的分析。还有其它很多功能，没有详细去看。

