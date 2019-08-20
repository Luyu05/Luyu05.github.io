---
title: ZAB协议
date: 2018-05-24 21:03:44
tags: Zookeeper
categories: Java中间件
---
zk本身提供分布式系统的协调服务；为了防止zk本身挂掉影响整个分布式集群，引入了ZAB协议
ZAB协议是zookeeper中专门设计的一种支持崩溃恢复的原子广播协议。
Ps:这篇文章必须加精
<!-- more -->
# 概述
<font color="red">消息广播<font/>

Ps：以和zk集群follower相连接的client发送一个写请求为例
1. 客户端提交事务 
2. 服务器生成与之对应的事务Proposal 
3. 并将该Proposal（提议）发给所有的Follower机器 
4. Follower收到Proposal后，将其以事务日志的形式写入到本地磁盘中 
5. 写入成功后反馈给Leader一个ACK 
6. Leader收到Quorum的ACK后，自身先完成对事务的提交 
7. 之后广播Commit消息给所有Follower服务器 
8. Follower执行事务的提交 

<font color="red">崩溃恢复<font/>

Leader选举算法：确保提交已经被旧Leader提交的事务Proposal，同时丢弃已经被旧Leader跳过的事务Proposal 
选举算法：保证新选举出来的Leader服务器拥有集群中所有机器最高编号（ZXID最大）的事务Proposal 
Ps: 
ZXID（事务的id）：低32位是简单的计数器，客户端每来一个事务请求，服务器产生一个新的事务的时候，都会对计数器加1； 
而高32位则代表了Leader周期epoch编号（朝代，每当新Leader产生就要++)   

---
# 详解
ZAB协议的三个阶段：
1. **发现(elect leader)**
    * Follower将自己最后接收的Proposal的epoch值发送给准Leader 
    * 接收到来自过半Follower的epoch后，会从这些epoch中选择最大的epoch并++，再发送给过半的Follower 
    * Follower收到新的epoch，检测最后处理过得proposal的epoch，如果小于新的epoch，那么更新为新的epoch，同时向准Leader发送Ack消息（包含当前的epoch以及历史处理过得Proposal的集合 ）
    * 准Leader收到过半Ack后，会从这过半的Ack中选择epoch最大的（如果相等选zxid最大的），将其历史Proposal的集合置为当前的初始化事务集合 
2. **同步** 
    * 准Leader会发送消息给Quorum中的Follower（消息内容=新的epoch+准Leader初始化的历史Proposal集合）
    * Follower接收到消息后，如果发现自己的epoch与准Leader的不相等，那么无法同步（不在一个朝代）；反之，Follower会接受并处理每一个Proposal，并发送反馈给Leader
    * 当准Leader收到过半Follower的反馈后，会向Follower发送Commit消息
    * Follower收到Commit后会依次处理并提交准Leader发来的且未处理过的事务
    Ps完成该步骤后，准Leader才真正称为Leader
3. **广播**
    * 客户端提交事务 
    * 服务器生成与之对应的事务Proposal 
    * 并将该Proposal（提议）发给所有的Follower机器 
    * Follower收到Proposal后，将其以事务日志的形式写入到本地磁盘中 
    * 写入成功后反馈给Leader一个ACK 
    * Leader收到Quorum的ACK后，自身先完成对事务的提交 
    * 之后广播Commit消息给所有Follower服务器 
    * Follower执行事务的提交 

在ZAB协议中，每一个zk节点都有可能处于以下三个状态：
* LOOKING：Leader选举阶段
* FOLLOWING：Follower服务器和Leader保持同步状态
* LEADING：Leader服务器作为Leader领导Follower状态

且每一个节点，都会循环执行这三个阶段，发现、同步、广播
Ps: Leader和Follower之间保持一个tcp长连接 ，如果在指定的超时时间内Leader无法从过半Follower获取到心跳包，那么Leader就结束对当前"epoch"的领导 其他的Follower会转换到LOOKING状态开始新的一轮选举

---
# 延伸思考
1.准Leader如何选举？
答：投票选举，每个服务器都会生成投票信息（myid,ZXID）,首先投给自己，之后再广播给集群中的其他机器，集群中的服务器接收到投票后会判断收到的票的有效性（包括是否是本轮投票，是否来自LOOKING状态的服务器），接下来将别人的票和自己的票做PK，优先PK ZXID，如果一样那么PK myid，将自己的票更新为PK胜利方的信息，并再次将票广播给其他机器，每次投票后，服务器都会统计所有投票，判断是否有过半的机器收到相同的投票信息 Ps myid并不重要，可能仅仅是代表一个序号，只要不重复就ok

2.如何保证不会丢失已经提交的提议（指的是整个集群）？
答：这个是相当的有意思，假设所有子民{0，1，2，3，4，5}，第一个朝代国王是0，子民是{1，2，3}，Ps成立朝代的标准是子民要超过半数，在某个提议整个朝代commit后（假设当前zxid=3），国王生病，这时候要更换新的国王，假设是5，而新的朝代子民也要超过半数，那么这就意味着新的王国内部必然有前朝子民，假设是{3，4，1}，新的国王会更新朝代epoch+1，并从当前子民中找到epoch最大或者epoch相等&zxid最大的（这种方式可以防止"复辟"，即小epoch大zxid当选leader），将其提交过的事务更新为自己做过的事务，并将自己的zxid置为0，然后勒令子民们和自己做过的事务进行同步，哈哈，这样就保证了朝代更替不会导致已经被提交的事务丢失~
Ps 这里说的zxid指的是zxid的低32位

---
# 总结
ZooKeeper集群：
1）读写数据库
2）保证一致性的集群
3）可以感知客户端的变化
4）主要解决分布式应用中经常遇到的一些<code>数据管理问题</code>

Ps:分布式系统的节点可以通过第三方节点进行数据共享，而这个第三方节点（zk）肯定也是由集群构成的（不然会有单点问题），所以第三方节点间也要通过协议保证（ZAB协议）数据的一致性