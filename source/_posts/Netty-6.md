---
title: Netty源码分析(5)-WriteAndFlush
date: 2018-11-02 07:28:51
tags: Netty
categories: Netty源码分析
---
WriteAndFlush
<!--more-->
WriteAndFlush这里分为两个步骤，Write,Flush
# Write
回忆上文追到的unsafe的write方法，这个方法就是本文的入口。
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
一共分为四个步骤
（1）确认当前是reactor线程
（2）过滤msg
```java
protected final Object filterOutboundMessage(Object msg) {
    if (msg instanceof ByteBuf) {
        ByteBuf buf = (ByteBuf) msg;
        if (buf.isDirect()) {
            return msg;
        }

        return newDirectBuffer(buf);
    }

    if (msg instanceof FileRegion) {
        return msg;
    }

    throw new UnsupportedOperationException(
            "unsupported message type: " + StringUtil.simpleClassName(msg) + EXPECTED_TYPES);
}
```
如果msg既不是ByteBuf类型也不是FileRegion类型的，那么直接抛出异常。这里还有一个值得注意的，方法会将所有非直接内存转换成直接内存DirectBuffer
（3）估算msg的size
（4）调用<code>outboundBuffer.addMessage(msg, size, promise)</code>
```java
//ChannelOutboundBuffer
public void addMessage(Object msg, int size, ChannelPromise promise) {
    Entry entry = Entry.newInstance(msg, size, total(msg), promise);
    if (tailEntry == null) {
        flushedEntry = null;
        tailEntry = entry;
    } else {
        Entry tail = tailEntry;
        tail.next = entry;
        tailEntry = entry;
    }
    if (unflushedEntry == null) {
        unflushedEntry = entry;
    }
    incrementPendingOutboundBytes(size, false);
}
```
这里涉及几个指针，tailEntry,flushedEntry,unFlushedEntry，在执行N次上述方法后，指针之间如下图所示
<img src='001.jpg'>
# Flush
在pipeline上调用的flush最终都会落到head节点上
```java
//HeadContext
public void flush(ChannelHandlerContext ctx) throws Exception {
    unsafe.flush();
}
```
```java
//AbstractUnsafe
public final void flush() {
    assertEventLoop();

    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        return;
    }

    outboundBuffer.addFlush();
    flush0();
}
```
这里主要看addFlush和flush0方法
```java
public void addFlush() {
    Entry entry = unflushedEntry;
    if (entry != null) {
        if (flushedEntry == null) {
            flushedEntry = entry;
        }
        do {
            flushed ++;
            if (!entry.promise.setUncancellable()) {
                int pending = entry.cancel();
                decrementPendingOutboundBytes(pending, false, true);
            }
            entry = entry.next;
        } while (entry != null);

        unflushedEntry = null;
    }
}
```
结合上面那个图，我们知道该方法执行完毕后，unFlushedEntry和flushedEntry位置对调了。接着去看flush0
```java
protected void flush0() {
    doWrite(outboundBuffer);
}
```
```java
protected void doWrite(ChannelOutboundBuffer in) throws Exception {
        int writeSpinCount = -1;
        boolean setOpWrite = false;
        for (;;) {
            Object msg = in.current();
            if (msg instanceof ByteBuf) {
                ByteBuf buf = (ByteBuf) msg;
                int readableBytes = buf.readableBytes();
                if (readableBytes == 0) {
                    in.remove();
                    continue;
                }

                boolean done = false;
                long flushedAmount = 0;
                if (writeSpinCount == -1) {
                    writeSpinCount = config().getWriteSpinCount();
                }
                for (int i = writeSpinCount - 1; i >= 0; i --) {
                    int localFlushedAmount = doWriteBytes(buf);
                    if (localFlushedAmount == 0) {
                        setOpWrite = true;
                        break;
                    }

                    flushedAmount += localFlushedAmount;
                    if (!buf.isReadable()) {
                        done = true;
                        break;
                    }
                }

                in.progress(flushedAmount);

                if (done) {
                    in.remove();
                } else {
                    break;
                }
            } 
            if (done) {
                in.remove();
            } else {
                break;
            }
        } else {
            throw new Error();
        }
        }
        incompleteWrite(setOpWrite);
    }
```
（1）通过current方法拿到第一个需要flush的节点
```java
public Object current() {
    Entry entry = flushedEntry;
    if (entry == null) {
        return null;
    }
    return entry.msg;
}
```
(2)获取自旋的次数，后文会提到为啥要自旋
(3)调用<code>doWriteBytes(buf)</code>
```java
protected int doWriteBytes(ByteBuf buf) throws Exception {
    final int expectedWrittenBytes = buf.readableBytes();
    return buf.readBytes(javaChannel(), expectedWrittenBytes);
}
```
这里解释下为啥要自旋，因为doWriteBytes并不保证一次会将entry的数据读取完毕，所以需要不断循环直到<code>!buf.isReadable()</code>。
我们看到这里调用了ByteBuf的readBytes方法将数据写到对应的channel中，官方文档如是说，Transfers this buffer's data to the specified stream starting at the current。
（4）删除节点，就是普通的链表删除头节点的套路
```java
private void removeEntry(Entry e) {
    if (-- flushed == 0) {
        flushedEntry = null;
        if (e == tailEntry) {
            tailEntry = null;
            unflushedEntry = null;
        }
    } else {
        flushedEntry = e.next;
    }
}
```
几个值得注意的点：
1.netty中的缓冲区中的byteBuf是DirectByteBuf
2.调用write方法实际是把数据写到了单向链表中
3.调用flush才是真正的把数据写到socket缓冲区

参考链接：
https://www.jianshu.com/p/feaeaab2ce56