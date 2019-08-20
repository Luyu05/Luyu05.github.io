---
title: Netty源码分析(1)-服务端启动过程
date: 2018-10-23 07:36:47
tags: Netty
categories: Netty源码分析
---
GoodLuck
<!-- more -->
# Demo
```java
public class DiscardServer {
    
    private int port;
    
    public DiscardServer(int port) {
        this.port = port;
    }

    public void run() throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup(); // (1)
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap serverBootStrap = new ServerBootstrap(); // (2)
            serverBootStrap.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class) // (3)
                    .handler(new SimpleServerHandler())
                    .childHandler(new ChannelInitializer<SocketChannel>() { // (4)
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                        }
                    })
                    .option(ChannelOption.SO_BACKLOG, 128)          // (5)
                    .childOption(ChannelOption.SO_KEEPALIVE, true); // (6)
    
            // Bind and start to accept incoming connections.
            ChannelFuture f = serverBootStrap.bind(port).sync(); // (7)
    
            // shut down your server.
            f.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
            bossGroup.shutdownGracefully();
        }
    }

    public static void main(String[] args) throws Exception {
            new DiscardServer(8080).run();
        }
    }

    private static class SimpleServerHandler extends ChannelInboundHandlerAdapter {
        @Override
        public void channelActive(ChannelHandlerContext ctx) throws Exception {
            System.out.println("channelActive");
        }

        @Override
        public void channelRegistered(ChannelHandlerContext ctx) throws Exception {
            System.out.println("channelRegistered");
        }

        @Override
        public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
            System.out.println("handlerAdded");
        }
    }
```
* <code>NioEventLoopGroup</code>是一个处理IO操作的多线程事件循环。其中EventLoop是一个死循环，主要做三件事：<font color=blue>检测IO事件，处理IO事件，执行任务</font>
* <code>ServerBootstrap</code>是一个辅助类，仅仅用来将组件拼装到一起
* <code>group(bossGroup, workerGroup)</code> 我们需要两种类型的人干活，一个是老板，一个是工人，老板负责从外面接活，接到的活分配给工人干，放到这里，bossGroup的作用就是不断地accept到新的连接，将新的连接丢给workerGroup来处理。bossGroup一般用来接收传入的连接；workerGroup负责处理连接的IO事件。这里的workerGroup即源码中的childGroup
* <code>channel</code>表示一个连接
* <code>option</code>一般用来配置TCP参数
* <code>childOption</code>用来配置父channel接收到的channel，本例子中父channel是NioServerSocketChannel
* <code>ch.pipeline().addLast(new SimpleServerHandler())</code>这里使用了管道模式，将自定义的handler放到pipeline中当做『拦路虎』
* <code>.handler</code>这里是father的handler，也就是bossGroup中channel使用的handler
* <code>.childHandler</code>注意这里是child的handler，也就是workerGroup中的channel使用的handler
* <code>handlerChannelInitializer</code>这个类是一个特殊的handler，当回调执行handlerAdded方法时候会调用initChannel方法

上面代码在本地执行之后，最终控制台输出为（注意这时候还没有客户端仅仅是服务端启动就触发的事件）：
```java
handlerAdded
channelRegistered
channelActive
```

# 分析源码
上面例子中，<code>serverBootStrap.bind(port).sync()</code>就是我们的入口点。
```java
public ChannelFuture bind(SocketAddress localAddress) {
    validate();
    if (localAddress == null) {
        throw new NullPointerException("localAddress");
    }
    return doBind(localAddress);
}
```
进入到doBind中
```java
private ChannelFuture doBind(final SocketAddress localAddress) {
    //...
    final ChannelFuture regFuture = initAndRegister();
    //...
    doBind0(regFuture, channel, localAddress, promise);
}
```
省略了很多细节，重点关注两个方法initAndRegister（初始化并注册channel）和doBind0（绑定channel监听端口），至此分道扬镳~

## initAndRegister
先看省略后的骨干代码
```java
final ChannelFuture initAndRegister() {
    //...
    final Channel channel = channelFactory().newChannel();
    //...
    init(channel);
    //...
    ChannelFuture regFuture = group().register(channel);
    //...
    return regFuture;
}
```
我们还是专注于核心代码，抛开边角料，我们看到 initAndRegister() 做了几件事情
1.new channel
2.init channel
3.register channel

