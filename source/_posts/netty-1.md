---
title: netty外层
date: 2018-10-15 21:48:38
tags: I/O
categories: I/O
---
总有一天我会把netty彻底搞清楚和线程池一样
<!-- more -->
## ByteBuf
### 原生的缺点
先说nio提供的ByteBuffer的缺点（结合例子）
```java
private static ByteBuffer bb = ByteBuffer.allocate(1024);
SocketChannel sc = (SocketChannel)sk.channel();
bb.clear();
while (sc.read(bb) > 0)
{
    bb.flip();
    while (bb.hasRemaining())
{
        System.out.print((char)bb.get());
    }
    System.out.println();
    bb.clear();
}
```
（1）ByteBuffer长度固定，一旦分配完成容量不能动态扩展和收缩
（2）只有一个标识位置的指针position，读写的时候要手工调用flip()

### ByteBuf的好处
（1）提供了readerIndex和writerIndex两个指针，写的时候writerIndex不断增加，读的时候readerIndex不断增加（但是不能超过writerIndex）
（2）屏蔽扩展细节，实现动态扩展
（3）discardReadBytes操作，将0-readerIndex范围的数据回收并续到capacity后面（目的是尽可能地重用缓冲区，但是因为改操作会发生字节数组的内存复制所以会影响性能）

## Channel功能说明
（1）channel是netty网络操作抽象类，包括网络的读、写，客户端发起连接，主动关闭连接，获取通信双方的网络地址等。
（2）channel需要注册到EventLoop的多路复用器上，用于处理IO事件，通过channel的eventLoop方法可以获取注册的EventLoop
（3）EventLoop的本质就是处理网络读写事件的Reactor线程
（4）在netty中每个channel对应一个物理连接，每个连接都由自己的tcp参数配置（tcp缓冲区大小、tcp超时时间等），通过metadata()可以获取当前channel的tcp参数设置

##  ChannelPipeline和ChannelHandler
（1）ChannelPipeline和ChannelHandler类似于Servlet和Filter，都是责任链模式的一种变形
（2）ChannelPipeline是Channel的数据管道的抽象，消息在ChannelPipeline中流动
（3）ChannelPipeline持有io事件拦截器ChannelHandler的链表，由ChannelHandler对IO事件进行拦截和处理，可以方便地新增和删除ChannelHandler
（4）这里关注下ChannelHandlerAdapter，因为ChannelHandler接口中的方法太多，所以如果自定义的handler直接继承该接口要实现很多用不到的方法，所以这里使用了适配器模式，简单地讲就是一个对接口中的方法进行了空实现的抽象类
（5）ChannelHandler用ChannelHandlerContext包裹着，有prev和next节点，可以获取前后ChannelHandler，read时从ChannelPipeline的head执行到tail，write时从tail执行到head，所以head既是read事件的起点也是write事件的终点，与io交互最紧密
## Reactor模型
### 单线程模型
指的是所有的io操作都在同一个nio线程上面完成，nio线程的职责如下
（1）接收客户端的TCP连接
（2）读取通信对端的请求或者应答
（3）向通信对端发送消息请求或者应答
这种模型，理论上可以处理所有的io相关操作。不过只适用于一些小容量的应用场景，对于高负载、大并发的应用不合适
（1）性能问题。一个nio线程处理成百上千个请求，性能无法支撑，哪怕CPU利用率100%
（2）可靠性问题。一旦nio线程意外进入死循环，那么会导致整个系统通信不可用
### 多线程模型
为了解决上述单线程模型的问题，引入了多线程模型。和单线程模型最大的区别是，有一组nio线程处理io操作。
（1）有一个专门的nio线程----Acceptor线程用于监听服务端，接收客户端的tcp连接请求
（2）网络io操作----读、写等由一个nio线程池负责，线程池负责消息的读取、编解码和发送
（3）一个nio线程可以同时处理n条链路，但是1个链路只对应1个nio线程，防止发生并发操作问题
### 主从模型
大多数情况，多线程模型都能满足需求。但是在及其特殊的场景中，一个nio线程负责监听和处理所有的客户端连接可能还是会有问题。例如百万级别的客户端连接，单独一个Acceptor线程可能会存在性能不足问题。为此，引出Reactor多线程模型
（1）服务端用于接收客户端连接的不再是一个单独的nio线程，而是一个独立的nio线程池。
（2）Acceptor接收到客户端tcp连接请求处理完成后，将新创建的SocketChannel注册到io线程池（sub reactor线程池）的某个io线程上。由它负责socketChannel的读写和编码工作。
（3）Acceptor线程池只用于客户端的连接，一旦链路建立成功，就将链路注册到subReactor线程池的io线程上，由io线程负责后续的io操作。