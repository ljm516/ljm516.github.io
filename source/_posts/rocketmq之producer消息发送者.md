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

producer 的具体实现在DefaultMQProducerImpl类中。启动过程时序图：

![启动过程](producer_start.jpg)

- 检查配置，主要是检测producerGroup。
- 通过MQClientManager创建Producer客户端实例。
- 注册producer实例，将producer存入到以groupName为key，producer为value的map中。
- producer实例启动
- 发送心跳

### mQClientFactory.start()

producer实例启动。

```java
public void start() throws MQClientException {
    synchronized (this) {
        switch (this.serviceState) {
            case CREATE_JUST:
                this.serviceState = ServiceState.START_FAILED;
                // If not specified,looking address from name server
                if (null == this.clientConfig.getNamesrvAddr()) { 
                    this.mQClientAPIImpl.fetchNameServerAddr();
                }
                // Start request-response channel
                this.mQClientAPIImpl.start(); // 启动mq客户端实例
                // Start various schedule tasks
                this.startScheduledTask(); // 启动各类定时任务
                // Start pull service
                this.pullMessageService.start(); // 启动拉取消息的服务
                // Start rebalance service
                this.rebalanceService.start(); // 负载均衡服务启动
                // Start push service
                this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
                log.info("the client factory [{}] start OK", this.clientId);
                this.serviceState = ServiceState.RUNNING;
                break;
            case RUNNING:
                break;
            case SHUTDOWN_ALREADY:
                break;
            case START_FAILED:
                throw new MQClientException("The Factory object[" + this.getClientId() + "] has been created before, and failed.", null);
            default:
                break;
        }
    }
}
```

**fetch namesrv**
如果producer显示的指定（编码方式）NameSrv地址，producer实例会默认从系统参数中回去NameSrv地址，如果还是没有获取到，会通过发起http请求`http://jmenv.tbsite.net:8080/rocketmq/nsaddr`获取。

**MQInstance.start()**
mQClientFactory.start() 启动一个remotingClient，进行和remotingServer进行交互工作。

**startScheduledTask()**
启动定时任务: 
- 定时获取NameSrv地址；
- 从NameSrv更新Topic路由信息；
- 剔除下线的Broker、向broker发送心跳；
- 持久化ConsumerOffset；调整线程池。

**pullMessageService.start()**
和consumer拉取消息有关，在consumer解析中讲解。

**rebalanceService.start()**
和consumer消息负载均衡有关，在consumer解析中讲解。

#### producer启动的疑问？
1. MQClientInstance.start()方法中最后this.defaultMQProducer.getDefaultMQProducerImpl().start(false);反过来又去调用一次DefaultMQProducerImpl.start()啥也没干，就调用了一次心跳服务. 
2. producer,consumer都是一个mqClientInstance,启动时都会去调用MQClientInstance.start().该方法启动的server大部分都是和consumer相关,而在producer启动也在调用。

## producer 发送消息

producer 发送消息过程，基于同步模式。发送消息的具体实现在DefaultMQProducerImpl.sendDefaultImpl()方法中。