我们逐步分析这三件事情
### new channel
chnnel是由channelFactory生产出来的，那么channelFactory是什么时候被初始化的呢，我们层层回溯，发现是在<code>.channel(NioServerSocketChannel.class)</code>方法的时候创建的
```java
public B channel(Class<? extends C> channelClass) {
    if (channelClass == null) {
        throw new NullPointerException("channelClass");
    }
    return channelFactory(new ReflectiveChannelFactory<C>(channelClass));
}
```
我们进入ReflectiveChannelFactory#newChannel方法看看
```java
@Override
public T newChannel() {
    try {
        return clazz.newInstance();
    } catch (Throwable t) {
        throw new ChannelException("Unable to create Channel from class " + clazz, t);
    }
}
```
原来是通过反射的方式来创建我们传入的NioServerSocketChannel，换言之通过了默认构造函数new出了一个NioServerSocketChannel。
接下来进入到NioServerSocketChannel的构造函数去看看
```java
private static final SelectorProvider DEFAULT_SELECTOR_PROVIDER = SelectorProvider.provider();
public NioServerSocketChannel() {
    this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}
private static ServerSocketChannel newSocket(SelectorProvider provider) {
    //...
    return provider.openServerSocketChannel();
}
```
这里通过SelectorProvider.openServerSocketChannel方法创建了一个channel
```java
public NioServerSocketChannel(ServerSocketChannel channel) {
    super(null, channel, SelectionKey.OP_ACCEPT);
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}
```
我们发现这里new了一个NioServerSocketChannelConfig，顶层接口为ChannelConfig，它表示channel的配置属性集合。
我们再来看上面代码部分的第一行，追溯到父类AbstractNioChannel中
```java
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);
    this.ch = ch;
    this.readInterestOp = readInterestOp;
    //...
    ch.configureBlocking(false);
    //...
}
```
将创建出来的ServerSocketChannel保存到成员变量；
将channel设置为非阻塞模式；
将传入的SelectionKey.OP_ACCEPT保存到成员变量；

接下来重点看super(parent)
```java
// AbstractChannel.java
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    id = newId();
    unsafe = newUnsafe();
    pipeline = newChannelPipeline();
}
```
这里new出了三个大件，分别为
1) id表示channel的唯一标识
2）Unsafe是在实际进行数据传输时候使用的类，相当于被channel封装了
3）pipeline表示处理输入输出事件的管道（类似双向链表）

**<font color=blue>总结，new一个channel的过程，就是通过channelFactory的反射机制创建的channel，而channelFactory是在调用.channel时候初始化的；接下来通过自己的构造函数和三个父类的构造函数分别创建了：</font>**
* channel,
* channelConfig,
* 保存了变量到成员变量,
* channelId,unsafe,pipeline

