---
title: Netty源码分析(4)-Pipeline
date: 2018-11-01 11:18:32
tags: Netty
categories: Netty源码分析
---
Netty中的pipeline
<!--more-->
# pipeline的实例化
回忆下Netty系列的第二篇文章，在创建channel的过程中有几个副产品
```java
// AbstractChannel
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    id = newId();
    unsafe = newUnsafe();
    pipeline = newChannelPipeline();
}
```
这里的pipeline就是本文研究的重点，我们进入到newChannelPipeline中一探究竟
```java
// AbstractChannel
protected DefaultChannelPipeline newChannelPipeline() {
    return new DefaultChannelPipeline(this);
}
//DefaultChannelPipeline
protected DefaultChannelPipeline(Channel channel) {
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
    succeededFuture = new SucceededChannelFuture(channel, null);
    voidPromise =  new VoidChannelPromise(channel, true);

    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}
```
在构造函数中，pipeline持有channel，并且实例化了pipeline中首尾两个阀门：AbstractChannelHandlerContext类型的head和tail，并构造了双向链表。
其中head和tail是双向链表的两个哨兵，这样编码时候就可以保证当前节点前后都不为空，不需要一些无谓的判断
# pipeline#addLast
```java
//DefaultChannelPipeline
public final ChannelPipeline addLast(ChannelHandler... handlers) {
    return addLast(null, handlers);
}
public final ChannelPipeline addLast(EventExecutorGroup executor, ChannelHandler... handlers) {
    if (handlers == null) {
        throw new NullPointerException("handlers");
    }

    for (ChannelHandler h: handlers) {
        if (h == null) {
            break;
        }
        addLast(executor, null, h);
    }

    return this;
}
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    synchronized (this) {
        checkMultiplicity(handler);

        newCtx = newContext(group, filterName(name, handler), handler);

        addLast0(newCtx);

        if (!registered) {
            newCtx.setAddPending();
            callHandlerCallbackLater(newCtx, true);
            return this;
        }

        EventExecutor executor = newCtx.executor();
        if (!executor.inEventLoop()) {
            newCtx.setAddPending();
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    callHandlerAdded0(newCtx);
                }
            });
            return this;
        }
    }
    callHandlerAdded0(newCtx);
    return this;
}
```
我们只关心最后的方法内容，分成了四个步骤
（1）检查handler是否重复，如果handler并未配sharable注解且被重复使用那么要抛出错误
（2）新建节点，为handler取一个独一无二的名字，并设置handlerContext的属性初值，包括name,pipeline,executor,inbound,outbound
（3）添加节点，就是普通的双向列表插入节点的套路
```java
private void addLast0(AbstractChannelHandlerContext newCtx) {
    AbstractChannelHandlerContext prev = tail.prev;
    newCtx.prev = prev;
    newCtx.next = tail;
    prev.next = newCtx;
    tail.prev = newCtx;
}
```
（4）执行回调，如果不在reactor线程中，那么将回调抽象成一次任务，放到reactor线程的任务池中；反之，直接调用callHandlerAdded0方法
```java
private void callHandlerAdded0(final AbstractChannelHandlerContext ctx) {
    ctx.handler().handlerAdded(ctx);
    ctx.setAddComplete();
}
```
# pipeline#remove
```java
@Override
public final ChannelPipeline remove(ChannelHandler handler) {
    remove(getContextOrDie(handler));
    return this;
}
private AbstractChannelHandlerContext remove(final AbstractChannelHandlerContext ctx) {
    assert ctx != head && ctx != tail;
    synchronized (this) {
        remove0(ctx);

        if (!registered) {
            callHandlerCallbackLater(ctx, false);
            return ctx;
        }

        EventExecutor executor = ctx.executor();
        if (!executor.inEventLoop()) {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    callHandlerRemoved0(ctx);
                }
            });
            return ctx;
        }
    }
    callHandlerRemoved0(ctx);
    return ctx;
}
```
和添加节点的套路很像，但是简单了很多，只有两个步骤。
（1）移除节点
```java
private static void remove0(AbstractChannelHandlerContext ctx) {
    AbstractChannelHandlerContext prev = ctx.prev;
    AbstractChannelHandlerContext next = ctx.next;
    prev.next = next;
    next.prev = prev;
}
```
（2）执行回调handler#handlerRemoved，和添加类似如果不在reactor线程也要添加到任务池
# fire pipeline
## intBound
回忆下服务端启动过程后，eventLoop不断循环做三件事情，其中第二件事情负责监听IO事件
```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    //...
    if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                unsafe.read();
                if (!ch.isOpen()) {
                    // Connection already closed - no need to handle write.
                    return;
                }
            }
}
```
我们只关注READ事件，继续跟进unsafe中
```java
//AbstractNioByteUnsafe
public final void read() {
    //...
    allocHandle.lastBytesRead(doReadBytes(byteBuf));
    pipeline.fireChannelRead(byteBuf);
}
```
终于fire了，继续跟进
```java
//DefaultChannelPipeline
@Override
public final ChannelPipeline fireChannelRead(Object msg) {
    //channel维度的事件传递会贯穿整个链表
    AbstractChannelHandlerContext.invokeChannelRead(head, msg);
    return this;
}
```
注意上面代码中的head。继续跟进
```java
//AbstractChannelHandlerContext
static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
    final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeChannelRead(m);
    }
    //... 
}
private void invokeChannelRead(Object msg) {
    //...
    ((ChannelInboundHandler) handler()).channelRead(this,msg);
    //...
}
```
这里要回忆下，this表示的是head，而msg表示的是读取到的byteBuf，接下来看看head的channelRead方法
```java
//DefaultChannelPipeline.HeadContext
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    ctx.fireChannelRead(msg);
}
```
```java
//AbstractChannelHandlerContext
public ChannelHandlerContext fireChannelRead(final Object msg) {
    invokeChannelRead(findContextInbound(), msg);
    return this;
}
```
重点看下findContextInbound方法，看名字就知道是寻找下一个Inbound类型的handler Ps：因为现在是读事件，所以走的是Inbound这条路
显然这样调用下去，最后一个被调用的是tail。
```java
//DefaultChannelPipeline.TailContext
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    onUnhandledInboundMessage(msg);
}
```
这里值得注意的是
1）InBoundHandler一般为被动响应事件，消息传递方向为链表的正向，from head
2）OutBoundHandler一般为主动调用，消息传递方向为链表的反向，from tail
3）一般针对channel or pipeline 维度的事件传递都是从头or从尾开始的，会贯穿整个链表
4）而针对单个context的事件传播都是从当前节点开始的
## outBound
一般在push系统中都会主动调用channel.writeAndFlush(msg)方法，我们的入口也是这里。
```java
//AbstractChannel
public ChannelFuture writeAndFlush(Object msg) {
    return pipeline.writeAndFlush(msg);
}
//DefaultChannelPipeline
public final ChannelFuture writeAndFlush(Object msg) {
    return tail.writeAndFlush(msg);
}
```
注意这里的入口是tail。
```java
//AbstractChannelHandlerContext
public ChannelFuture writeAndFlush(Object msg, ChannelPromise promise) {
    //...
    write(msg, true, promise);
    //...
}

private void write(Object msg, boolean flush, ChannelPromise promise) {
    AbstractChannelHandlerContext next = findContextOutbound();
    final Object m = pipeline.touch(msg, next);
    if (flush) {
        next.invokeWriteAndFlush(m, promise);
    } else {
        next.invokeWrite(m, promise);
    }
}
```
因为我们调用的是writeAndFlush所以这里传入的flush参数为true。可以看到方法中的findContextOutbound方法，和上面的findContextInbound方法类似，这个方法用来寻找Outbound类型的handler。接着调用下面这个方法
```java
private void invokeWriteAndFlush(Object msg, ChannelPromise promise) {
    if (invokeHandler()) {
        invokeWrite0(msg, promise);
        invokeFlush0();
    } else {
        writeAndFlush(msg, promise);
    }
}
```
这里我们研究的是事件的传播，所以重点关注invokeWrite0方法
```java
private void invokeWrite0(Object msg, ChannelPromise promise) {
    try {
        ((ChannelOutboundHandler) handler()).write(this, msg, promise);
    } catch (Throwable t) {
        notifyOutboundHandlerException(t, promise);
    }
}
```
和inBound类似，OutBound也会不断调用handler的write方法，直到到达head，下面看看head做了啥
```java
//DefaultChannelPipeline.HeadContext
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    unsafe.write(msg, promise);
}
```
```java
public final void write(Object msg, ChannelPromise promise) {
    assertEventLoop();

    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        safeSetFailure(promise, WRITE_CLOSED_CHANNEL_EXCEPTION);
        ReferenceCountUtil.release(msg);
        return;
    }

    int size;
    try {
        msg = filterOutboundMessage(msg);
        size = pipeline.estimatorHandle().size(msg);
        if (size < 0) {
            size = 0;
        }
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        ReferenceCountUtil.release(msg);
        return;
    }
    outboundBuffer.addMessage(msg, size, promise);
}
```
我们就追到这里，后续的放到下篇文章去分析。