```java
private SendResult sendDefaultImpl(
    Message msg,
    final CommunicationMode communicationMode,
    final SendCallback sendCallback,
    final long timeout
) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    this.makeSureStateOK();
    Validators.checkMessage(msg, this.defaultMQProducer); // 检测消息有效性

    final long invokeID = random.nextLong();
    longtam beginTimespFirst = System.currentTimeMillis(); // 记录开始时间
    long beginTimestampPrev = beginTimestampFirst; 
    long endTimestamp = beginTimestampFirst; // 记录结束时间
    TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic()); // 获取topic路由信息 1.
    if (topicPublishInfo != null && topicPublishInfo.ok()) {
        boolean callTimeout = false;
        MessageQueue mq = null;
        Exception exception = null;
        SendResult sendResult = null;
        int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1; // 如果是同步方式，默认重试3次
        int times = 0;
        String[] brokersSent = new String[timesTotal];
        for (; times < timesTotal; times++) {
            String lastBrokerName = null == mq ? null : mq.getBrokerName();
            MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName); // 选择要发送到的MessageQueue 2.
            if (mqSelected != null) {
                mq = mqSelected;
                brokersSent[times] = mq.getBrokerName();
                try {
                    beginTimestampPrev = System.currentTimeMillis();
                    long costTime = beginTimestampPrev - beginTimestampFirst;
                    if (timeout < costTime) {
                        callTimeout = true;
                        break;
                    }

                    sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime); // 发送消息的核心实现 3.
                    endTimestamp = System.currentTimeMillis();
                    this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
                    switch (communicationMode) {
                        case ASYNC:
                            return null;
                        case ONEWAY:
                            return null;
                        case SYNC:
                            if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
                                if (this.defaultMQProducer.isRetryAnotherBrokerWhenNotStoreOK()) {
                                    continue;
                                }
                            }

                            return sendResult;
                        default:
                            break;
                    }
                } 
                /**
                catch exception
                */
            } else {
                break;
            }
        }

        if (sendResult != null) {
            return sendResult;
        }

        String info = String.format("Send [%d] times, still failed, cost [%d]ms, Topic: %s, BrokersSent: %s",
            times,
            System.currentTimeMillis() - beginTimestampFirst,
            msg.getTopic(),
            Arrays.toString(brokersSent));

        info += FAQUrl.suggestTodo(FAQUrl.SEND_MSG_FAILED);

        MQClientException mqClientException = new MQClientException(info, exception);
        if (callTimeout) {
            throw new RemotingTooMuchRequestException("sendDefaultImpl call timeout");
        }

        if (exception instanceof MQBrokerException) {
            mqClientException.setResponseCode(((MQBrokerException) exception).getResponseCode());
        } else if (exception instanceof RemotingConnectException) {
            mqClientException.setResponseCode(ClientErrorCode.CONNECT_BROKER_EXCEPTION);
        } else if (exception instanceof RemotingTimeoutException) {
            mqClientException.setResponseCode(ClientErrorCode.ACCESS_BROKER_TIMEOUT);
        } else if (exception instanceof MQClientException) {
            mqClientException.setResponseCode(ClientErrorCode.BROKER_NOT_EXIST_EXCEPTION);
        }

        throw mqClientException;
    }
```

1. producer发送消息，首先要根据topic找到topicPublishInfo。看一下TopicPublishInfo类的结构：

**TopicPublishInfo.java 数据结构**
    - boolean orderTopic // 是否顺序的
    - boolean haveTopicRouterInfo // 是否可用
    - List<MessageQueue> messageQueueList // messageQueue 信息
        + String topic // topic 名称
        + String brokerName // broker名称
        + int queueId // 队列Id
    - TopicRouteDate topicRouteData // topic路由信息
        + String orderTopicConf // 顺序topic的配置
        + List<QueueData> queueDatas // 队列数据
            * String brokerName; // broker名称
            * int readQueueNums; // 读队列数
            * int writeQueueNums; // 写队列数
            * int perm; // 权限级别
            * int topicSynFlag; // topic同步标记
        + List<BrokerData> brokerDatas // broker 数据
            * String cluster; // 所属集群
            * String brokerName; // broker名称
            * HashMap<Long/* brokerId */, String/* broker address */> brokerAddrs;  // broker 地址映射
        + HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable

获取TopicPublishInfo
```java
private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
    TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
    if (null == topicPublishInfo || !topicPublishInfo.ok()) {
        this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
    }

    if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {
        return topicPublishInfo;
    } else {
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
        return topicPublishInfo;
    }
}
```

查找topic路由信息时，首先是从缓存中，如果获取不到或者不可用，就会新建一个，然后更新NameSrv中的Topic路由信息。

2. 根据topicPublishInfo和brokerName 获取要发送到的MessageQueue

