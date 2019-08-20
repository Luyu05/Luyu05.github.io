---
title: 深入分析java线程池
date: 2018-07-09 11:11:19
tags: 多线程
categories: 多线程
---
1.线程池状态
2.构造函数
3.常用的几种线程池
4.任务执行
5.后处理
6.线程池的终止
Ps有任何问题请在下方评论，或者邮箱联系我luyucareer@163.com
<!-- more -->
### <font color=red size=5>线程池状态</font>
线程池的几种状态  RUNNING SHUTDOWN STOP TIDYING TERMINATED
几种状态之间的流转关系如下图
<img src='threadState.png' />
```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;  //111
private static final int SHUTDOWN   =  0 << COUNT_BITS;  //000
private static final int STOP       =  1 << COUNT_BITS;  //001
private static final int TIDYING    =  2 << COUNT_BITS;  //010
private static final int TERMINATED =  3 << COUNT_BITS;  //011

// Packing and unpacking ctl
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
```
重点关注AtomicInteger ctl 共32位，其中高3位表示"线程池状态"，低29位代表"线程池中的任务数量"
各个状态详细说明如下:
```java
*   RUNNING:  Accept new tasks and process queued tasks
*   SHUTDOWN: Don't accept new tasks, but process queued tasks
*   STOP:     Don't accept new tasks, don't process queued tasks,
*             and interrupt in-progress tasks
*   TIDYING:  All tasks have terminated, workerCount is zero,
*             the thread transitioning to state TIDYING
*             will run the terminated() hook method
*   TERMINATED: terminated() has completed
```
### <font color=red size=5>构造函数</font>
```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```
参数说明
Ps 线程池中有两大容器，分别是workers(Set:存放线程 保存消费者) 和 queue(blockingQueue:存放任务 保存生产的任务)

<font color=blue>corePoolSize</font>
   核心线程的数量，线程池初始化后，每接到一个任务就会创建一个线程来执行任务，直到当前的线程数目到达corePoolSize，此时新的任务将会进入queue中，只有当queue满了之后，maximunPoolSize才发挥作用
核心线程被保存在pool中，即使线程处于闲置状态也不会被回收，除非allowCoreThreadTimeOut被设置，从名字可以看出这是用来控制核心线程是否可以超时被回收的一个参数。
Ps核心线程可以理解为工厂的长工

<font color=blue>maximumPoolSize</font>
     pool中所允许的最大线程数。线程池的queue满了之后，如果还有新的任务到来，此时如果线程数目小于maximumPoolSize，则会新建线程来执行任务。
Ps 非核心线程可以理解为工厂的短工 最大值=maximumPoolSize-corePoolSize

<font color=blue>keepAliveTime</font>
    线程空闲的时间，默认情况该参数只针对"短工"有效（短工空闲太久就要被辞退），只有当配置allowCoreThreadTimeOut时该参数才对"长工"生效

<font color=blue>unit</font>
    keepAliveTime的单位

<font color=blue>workQueue</font>
    上文提到的queue，用来保存等待执行的任务的阻塞队列

<font color=blue>ThreadFactory</font>
    线程工厂，可以用户自己配置，默认的ThreadFactory 1.给线程命名 2.将线程设置为非守护线程 3.优先级设置为NORM

<font color=blue>handler</font>
    拒绝策略：当线程数=maximumPoolSize 且 queue已满 这时候新提交的任务会被拒绝（消费者已达到max，而待消费的任务也达到max）
    1.AbortPolicy：直接抛出异常，默认策略；
    2.CallerRunsPolicy：用调用者所在的线程来执行任务；
    3.DiscardOldestPolicy：丢弃阻塞队列中靠最前的任务，并执行当前任务；
    4.DiscardPolicy：直接丢弃任务；
    