总结，read,write,flush等操作都是先执行当前handler覆写的方法，如果没有覆写那么默认调用父类ChannelOutHandlerAdapter,ChannelInHandlerAdapter的对应方法，而父类中的默认方法就是简单地将事件进行传递
# pipeline exception
## inBound
```java
//AbstractChannelHandlerContext
private void invokeChannelRead(Object msg) {
    try {
        ((ChannelInboundHandler) handler()).channelRead(this, msg);
    } catch (Throwable t) {
        notifyHandlerException(t);
    }
}
```
我们看catch代码块中的方法
```java
//AbstractChannelHandlerContext
private void notifyHandlerException(Throwable cause) {
    invokeExceptionCaught(cause);
}
private void invokeExceptionCaught(final Throwable cause) {   
    handler().exceptionCaught(this, cause);  
}
```
这里可以看到都是直接调用的handler的exceptionCaught方法，如果当前handler没有覆写这个方法，那么会去调用下面这个方法
```java
//ChannelHandlerAdapter
 @Override
public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
    ctx.fireExceptionCaught(cause);
}
```
进到这个方法那就是一直向后推了，直到到了某个覆写了exceptionCaught方法的handler，这也是一般将解决Exc的handler配置在tail之前的一个原因。
## Outbound
看过了Inbound，再去看Outbound就很简单了，我们发现即使Outbound是从tail向前传递，但是也一样是调用notifyHandlerException方法，那么问题就很简单了，无论在哪个handler抛出异常，异常都是正向传递的且会忽略handler本身的类型。
比如
IA IB IC OA OB OC其中IB抛出了异常，那么会遍历IB IC OA OB OC如果异常一直被向后传递，最终会fall到tail节点，该节点会将异常打印到控制台

参考链接：
https://www.jianshu.com/p/6efa9c5fa702
https://www.jianshu.com/p/087b7e9a27a2