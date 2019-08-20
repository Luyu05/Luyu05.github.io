---
title: Netty源码分析(6)-HashedWheelTimer
date: 2018-11-08 18:00:35
tags: Netty
categories: Netty源码分析
---
Netty正牌哈希轮，还是自顶向下的去分析
<!--more-->
# 宏观概览
```java
HashedWheelTimer
    //inner class
    HashedWheelBucket
    HashedWheelTimeout
    Worker
    //method
    HashedWheelTimer
    HashedWheelTimer
    HashedWheelTimer
    HashedWheelTimer
    HashedWheelTimer
    HashedWheelTimer
    HashedWheelTimer
    createWheel
    newTimeout
    normalizeTicksPerWheel
    start
    stop
    //field
    cancelledTimeouts
    leak
    leakDetector
    logger
    mask
    startTime
    startTimeInitialized
    tickDuration
    timeouts
    wheel
    worker
    WORKER_STATE_INIT
    WORKER_STATE_SHUTDOWN
    WORKER_STATE_STARTED
    WORKER_STATE_UPDATER
    workerState
    workerThread
```

这是HashedWheelTimer的组成部分，这里有三个重要的类：
1）HashedWheelBucket，表示哈希桶，是任务链的容器；包含一些添加、移除timeout的方法
2）HashedWheelTimeout，持有task，本身是链式结构；包含一些cancel,expire方法
3）Worker，实现Runnable，是真正干活的线程，它的run方法执行了对应tick的任务，并且让哈希轮转起来

方法：
1）createWheel:初始化哈希轮，主要就是初始化HashedWheelBucket[] wheel
2）newTimeout:新建一个延时任务，该方法会调用start方法让哈希轮动起来
3）normalizeTicksPerWheel:令哈希轮数组的数为2的n次方，方便取余运算
4）start:让哈希轮开始运转
5）stop:中断worker线程并返回未处理的任务

属性：
1）long startTime:哈希轮开始运转的时间，nanotime形式
这里简单拓展一下nanoTime相关的知识，我们看到jdk给出的关于System.nanotime的注解是
```java
This method can only be used to measure elapsed time and is not related to any 
other notion of system or wall-clock time. The value returned represents
nanoseconds since some fixed but arbitrary origin time (perhaps in the future, so
values may be negative). The same origin is used by all invocations of this method
in an instance of a Java virtual machine; other virtual machine instances are
likely to use a different origin.
```
这个方法只能测量流逝的时间，和真正的时间并无对应关系，所以无法相互转换。这个返回值表示的是相对某个随意的时间点而言的纳秒值，而这个时间点还有可能是未来，所以可能返回的纳秒值是个负数。在同一个jvm中调用该方法的时间锚点（某个随意的时间点）都是同一个。
2）long tickDuration: 指针每次运转的间隔时间
3）Queue<HashedWheelTimeout> timeouts: 刚刚new的timeOut会被放到这里，到下个tick才会分配到指定的bucket
4）HashedWheelBucket[] wheel: 哈希轮实体，是一个bucket数组
5）Runnable worker: 真正执行timeOut延时任务的线程
6）workerState: worker线程的状态