### <font color=red size=5>常用的几种线程池</font>
<font color=blue>FixedThreadPool</font>
```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```
此时corePoolSize = maximumPoolSize , 这意味着不再有任何"短工"，当线程池中的线程数量达到corePoolSize后，新来的任务被放到queue中，当queue满了之后，线程池将不再接收任何任务，
如果再继续提交任务，则会直接执行拒绝策略。这里执行的是默认的拒绝策略，直接抛出异常。
这里的queue使用的是LinkedBlockingQueue，queue的容量是MAX_VALUE，所以构造函数中的maximumPoolSize与keepAliveTime几乎无效
而ThreadPoolExectutor中的allowCoreThreadTimeOut也未被设置，所以默认为false
<font color=blue>SingleThreadExecutor</font>
```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```
相当于newFixedThreadPool(1)
case：在for循环内部创建了过多线程池且未shutdown
上面的case 就是在for循环内部创建了过多的singleThreadExecutor，每个singleThreadExecutor会维持一个核心线程（不调用shutdown，该线程不会释放的），最后导致OOM
<font color=blue>CachedThreadPool</font>
```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```
任务一旦提交，首先会进入queue，此时的queue是synchronousQueue（不存储任何元素），所以会直接创建新的"短工"（线程），当这些线程闲置超过60s之后会被终止。
CachedThreadPool 会有一个问题 可能会创建大量的线程 可能导致耗尽CPU 和 内存。所以 一定要控制并发量
### <font color=red size=5>任务执行</font>
线程池有两种执行任务的方式，execute() & submit(),submit一般用于有返回值的场景并且该方法是从ThreadPoolExecutor的父类中继承的。
以execute()为例子，分析内部的执行原理
```java
//ThreadPoolExecutor.class
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    //获取当前线程池ctl
    //value是volatile类型的
    int c = ctl.get();
    //正常情况，新任务新线程
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    //bq offer 线程安全
    //核心线程max，塞到队列
    if (isRunning(c) && workQueue.offer(command)) {
        //入队成功，二次check状态
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        //插入队列后发现 worker数量为0，需要新建worker
        //比如说cachedThreadExecutor，经常会有这种情况
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    //队列满了，新建非核心线程
    else if (!addWorker(command, false))
        reject(command);
}
```
<img src='execute.png' />
整个代码分三个步骤：
a)如果当前线程数目小于corePoolSize,则直接调用addWorker创建新线程来执行任务，如果addWorker失败，再次读取ctl,并执行步骤b
b)如果线程池处于RUNNING状态且当前任务进入queue成功，此时会对当前的ctl进行二次检查，
           如果在这个过程中线程池不处于RUNNING状态，那么从queue中移除任务并执行拒绝策略
           反之，如果二次检查通过，如果当前线程数目为0，则会创建新的工作线程，否则queue中的任务将无法执行。Ps等下分析addWorker(null, false)
c)入队失败，此时调用addWorker(command,false)创建非核心线程，若创建失败执行拒绝策略。Ps显然addWorker方法需要再次对线程池状态进行判断，不然b中如果非RUNNING状态，也会执行到c

addWorker()  Ps有意思的一段code
```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    //双层死循环，当新增worker成功时break，状态不满足时直接return，cas失败&状态变更时continue
    //这里的目的是通过cas增加automicInteger类型的ctl的值
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
    //只有cas成功新建的worker才能开始工作
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        //类级别的锁
        final ReentrantLock mainLock = this.mainLock;
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int c = ctl.get();
                int rs = runStateOf(c);

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```
<img src='detail1.png' />
a)读取当前线程池状态
           如果rs>=SHUTDOWN（线程池处于SHUTDOWN,STOP,TIDYING,TERMINATED状态），这个时候addWorker是要失败的，除了一种情况，当前线程池处于SHUTDOWN状态，且firstTask=null且queue不为空，我们知道这种情况，线程池是要执行完queue中的任务的，所以这种情况是可以增加worker的。
b) 内层循环，如果当前线程的数量满足要求（核心线程数<corePoolSize 非核心线程<maximumPoolSize），那么通过CAS使woker自增，成功跳出双重循环，否则继续自旋
c) 获取锁，再次判断线程状态，如果rs<SHUTDOWN 或者rs=SHUTDOWN且当前入参的firstTask=null 才可以继续向下执行，将woker添加到wokers(Set)中，然后释放锁并启动线程；否则直接释放锁。Ps这里加锁的目的一是为了wokers添加元素，二是为了更新largestPoolSize
d) t.start()启动任务

Worker
```java
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
{
    private static final long serialVersionUID = 6138294804551838833L;

    /** Thread this worker is running in.  Null if factory fails. */
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;

    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        //构造函数传入的ThreadFactory去创建线程
        this.thread = getThreadFactory().newThread(this);
    }

    /** Delegates main run loop to outer runWorker  */
    public void run() {
        runWorker(this);
    }
	/**省略部分代码*/
```
Worker继承自AQS且实现了Runnable接口 Ps这里state设置为-1的目的是这时候禁止中断，直到线程开始运行才可以中断
线程启动的时候，实际上是调用了runWorker方法
```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;//突然完成
    try {
        while (task != null || (task = getTask()) != null) {
            //shutDwon的时候，会去遍历works中的work执行tryLock如果成功
            //代表当前线程处于idle状态，那么将其中断
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```
a)方法一开始就调用unlock方法，此时会将state置为0（代表解锁状态，此时允许线程中断）interruptWorkers()方法只有在state>=0才能执行
b)有趣的循环，首先执行传递进来的任务，如果当前任务执行完毕，会调用getTask方法去queue中提取任务执行
c)当前有任务执行时，首先获取锁置state=1
          如果当前线程池状态>=STOP 且当前线程未被中断，那么中断当前线程
          如果当前线程池状态<STOP 且当前线程被中断了(interrupted会将中断位置为false) 再次检查当前线程池状态 如果>=STOP 那么中断当前线程
