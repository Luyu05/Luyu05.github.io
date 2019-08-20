---
title: Netty源码分析(3)-新连接接入
date: 2018-10-25 07:26:18
tags: Netty
categories: Netty源码分析
---
上文介绍了服务端启动的过程，这篇文章介绍接收新连接的过程。
<!-- more -->
# (BOSS)EventLoop的启动过程
上回书说到在注册channel过程中有这样的一段代码，当时说是BossEventLoop reactor线程启动的入口，我们从这里去分析本文。
```java
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    //...
    AbstractChannel.this.eventLoop = eventLoop;
    if (eventLoop.inEventLoop()) {
        register0(promise);
    } else {
        eventLoop.execute(new Runnable() {
            @Override
            public void run() {
                register0(promise);
            }
        });
    }
}
```
这段代码就是将channel的注册行为抽象成了一次异步的任务，而eventLoop在执行execute方法时候也同样启动了。我们这里的channel.eventLoop指的是当前channel对应的单个EventLoop(我们这里分析的是BossEventLoop)，而SingleThreadEventExecutor是EventLoop的父类，所以我们根据eventLoop的execute方法追溯到了这里
```java
//SingleThreadEventExecutor.class
@Override
public void execute(Runnable task) {
   //...
    boolean inEventLoop = inEventLoop();
    if (inEventLoop) {
        addTask(task);
    } else {
        startThread();
        addTask(task);
        if (isShutdown() && removeTask(task)) {
            reject();
        }
    }
    //...
}
```
简单地调用栈是这样的startThread()->doStartThread()->run()->NioEventLoop#run
NioEventLoop的run方法是一个很重要的方法详情见[Netty源码分析(2)-Reactor线程模型](https://luyu05.github.io/2018/10/29/Netty-4/)，这是NioEventLoop启动的地方，在这里进行无限循环。执行NioEventLoop的三板斧：轮询注册在selector上的IO事件、处理IO事件、同步执行异步task。而新连接的接入过程就位于第二步骤（处理IO事件），我们接下来去分析。
# (BOSS)EventLoop#processSelectedKeys()
从无参的processSelectedKeys()追到了processSelectedKeys(SelectionKey k, AbstractNioChannel ch)中
```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    //...
    if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
        unsafe.read();
    }
    //...
}
```
---
<font color=red>这里有一个疑问，和当前channel绑定的eventLoop是如何确定的？</font>
答：在channel注册过程中，通过next()方法从bossGroup中随机选择的一个eventLoop

---
我们看到此时当EventLoop（BossGroup中的一员）轮询到了ACCEPT事件会调用unsafe.read()
```java
public void read() {
    assert eventLoop().inEventLoop();
    final ChannelPipeline pipeline = pipeline();
    final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
    do {
        int localRead = doReadMessages(readBuf);
        if (localRead == 0) {
            break;
        }
        if (localRead < 0) {
            closed = true;
            break;
        }
    } while (allocHandle.continueReading());
    int size = readBuf.size();
    for (int i = 0; i < size; i ++) {
        pipeline.fireChannelRead(readBuf.get(i));
    }
    readBuf.clear();
    pipeline.fireChannelReadComplete();
}
```
这里重点关注doReadMessages和fireChannelRead方法
## doReadMessages
```java
//NioServerSocketChannel.class
protected int doReadMessages(List<Object> buf) throws Exception {
    SocketChannel ch = javaChannel().accept();
    try {
        if (ch != null) {
            buf.add(new NioSocketChannel(this, ch));
            return 1;
        }
    } catch (Throwable t) {
        logger.warn("Failed to create a new channel from an accepted socket.", t);
        try {
            ch.close();
        } catch (Throwable t2) {
            logger.warn("Failed to close a socket.", t2);
        }
    }
    return 0;
}
```
第一行代码就是jdk nio接收新连接的代码，因为是监听到了ACCEPT事件所以这里会立刻返回一个SocketChannel，这个也是nio中的channle。我们可以看到buf是一个list容器，再将socketChannel放到容器之前，还要对其进行包装（包装为netty中的channel类型）。
下面去看看具体咋包装的
```java
public NioSocketChannel(Channel parent, SocketChannel socket) {
    super(parent, socket);
    config = new NioSocketChannelConfig(this, socket.socket());
}
```
诶等一等，这个代码看着有点眼熟啊，这就是[Netty源码分析(1)-服务端启动过程](https://luyu05.github.io/2018/10/23/Netty-2/)中new Channel那段代码，所以就不去看了，大概创建了这样几个东西
* channel,
* channelConfig,
* 保存了变量到成员变量,
* channelId,unsafe,pipeline

不过这里还是有一些不同的，这里注册的事件是SelectionKey.OP_READ，而服务端channel注册的是SelectionKey.OP_ACCEPT事件；而且这里的channel是NioScoketChannel

## fireChannelRead
将readBuf填充满NioSocketChannel后接下来轮询这个list去调用下面这个方法
<code>pipeline.fireChannelRead(readBuf.get(i));</code>
我们当前pipeline中除了head和tail还包含一个特殊的ChannelInboundHandler-ServerBootstrapAcceptor，我们暂且忽略在handler链前方的handler直接看ServerBootstrapAcceptor，而ServerBootstrapAcceptor我们在上篇文章已经分析过了，这里再回忆下
```java
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg;
    child.pipeline().addLast(childHandler);
    for (Entry<ChannelOption<?>, Object> e: childOptions) {
        try {
            if (!child.config().setOption((ChannelOption<Object>) e.getKey(), e.getValue())) {
                logger.warn("Unknown channel option: " + e);
            }
        } catch (Throwable t) {
            logger.warn("Failed to set a channel option: " + child, t);
        }
    }
    for (Entry<AttributeKey<?>, Object> e: childAttrs) {
        child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
    }
    try {
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}
```
首先Acceptor接收到的信息强制转换为channel，接着执行childChannel初始化的三板斧，最后将获取到的channel注册到childGroup上，其实到这里这整个流程已经串起来了。 Ps这里的channel实际上是NioSocketChannel
**<font color=blue>（1）Server端调用bind，执行NioServerSocketChannel的实例化、初始化和注册（注册到BossGroup,这个时候触发了boss eventLoop的启动，开始无限循环执行那三件事情），在初始化过程中塞进了一个特殊的handler-ServerBootstrapAcceptor；接着执行bind方法和地址绑定到一起</font>**
**<font color=blue>（2）当有新连接到来的时候，会被eventLoop捕获到。捕获到的channel最开始还是原生的nio channel需要包装成netty中的channel，接着针对接收到的每个channel调用pipeline上的handler中对应的方法，这个时候执行到了ServerBootstrapAcceptor的channelRead方法，而在这个方法中将channel注册到了childGroup（注册到WorkerGroup,这个时候触发了worker eventLoop的启动，开始无限循环执行那三件事情）</font>**
Ps:接收到的channel以后就托管给childGroup了，对应事件的轮询也要由childGroup中的eventLoop来执行

不过为了『追求卓越』我们还是深入看下这里的register方法
我们追着代码走到了这里
```java
//MultithreadEventLoopGroup.class
 @Override
public ChannelFuture register(Channel channel) {
    return next().register(channel);
}
```
先来看看这个next()
```java
@Override
public EventExecutor next() {
    return chooser.next();
}
```
这里的chooser是一个EventExecutorChooser类，它的next方法从eventGroup中选择一个eventLoop，找到了eventLoop之后就是普通的channel注册过程了，和上篇文章中讲到的注册过程完全相同，最终都会执行
<code>selectionKey = javaChannel().register(eventLoop().selector, 0, this);</code>
这段代码是经典的nio注册channel的代码，可见netty其实说白了就是对jdk nio的封装，使他更为好用罢了
Ps可见无论是通过channel向eventLoop注册or eventLoop注册channel最终都是执行上面那段code
<font color=red>极其重要！！！Worker的reactor线程在注册NioSocketChannel过程启动（之后就去无限循环它的那三件事情），以后该channel的读写事件就可以由worker的reactor线程来处理了<font>

在register的过程中会执行很多用户定义的回调方法，比如invokeHandlerAddedIfNeeded()会调用pipeline上handler的handlerAdded方法，注意这个时候已经是childGroup中了，所以根据用户初始时候的输入
```java
.childHandler(new ChannelInitializer<SocketChannel>() {
    @Override
    public void initChannel(SocketChannel ch) throws Exception {
    }
})
```
此时要执行ChannelInitializer的handlerAdded方法，而该方法会调用我们组装过程中定义的initChannel方法，这个方法调用完成后ChannelInitializer会将自己从pipeline中删除。

小结：
1.new server channel
2.init server channel(3 steps)
--- 2.1 in the 3rd step add Acceptor to bossGroup pipeline as a handler
--- Ps:Acceptor#channelRead init newChannel which just connects,and register it on childGroup
3.register server channel to bossGroup
--- 3.1 boss reactor thread start(do 3 things)
--- 3.2 fireManyMethods such as invokeHandlerAddedIfNeeded,fireChannelRegistered
4.bind address to server channel
5.new connect come
6.bossGroup catch new channel and package it as a Netty Channel
7.fire server channel pipeline 
8.run into Acceptor#channelRead and register new channel to childGroup(in this process child reactor thread start)

参考链接：
https://www.jianshu.com/p/0242b1d4dd21