### init channel
```java
@Override
void init(Channel channel) throws Exception {
    //....
}
```
init内代码太长，接下来逐步分析
```java
final Map<ChannelOption<?>, Object> options = options();
synchronized (options) {
    channel.config().setOptions(options);
}

final Map<AttributeKey<?>, Object> attrs = attrs();
synchronized (attrs) {
    for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
        @SuppressWarnings("unchecked")
        AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
        channel.attr(key).set(e.getValue());
    }
}

ChannelPipeline p = channel.pipeline();
if (handler() != null) {
    p.addLast(handler());
}
```
这里面的option()/attrs()/handler()方法都是提取Bootstrap组装时传入的参数。我们看这段代码其实就是把用户输入的那些属性配置到channel中。注意这里的channel是NioServerSocketChannel
Ps channel初始化三板斧：option/attr/pipeline.addHandler
```java
final EventLoopGroup currentChildGroup = childGroup;
final ChannelHandler currentChildHandler = childHandler;
final Entry<ChannelOption<?>, Object>[] currentChildOptions;
final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
synchronized (childOptions) {
    currentChildOptions = childOptions.entrySet().toArray(newOptionArray(childOptions.size()));
}
synchronized (childAttrs) {
    currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(childAttrs.size()));
}

p.addLast(new ChannelInitializer<Channel>() {
    @Override
    public void initChannel(Channel ch) throws Exception {
        ch.pipeline().addLast(new ServerBootstrapAcceptor(
                currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
    }
});
```
这段代码简单讲就是在NioServerSocketChannel中配置了最后一个『写死』的handler，我们进入到new ServerBootstrapAcceptor看看细节
Ps这里的childGroup就是我们demo中传入的workGroup
Ps值得注意的是，这里还并未执行initChannel方法，该方法是在执行回调方法handlerAdded后调用的
```java
private static class ServerBootstrapAcceptor extends ChannelInboundHandlerAdapter {
    private final EventLoopGroup childGroup;
    private final ChannelHandler childHandler;
    private final Entry<ChannelOption<?>, Object>[] childOptions;
    private final Entry<AttributeKey<?>, Object>[] childAttrs;

    ServerBootstrapAcceptor(
            EventLoopGroup childGroup, ChannelHandler childHandler,
            Entry<ChannelOption<?>, Object>[] childOptions, Entry<AttributeKey<?>, Object>[] childAttrs) {
        this.childGroup = childGroup;
        this.childHandler = childHandler;
        this.childOptions = childOptions;
        this.childAttrs = childAttrs;
    }
//...
}
```
我们看这个类很简单，它是ServerBootStrap的一个内部类。它持有很多child开头的属性，这里的构造方法就是简单地将值赋给成员变量。下面看一个关键的方法
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
这个方法是NioServerSocketChannel的读事件就绪时候触发的，可以看到此时的msg是一个channel，而这个channel是要传递给workGroup的，我们看到首先对这个channel进行了『初始化三板斧』，接下来将这个channel注册到了childGroup也就是workerGroup，这里就是Boss将任务分配给Worker的地方。
<font color=red>为何这里可以直接将msg强制转为Channel?</font>
答：这个追溯到eventLoop的第二板斧中，在那里显示通过jdk原生的javaChannel().accept()获得了SocketChannel，接下来再将这个原生channel封装成NioSocketChannel，最后将封装好的channel传递到pipeline中

至此channel初始化工作也完成了，又到了总结时间。
**<font color=blue>首先对NioServerSocketChannel执行初始化三板斧（attr/option/handler），在最后一板斧中将一个特殊的handler-ServerBootStrapAcceptor塞到了pipeline中，这个handler的read方法比较特殊，它将接收到的channel注册到了workGroup</font>**

这里对表示连接的channel进行宏观概览
<img src='001.jpg'>

### register channel
<code>ChannelFuture regFuture = config().group().register(channel);</code>，这里的config表示的是bootStrap的config，而这里的group表示的是bossGroup
```java
//MultithreadEventLoopGroup.class
//这个next其实就是从group中选择一个eventLoop
@Override
public ChannelFuture register(Channel channel) {
    return next().register(channel);
}

//SingleThreadEventLoop.class
public ChannelFuture register(Channel channel) {
    return this.register((ChannelPromise)(new DefaultChannelPromise(channel, this)));
}

public ChannelFuture register(ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    promise.channel().unsafe().register(this, promise);
    return promise;
}
```
<font color=red>之前代码追到这里就停了，导致忽略了很重要的点。我们继续向下追<font>
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
由于我们处于main线程中，所以这里会执行else的逻辑，注意这里的evenLoop是我们自己new的EvenLoopGroup随机挑选的一个，在这里赋给了当前的channel。
这里会执行<code>eventLoop.execute</code>，而这行代码是EvenLoop也就是reactor线程的触发点，执行之后开启reactor线程。我们继续跟进注册过程
```java
private void register0(ChannelPromise promise) {
    //...
    doRegister();
    //...
    pipeline.invokeHandlerAddedIfNeeded();
    pipeline.fireChannelRegistered();
    if (isActive()) {
        if (firstRegistration) {
            pipeline.fireChannelActive();
        } else if (config().isAutoRead()) {
            beginRead();
        }
    }
}
```
```java
@Override
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
            selectionKey = javaChannel().register(eventLoop().selector, 0, this);
            return;
        } catch (CancelledKeyException e) {
            if (!selected) {
                eventLoop().selectNow();
                selected = true;
            } else {
                throw e;
            }
        }
    }
}
```
最后跟到了这行代码<code>selectionKey = javaChannel().register(eventLoop().selector, 0, this);</code>这里就是经典的jdk注册channel的方式。至此注册行为结束，接下来会触发一些回调方法
<code>pipeline.invokeHandlerAddedIfNeeded()</code>触发了最开始我们定义的SimpleServerHandler中的handlerAdded方法，此时控制台输出"handlerAdded"。在这里会执行ChannelInitializer#handlerAdded方法，该方法会回调我们自行重写的initChannel方法，一般这个方法用于将handler塞到pipeline中
<code>pipeline.fireChannelRegistered()</code>触发了最开始我们定义的SimpleServerHandler中的channelRegistered方法，此时控制台输出"channelRegistered"
<code>isActive()</code>我们进到isActive()方法，发现该方法表示channel对应的socket是否已经处于绑定状态，显然这时候还没有绑定。所以此时还不能回调channelActive方法