d) 终于可以执行任务了，这里有两个模板方法，beforeExecute and afterExecute，ThreadPoolExecutor中给了空实现
e)最后置task=null 并将当前Woker的工作量++ 最后释放锁 接下来继续循环获取queue中的任务

getTask()
```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 1）rs >= STOP  2) rs = SHUTDOWN && workQueue为空
        // 无需去队列拿去任务，此时可以销毁当前worker
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        boolean timed;      // Are workers subject to culling?

        for (;;) {
            int wc = workerCountOf(c);
            //如下两种情况需要引入超时机制
            //1）核心线程具有超时机制 2）worker数量大于corePoolSize
            timed = allowCoreThreadTimeOut || wc > corePoolSize;
            //当前worker数量小于max && !(符合超时机制且超时)
            if (wc <= maximumPoolSize && ! (timedOut && timed))
                break;
            //超时执行CAS
            if (compareAndDecrementWorkerCount(c))
                return null;
            // else CAS failed due to workerCount change; retry inner loop
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
        }

        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            //此时再次进入for循环，会执行workerCount-1 & return null
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```
a) 首先根据线程池的状态进行下一步的决策，如果当前线程池状态>=STOP or =SHUTDOWN且queue为空 那么woker数量--  并返回null
b)接下来判断是否需要超时机制，如果allowCoreThreadTimeOut 或者 当前线程数大于corePoolSize（说明有短工） 那么需要有超时控制
c)如果当前线程的数量处于正常状态，<=maximumPoolSize 且timeout timed其中有一个是false(非有超时机制且已经超时的情况) 那么跳出内层循环 直接去从queue中获取任务 并return  Ps如果当前线程数目大于max 或 线程有超时机制并已经超时 那么workerCount -- 
d) 如果从queue中取值结果为null  那么timeout会被置true  再次执行内层循环 简单说就是queue中无任务 且 有超时机制 那么workcount要-- （这里ctl-- 与之等价）  Ps 阻塞队列的take 在队列为空的时候 会一直阻塞 而poll 超时后会返回一个null
<img src='tasks.png' />
### <font color=red size=5>后处理</font>
在runwoker方法中 finally代码块中调用了 processWorkerExit方法 用来对Worker进行后处理
```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }

    tryTerminate();

    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        addWorker(null, false);
    }
}
```
a)如果是被意外终止（用户异常），那么将worker的数量-1
b)加锁去更新共享参数completedTaskCount 加上当前worker处理的任务数量 并从set中移除当前woker
c)调用tryTerminate方法（下文有源码）
d)再次判断当前ctl，如果当前状态<STOP,1）意外中断，那么需要补上一个worker；2）非意外中断，但当前worker已经比最小值还低，那么需要补上一个worker