# 深入细节
## 初始化工作
从哈希轮的构造函数开始
```java
public HashedWheelTimer(
        ThreadFactory threadFactory,
        long tickDuration, TimeUnit unit, int ticksPerWheel, boolean leakDetection) {
    //...
    wheel = createWheel(ticksPerWheel);
    mask = wheel.length - 1;

    this.tickDuration = unit.toNanos(tickDuration);

    if (this.tickDuration >= Long.MAX_VALUE / wheel.length) {
        throw new IllegalArgumentException(String.format(
                "tickDuration: %d (expected: 0 < tickDuration in nanos < %d",
                tickDuration, Long.MAX_VALUE / wheel.length));
    }
    workerThread = threadFactory.newThread(worker);

    leak = leakDetection || !workerThread.isDaemon() ? leakDetector.open(this) : null;
}
```
大体的流程是，先进行参数的校验，接着创建哈希轮，再初始化thread，最后进行泄露检测相关的工作。这里重点关注createWheel方法
```java
private static HashedWheelBucket[] createWheel(int ticksPerWheel) {
    //...
    ticksPerWheel = normalizeTicksPerWheel(ticksPerWheel);
    HashedWheelBucket[] wheel = new HashedWheelBucket[ticksPerWheel];
    for (int i = 0; i < wheel.length; i ++) {
        wheel[i] = new HashedWheelBucket();
    }
    return wheel;
}
```
先调用了<code>normalizeTicksPerWheel(ticksPerWheel)</code>这个方法很简单，就是将ticksPerWheel二次方化，比如传7那么这里将返回8；
接着对wheel进行了初始化，至此初始化工作就结束了，接着看看哈希轮如何运转的。
## 哈希轮的运转
哈希轮的使用套路一般是，先new一个哈希轮，接着调用下面这个方法就ok了，所以这里就是哈希轮运转的入口
```java
public Timeout newTimeout(TimerTask task, long delay, TimeUnit unit) {
    if (task == null) {
        throw new NullPointerException("task");
    }
    if (unit == null) {
        throw new NullPointerException("unit");
    }
    start();

    // Add the timeout to the timeout queue which will be processed on the next tick.
    // During processing all the queued HashedWheelTimeouts will be added to the correct HashedWheelBucket.
    //这里要注意，deadline是相对startTime需要经历的时间长短
    long deadline = System.nanoTime() + unit.toNanos(delay) - startTime;
    HashedWheelTimeout timeout = new HashedWheelTimeout(this, task, deadline);
    timeouts.add(timeout);
    return timeout;
}
```
这个方法先是进行了必要参数的校验，接着调用了start方法，最后new了一个timeOut并放到了Queue&lt;HashedWheelTimeout&gt;timeouts中。
那么为啥在这里调用start方法？
这也是netty将优化做到丧心病狂的体现，因为如果没有任何任务，哈希轮『空转』是没有任何意义的，所以至少有一个timeout，哈希轮才转起来。
我们接下来就先看看start方法
```java
public void start() {
    switch (WORKER_STATE_UPDATER.get(this)) {
        case WORKER_STATE_INIT:
            if (WORKER_STATE_UPDATER.compareAndSet(this, WORKER_STATE_INIT, WORKER_STATE_STARTED)) {
                workerThread.start();
            }
            break;
        case WORKER_STATE_STARTED:
            break;
        case WORKER_STATE_SHUTDOWN:
            throw new IllegalStateException("cannot be started once stopped");
        default:
            throw new Error("Invalid WorkerState");
    }

    // Wait until the startTime is initialized by the worker.
    while (startTime == 0) {
        try {
            startTimeInitialized.await();
        } catch (InterruptedException ignore) {
            // Ignore - it will be ready very soon.
        }
    }
}
```
首先通过CAS更新了workerState的状态（无锁化的体现），接着异步调用了<code>workerThread.run()</code>，我们先不去看这个方法，先看后续的流程，代码最后是一个while循环，循环结束的条件是startTime!=0，这里用这个条件表示start成功，接下来具体到worker中再看。
```java
public void run() {
    // Initialize the startTime.
    startTime = System.nanoTime();
    //严谨的一批，真不知道nanoTime返回0的概率是多少，我觉得比国战100连胜还要难
    //这里startTime有可能返回0，但是我们用0表示并未初始化，所以这里要换一个值
    if (startTime == 0) {
        // We use 0 as an indicator for the uninitialized value here, so make sure it's not 0 when initialized.
        startTime = 1;
    }

    // Notify the other threads waiting for the initialization at start().
    // 这里是一个countDownLatch，当这里countDown之后，
    // start方法中while循环中的await就可以被唤醒了，进而newTimeout也被唤醒，
    // 可以计算delayTime了
    startTimeInitialized.countDown();
    
    //哈希轮开始转
    do {
        //这个方法可能会sleep一段时间，之后返回的是系统当前的时刻，而这个当前时刻=指针移动的时刻
        //本文后续会分析这个方法
        final long deadline = waitForNextTick();
        if (deadline > 0) {
            //tick取余，定位到index
            int idx = (int) (tick & mask);
            //处理被cancel的任务
            processCancelledTasks();
            //根据index定位到bucket
            HashedWheelBucket bucket =
                    wheel[idx];
            //把timeouts queue中的任务 放到属于它的bucket中 每次取100000个
            transferTimeoutsToBuckets();
            //重点方法，执行当前bucket中到期的任务
            bucket.expireTimeouts(deadline);
            tick++;
        }
    } while (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_STARTED);


    //代码执行到这里，哈希轮已经不转了~
    // Fill the unprocessedTimeouts so we can return them from stop() method.
    //这里是从每个bucket中移除未被处理的任务
    for (HashedWheelBucket bucket: wheel) {
        bucket.clearTimeouts(unprocessedTimeouts);
    }
    //这里是从timeouts queue中移除未被处理的任务
    for (;;) {
        HashedWheelTimeout timeout = timeouts.poll();
        if (timeout == null) {
            break;
        }
        if (!timeout.isCancelled()) {
            unprocessedTimeouts.add(timeout);
        }
    }
    processCancelledTasks();
}
```
按照上面代码的顺序，逐一去看每个方法
```java
//Worker.class
private long waitForNextTick() {
    //相对startTime来讲,下次指针移动还要过多久
    long deadline = tickDuration * (tick + 1);

    for (;;) {
        //当前时间相对startTime，已经开始了多久
        final long currentTime = System.nanoTime() - startTime;
        //距离指针开动还有多久ms，假如是deadline-currentTime= 2000002，通过+一个999999，此时sleepTimeMs = 3
        long sleepTimeMs = (deadline - currentTime + 999999) / 1000000;
        //sleep之后，代码会进入到这里,正常情况return的就是当前时间
        if (sleepTimeMs <= 0) {
            if (currentTime == Long.MIN_VALUE) {
                return -Long.MAX_VALUE;
            } else {
                return currentTime;
            }
        }

        //这里是说windows系统可能会有些bug，要把sleep的时间转为10的倍数
        if (PlatformDependent.isWindows()) {
            sleepTimeMs = sleepTimeMs / 10 * 10;
        }

        //开始沉睡，直到哈希轮指针可以移动
        try {
            Thread.sleep(sleepTimeMs);
        } catch (InterruptedException ignored) {
            if (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_SHUTDOWN) {
                return Long.MIN_VALUE;
            }
        }
    }
}
```
下面去看<code>expireTimeouts</code>
```java
public void expireTimeouts(long deadline) {
    HashedWheelTimeout timeout = head;

    // process all timeouts
    //从头到尾去遍历链表
    while (timeout != null) {
        boolean remove = false;
        //这里round是回合的意思，一个新的任务会定位到，几圈零几个格子。这里几圈就是round
        if (timeout.remainingRounds <= 0) {
            //its time to execute task
            if (timeout.deadline <= deadline) {
                //稍后看看这个方法
                timeout.expire();
            } else {
                // The timeout was placed into a wrong slot. This should never happen.
                throw new IllegalStateException(String.format(
                        "timeout.deadline (%d) > deadline (%d)", timeout.deadline, deadline));
            }
            remove = true;
        } else if (timeout.isCancelled()) {
            remove = true;
        } else {
            timeout.remainingRounds --;
        }
        // store reference to next as we may null out timeout.next in the remove block.
        HashedWheelTimeout next = timeout.next;
        //将任务从链表中移除
        if (remove) {
            remove(timeout);
        }
        timeout = next;
    }
}
```
终于到了，任务执行的地方了
```java
//HashedWheelTimeout.class
public void expire() {
    if (!compareAndSetState(ST_INIT, ST_EXPIRED)) {
        return;
    }

    try {
        //这个task是用户传递进来的
        task.run(this);
    } catch (Throwable t) {
        if (logger.isWarnEnabled()) {
            logger.warn("An exception was thrown by " + TimerTask.class.getSimpleName() + '.', t);
        }
    }
}
```
最后再来看看，stop方法
```java
public Set<Timeout> stop() {
    //确保当前线程不是任务线程，防止恶意任务使得哈希轮停止
    if (Thread.currentThread() == workerThread) {
        throw new IllegalStateException(
                HashedWheelTimer.class.getSimpleName() +
                        ".stop() cannot be called from " +
                        TimerTask.class.getSimpleName());
    }
    //这里的CAS操作成功后，run方法内的while循环便会停止，接着去收集那些未被处理的任务
    if (!WORKER_STATE_UPDATER.compareAndSet(this, WORKER_STATE_STARTED, WORKER_STATE_SHUTDOWN)) {
        // workerState can be 0 or 2 at this moment - let it always be 2.
        WORKER_STATE_UPDATER.set(this, WORKER_STATE_SHUTDOWN);

        if (leak != null) {
            leak.close();
        }

        return Collections.emptySet();
    }
    //中断worker线程
    boolean interrupted = false;
    //这里while循环的目的是，让worker线程执行完收集未处理的任务=!workerThread.isAlive()
    while (workerThread.isAlive()) {
        //todo 这里要研究下线程的中断 见文章末尾
        //这里执行调用中断方法的目的是，使得处于休眠状态（等待下次tick动）的worker线程醒过来（waitForNextTick）然后去收集未处理的任务
        workerThread.interrupt();
        try {
            workerThread.join(100);
        } catch (InterruptedException ignored) {
            interrupted = true;
        }
    }

    if (interrupted) {
        Thread.currentThread().interrupt();
    }

    if (leak != null) {
        leak.close();
    }
    //返回未被处理的任务们
    return worker.unprocessedTimeouts();
}
```
至此，Netty中大名鼎鼎的哈希轮就分析完毕了，还是有很多值得借鉴的地方的~