**<font color=blue>finally，终于分析完了第一步骤。三件事</font>**

**<font color=blue>1.new一个channel</font>**
    **new一个channel的过程，就是通过channelFactory的反射机制创建的channel，而channelFactory是在调用.channel时候初始化的；接下来通过自己的构造函数和三个父类的构造函数分别创建了：channelchannelConfig,保存了变量到成员变量,channelId,unsafe,pipeline**

**<font color=blue>2.init这个channel</font>**
    **首先对NioServerSocketChannel执行初始化三板斧（attr/option/handler），在最后一板斧中将一个特殊的handler-ServerBootStrapAcceptor塞到了pipeline中，这个handler的read方法比较特殊，它将接收到的channel注册到了workGroup**

**<font color=blue>3.将这个channel register到某个对象（Boss的Reactor线程在这一步骤中启动）</font>**
    **将NioServerSocketChannel注册到了bossGroup的selector上;回调handlerAdd;回调channelRegister**

## doBind0
```java
private static void doBind0(
        final ChannelFuture regFuture, final Channel channel,
        final SocketAddress localAddress, final ChannelPromise promise) {
    // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
    // the pipeline in its channelRegistered() implementation.
    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            if (regFuture.isSuccess()) {
                channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                promise.setFailure(regFuture.cause());
            }
        }
    });
}
```
注意这里的execute方法已经不会再触发reactor线程的运转了（通过CAS保证了只会触发一次），这里只是单纯地新增了一个task供Reacotr线程执行。我们发现这里将绑定的具体行为，封装成了一个异步的任务放到了eventLoop中执行。
<font color=red>Ps 这样的话能够保证优先执行异步封装的register任务再执行bind任务</font>
我们重点去看<code>channel.bind</code>
**<font color=red>这里并不是eventLoop启动的地方！！！</font>**
```java
@Override
public ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
    return pipeline.bind(localAddress, promise);
}
```
层层深入我们跟到了这里，发现调用的是pipeline的bind，这里我也妥协了打了断点跟到了下面这段代码
```java
//AbstractChannel.class
public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
    assertEventLoop();
    //...
    boolean wasActive = isActive();
    doBind(localAddress);
    //...
    if (!wasActive && isActive()) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                pipeline.fireChannelActive();
            }
        });
    }
}
```
大概两件事1）doBind 2）pipeline.fireChannelActive。先来看第一件事情doBind
```java
//NioServerSocketChannel
@Override
protected void doBind(SocketAddress localAddress) throws Exception {
    if (PlatformDependent.javaVersion() >= 7) {
        javaChannel().bind(localAddress, config.getBacklog());
    } else {
        javaChannel().socket().bind(localAddress, config.getBacklog());
    }
}
```
我们可以看到就是简单地jdk形式的绑定地址。绑定过之后，再执行isActive()返回的就是true了，此时会触发channelActive事件，console打印"channelActive"

Finally终于分析完了，自认还算比较清晰，而且将参考文章中一些没有的地方也提到了。

参考链接：https://www.jianshu.com/p/c5068caab217