tryTerminate()
```java
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        //1.running 2.tidying or terminated 3.shutdown and queue is not empty
        if (isRunning(c) ||
            runStateAtLeast(c, TIDYING) ||
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
            return;
        //wokercount > 0  有资格去终止
        if (workerCountOf(c) != 0) { // Eligible to terminate
            //terminate one worker
            interruptIdleWorkers(ONLY_ONE);
            return;
        }

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //set ctl -> ctlof(TIDYING,0)
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                    terminated();
                } finally {
        			//set ctl -> ctlof(TERMINATED,0)
                    ctl.set(ctlOf(TERMINATED, 0));
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
        // else retry on failed CAS
    }
}
```
a)如果当前状态either (SHUTDOWN and pool and queue empty) or (STOP and pool empty) 那么直接return
b)如果当前woker数！=0 那么可以去中断，那么调用interruptIdleWorkers(ONLY_ONE) 去终止Worker
c)如果当前worker = 0 那么去终止线程池 首先将ctl置为TIDYING&0 接下来调用terminated()终止线程池 ，最后将ctl置为TERMINATED&0
### <font color=red size=5>线程池的终止</font>
shutdown()
```java
public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            //SHUTDOWN
            advanceRunState(SHUTDOWN);
            // 中断空闲的线程
            interruptIdleWorkers();
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }
```
shutDownNow()
```java
public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            // STOP
            advanceRunState(STOP);
            // 中断所有线程
            interruptWorkers();
            //取出queue中的任务 最后return
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }
```
### <font color=red size=5>几个细节</font>
1.当调用shutdown的时候，有可能将所有的worker线程都中断或者当前worker < corePoolSize
不过此时worker线程会执行processWorkerExit(curWorker,conpletedAbruptly true)
该方法内部会将worker补齐至所需线程的最小值(1 or corePoolSize) Ps addWork(null,false)
这里就是经典的 rs == SHUTDOWN && firstTask == null && ! workQueue.isEmpty()
2.当调用shutdown时候有一行关键代码interruptIdleWorkers(),这个方法会执行worker.tryLock如果成功则对worker线程执行interrupt()；而当worker线程工作的时候会首先执行worker.lock()，也就是说正在工作的worker是不会被中断的
3.shutdownNow不care queue中的排队线程，不过会返回当前存储任务的blocking queue
4.workers中的worker有的是带着任务进入的集合，这部分要先执行所带的任务，之后再去queue中取任务执行；而没有任务的worker直接去queue中拉取任务执行
5.Worker类继承自AQS，此时volatile类型的state表示worker是否可以中断
6.对workers(普通的hashSet)的add/remove操作都需要上锁
7.实际上在workers内部不区分core和non-core，当worker数量大于corePoolSize且超时时候，通过CAS操作对部分执行processWorkerExit
8.通过block queue的poll(time)实现worker的超时机制
9.普通的超时退出，不会对线程执行interrupt()，此时只是简单地从workers中remove；只有主动调用shutdown/shutdownNow才会

### <font color=red size=5>线程池自问自答</font>
1.线程池如何处理queue中的任务？
worker执行完当前任务后，会从queue中获取任务并执行

2.线程池如何引入超时机制？
通过从queue中获取任务过程引入，Blockingqueue.poll(time,timeUnit)；没有超时机制的时候调用take()方法阻塞在那里

3.shutdown和shutdownNow区别
前者不再接收新的任务，但是会将当前池子内的任务执行完毕；后者直接中断所有worker线程，此时会将queue中任务返回

4.线程池有几种状态？
RUNNING SHUTDOWN STOP TIDYING TERMINATE

5.Worker为啥继承AQS
在woker执行任务的时候加锁，比如调用shutDwon方法时候，会遍历所有的worker如果当前worker没有加锁（视为idle线程）会被执行中断操作

6.jdk提供的几种线程池的区别
FixedThreadPool 只有核心线程，没有短工（所以没有超时时间），queue使用的是无界的阻塞队列
CachedTheadPool 没有核心线程，只有短工（默认超时时间为60s） queue是没有存储能力的阻塞队列

7.什么情况会调用addWorker(null,false)
发现当前线程池中的Worker数量不够的时候
当新增任务的时候，入队之后发现queue中worker数量为0；
worker线程退出时候会检测当前线程的数量，如果小于最小值（0 or corePoolSize）需要addWorker(null,false)

8.CachedTheadPool中的SynchronousQueue既然没有存储能力，那有什么用？
线程池的超时机制就是通过queue来实现的，queueu.poll(time,unit);所以SynchronousQueue可以用来控制超时机制

小结
线程池的设计真的是巧妙，通过一些参数比如addWorker(Runnable firstTask, boolean core)表明是否是核心线程；
completedAbruptly表示当前线程是否是意外退出，如果意外退出会有一些补偿机制
```java
try {
        while (task != null || (task = getTask()) != null) {
            blala
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
```
再比如，interruptIdleWorkers(ONLY_ONE)和interruptIdleWorkers()分别表示中断一个线程和中断所有线程;

<div id="container"></div>
<link rel="stylesheet" href="https://imsun.github.io/gitment/style/default.css">
<script src="https://imsun.github.io/gitment/dist/gitment.browser.js"></script>
<script>
var gitment = new Gitment({
  id: '深入分析java线程池',
  owner: 'Luyu05',
  repo: 'luyu.net',
  oauth: {
    client_id: '66302af0aad3978dc224',
    client_secret: '2f69936866c0b9a0905542e613096d964aa35b20',
  },
})
gitment.render('container')
</script>