最最后，简单看下Thread.interrupt()方法，其实这个方法是没啥用的一个方法，为啥这么说呢。
举个例子，有线程t1和线程t2，如果t1调用t2.interrupt()方法，t2线程不会立即中断并且根本无视t1线程的这次调用（甚至有点想吃黄焖鸡米饭）
只有一种情况可以让其中断，比如t2调用了sleep方法，这时候如果t1调用t2.interrupt()，t2的sleep方法会抛出异常，注意如果此时t2还未执行到sleep方法，那么当其执行到的时候亦会抛出异常。
<font color=blue>没有任何语言方面的需求一个被中断的线程应该终止。中断一个线程只是为了引起该线程的注意，被中断线程可以决定如何应对中断。某些线程非常重要，以至于它们应该不理会中断，而是在处理完抛出的异常之后继续执行。</font>

# 总结
大体分为两个步骤
一、new HashWheelTimer(xx)
* createWheel，初始化数组
* threadFactory.newThread(worker)，初始化workerThread

二、timer.newTimeout(xx)
* 执行start方法让哈希轮开转（只有第一个任务会执行成功，其他直接return）
* 执行start方法后，计算任务的预计执行时间
* 初始化当前哈希轮任务
* 将任务存放到timeout queue中，等待调度

话分两边，再来看看哈希轮转起来之后干了啥
* 先是初始化starttime（=哈希轮开始运转的nanotime）
* sleep until 执行下次tick移动
* 处理被cancel的任务
* 分配两次tick移动之间，放到timeout queue中的任务到指定的bucket中
* 处理当前tick指向的bucket中到期的任务
* 不断执行上述过程，直到哈希轮状态不为STARTED