// 选择要发送到的MessageQueue
```java
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
    if (this.sendLatencyFaultEnable) {
        try {
            int index = tpInfo.getSendWhichQueue().getAndIncrement();
            for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {
                int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();
                if (pos < 0)
                    pos = 0;
                MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
                if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) {
                    if (null == lastBrokerName || mq.getBrokerName().equals(lastBrokerName))
                        return mq;
                }
            }

            final String notBestBroker = latencyFaultTolerance.pickOneAtLeast();
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

    return tpInfo.selectOneMessageQueue(lastBrokerName);
}
```
    

3. 发送消息的核心实现sendKernelImpl
 在发送消息之前，会将消息体进行压缩。然后构建请求头，将必须要的信息加入到请求头中（SendMessageRequestHeader）。

```java
// 压缩
boolean msgBodyCompressed = false;
if (this.tryToCompressMessage(msg)) {
    sysFlag |= MessageSysFlag.COMPRESSED_FLAG;
    msgBodyCompressed = true;
}

// 请求头
SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();
requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
requestHeader.setTopic(msg.getTopic());
requestHeader.setDefaultTopic(this.defaultMQProducer.getCreateTopicKey());
requestHeader.setDefaultTopicQueueNums(this.defaultMQProducer.getDefaultTopicQueueNums());
requestHeader.setQueueId(mq.getQueueId());
requestHeader.setSysFlag(sysFlag);
requestHeader.setBornTimestamp(System.currentTimeMillis());
requestHeader.setFlag(msg.getFlag());
requestHeader.setProperties(MessageDecoder.messageProperties2String(msg.getProperties()));
requestHeader.setReconsumeTimes(0);
requestHeader.setUnitMode(this.isUnitMode());
requestHeader.setBatch(msg instanceof MessageBatch);
if (requestHeader.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
    String reconsumeTimes = MessageAccessor.getReconsumeTime(msg);
    if (reconsumeTimes != null) {
        requestHeader.setReconsumeTimes(Integer.valueOf(reconsumeTimes));
        MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_RECONSUME_TIME);
    }

    String maxReconsumeTimes = MessageAccessor.getMaxReconsumeTimes(msg);
    if (maxReconsumeTimes != null) {
        requestHeader.setMaxReconsumeTimes(Integer.valueOf(maxReconsumeTimes));
        MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_MAX_RECONSUME_TIMES);
    }
}
 ```

rocketmq的通信模块是基于netty实现的，客户端给服务端发送请求，不同的请求都会带上不同的请求码，不同的请求在broker中也是通过不同的处理器（Processor）处理的。这里发送消息的REQUEST_CODE=SEND_MESSAGE。

```java
public RemotingCommand invokeSync(String addr, final RemotingCommand request, long timeoutMillis)
    throws InterruptedException, RemotingConnectException, RemotingSendRequestException, RemotingTimeoutException {
    long beginStartTime = System.currentTimeMillis();
    final Channel channel = this.getAndCreateChannel(addr);
    if (channel != null && channel.isActive()) {
        try {
            if (this.rpcHook != null) {
                this.rpcHook.doBeforeRequest(addr, request);
            }
            long costTime = System.currentTimeMillis() - beginStartTime;
            if (timeoutMillis < costTime) {
                throw new RemotingTimeoutException("invokeSync call timeout");
            }
            RemotingCommand response = this.invokeSyncImpl(channel, request, timeoutMillis - costTime);
            if (this.rpcHook != null) {
                this.rpcHook.doAfterResponse(RemotingHelper.parseChannelRemoteAddr(channel), request, response);
            }
            return response;
        } catch (RemotingSendRequestException e) {
            log.warn("invokeSync: send request exception, so close the channel[{}]", addr);
            this.closeChannel(addr, channel);
            throw e;
        } catch (RemotingTimeoutException e) {
            if (nettyClientConfig.isClientCloseSocketIfTimeout()) {
                this.closeChannel(addr, channel);
                log.warn("invokeSync: close socket because of timeout, {}ms, {}", timeoutMillis, addr);
            }
            log.warn("invokeSync: wait response timeout exception, the channel[{}]", addr);
            throw e;
        }
    } else {
        this.closeChannel(addr, channel);
        throw new RemotingConnectException(addr);
    }
}
```



