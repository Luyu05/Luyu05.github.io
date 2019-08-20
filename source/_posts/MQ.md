---
title: MQ-入门
date: 2018-09-27 20:23:58
tags: 好文收集
categories: Java中间件
---
Outline
1.使用场景
2.如何保证消息可靠到达
3.如何保证幂等
4.如何削峰填谷

<!-- more -->
## <font color=green>使用场景</font>
<code>不适用的场景：</code>
1）调用方实时依赖结果的业务场景

<code>适用的场景：</code>
1）有执行顺序的job，可以通过cron执行第一个job，当第一个job执行成功后通过mq通知第二个job，依次类推
2）上游不关心执行结果，以预约场景为例，用户预约成功后对于后续的一些处理过程（比如发送消息通知商户等）并不关心，这种情况很适合采用mq和预约行为代码解耦。而且当需要增加一些其他后处理行为的时候，不需要修改上游代码。
## <font color=green>MQ核心架构</font>
<code>发送方：</code>
1）业务调用方
2）MQ-client-sender（核心api：sendMsg(),sendCallback()）

<code>MQ集群：</code>
1）MQ-server
2）zk
3）db
4）管理后台web

<code>接收方：</code>
1）业务接收方
2）MQ-client-reciever（核心api：revcCallback(),sendAck()）

## <font color=green>MQ运转流程</font>
<img src='001.jpg'>
<code>MQ消息投递上半场，MQ-client-sender到MQ-server流程见上图：</code>

（1）MQ-client将消息发送给MQ-server（此时业务方调用的是API：SendMsg）

（2）MQ-server将消息落地，落地后即为发送成功

（3）MQ-server将应答发送给MQ-client（此时回调业务方是API：SendCallback）

 

<code>MQ消息投递下半场，MQ-server到MQ-client-receiver流程见上图4-6：</code>

（4）MQ-server将消息发送给MQ-client（此时回调业务方是API：RecvCallback）

（5）MQ-client回复应答给MQ-server（此时业务方主动调用API：SendAck）

（6）MQ-server收到ack，将之前已经落地的消息删除，完成消息的可靠投递

## <font color=green>如何保证消息可靠到达</font>
### 机制
1）消息落地
2）消息超时重传、消息确认

### 具体措施
<code>上半场的超时与重传</code>

MQ上半场的1或者2或者3如果丢失或者超时，MQ-client-sender内的timer会重发消息，直到期望收到3，如果重传N次后还未收到，则SendCallback回调发送失败，需要注意的是，这个过程中MQ-server可能会收到同一条消息的多次重发。

 
<code>下半场的超时与重传</code>

MQ下半场的4或者5或者6如果丢失或者超时，MQ-server内的timer会重发消息，直到收到5并且成功执行6，这个过程可能会重发很多次消息，一般采用指数退避的策略，先隔x秒重发，2x秒重发，4x秒重发，以此类推，需要注意的是，这个过程中MQ-client-receiver也可能会收到同一条消息的多次重发。
## <font color=green>如何保证幂等</font>
<img src='001.jpg'>
### 上半场幂等的设计
此时重发是MQ-client发起的，消息的处理是MQ-server，为了避免步骤2落地重复的消息，对每条消息，MQ系统内部必须生成一个inner-msg-id(该id在MQ-client端生成，当需要重试的时候使用同样的消息Id，而不要在server端自动生成消息)，作为去重和幂等的依据，这个内部消息ID的特性是：

（1）全局唯一

（2）MQ生成，具备业务无关性，对消息发送方和消息接收方屏蔽

Ps：上半场的幂等不涉及到调用方的工作，主要保证不会落地重复
### 下半场幂等的设计
此时重发是MQ-server发起的，消息的处理是消息消费业务方，消息重发势必导致业务方重复消费（上例中的一次付款，重复扣减库存），为了保证业务幂等性，业务消息体中，必须有一个biz-id，作为去重和幂等的依据，这个业务ID的特性是：

（1）对于同一个业务场景，全局唯一

（2）由业务消息发送方生成，业务相关，对MQ透明

（3）由业务消息消费方负责判重，以保证幂等

比如，代码中对biz-id进行判断，如果处理过了直接return，以保证幂等

Ps：下半场的幂等涉及到消费方的业务逻辑的改动


## <font color=green>MQ削峰填谷</font>
传统的rpc框架，服务提供方会有限流以及鉴权措施
对于通过mq传递引用的场景，如何防止流量过大造成下游服务雪崩？

1）原来，MQ-client-reciver除了推送模式，还有一种拉模式，reciver每隔一定时间从server拉取一定的消息，以实现流量控制，保护自身。这个是MQ提供的通用功能，无须上下游修改代码。

2）于此同时，对于批量拉取的消息，消费放最好能够做到批量消费，这需要对原有的1对1的代码进行修改，以实现批量处理


## <font color=green>如何保证业务操作和消息发送的一致性</font>
正向：
1）业务将消息投递到消息中间件
2）中间件收到消息后，将消息存入db，标记消息状态为待处理
3）中间件返回消息处理的结果（入库结果）
4）业务收到入库结果，如果成功执行业务操作；反之，放弃业务处理，结束
5）业务操作完成，将业务操作结果发送到中间件
6）如果业务成功，更新消息状态为待发送，并进行消息投递；如果失败，删除db中的消息，结束

问题：可能出现的问题，全流程执行后（可能有fail）消息仍然处于待处理状态，此时需要反向操作

反向：
1）中间件主动询问业务执行结果
2）检查业务执行结果
3）如果成功，那么更新消息为待发送；反之删除


原文链接：https://mp.weixin.qq.com/s/CIPosICgva9haqstMDIHag