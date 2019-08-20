---
title: 线程之间的协作
date: 2018-07-05 21:01:52
tags: 多线程
categories: 多线程
---
本文讲述下线程间协作的集中方式
Thread,join()
CountdownLatch
CyclicBarrier
Semaphore
<!-- more -->
### <font color=red size=5>Thread.join()</font>
适用于一个线程等待另外n个线程执行结束再去做某件事的场景，需要持有其他线程的引用
```java
public class joinTest {
    private static Thread[] ts = new Thread[10];
    private static AtomicInteger ai = new AtomicInteger(0);

    public joinTest() {
        for (int i = 0; i < 10; i++) {
            ts[i] = new Thread() {
                @Override
                public void run() {
                    try {
                        Thread.sleep(100);
                    } catch (Exception e) {
                    }
                    ai.incrementAndGet();
                }
            };
        }
    }

    public static void main(String[] args) throws Exception {
        new joinTest();
        for(int i;i<10;i++){
            ts[i].start();
        }
        for(int i;i<10;i++){
            ts[i].join();
        }
        System.out.println(ai);
    }
}
```
输出结果10
### <font color=red size=5>CountdownLatch</font>
和Thread.join类似，一个线程在另外n个线程都执行过某个点后才去执行（不需要等待线程执行完毕），此时不需要持有其他线程的引用
```java
public class controlThread {
    private static final int ThreadCount = 50;
    private static AtomicInteger result = new AtomicInteger(0);
    private static CountDownLatch endCount = new CountDownLatch(ThreadCount);

    static void helper(){
        for(int i=0;i<50;i++){
            new Thread() {
                @Override
                public void run() {
                    result.incrementAndGet();
                    endCount.countDown();
                    try {
                        Thread.sleep(5000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println("awake");
                }
            }.start();
        }
    }

    public static void main(String[] args) throws Exception{
        new Thread(){
            @Override
            public void run(){
                helper();
            }
        }.start();
        endCount.await();
        print(result);
    }
}
```
输出结果50 
过5s后 50个awake
### <font color=red size=5>CyclicBarrier</font>
适用于一起开始的业务场景，n个线程大家互相等待，都完成了才继续进行
比如多线程计算的场景
而且是可重用的
```java
public class cyclicBarrierTest {
    private static CyclicBarrier cyclic = new CyclicBarrier(5,new BarrierRun());
    private static volatile boolean flag = false;
    static class BarrierRun implements Runnable{
        @Override
        public void run(){
            if(!flag)
            print("we five are ready!");
            else
                print("we five are ready again!");
        }
    }
    public static void main(String[] args) throws Exception {
        for (int i = 0; i < 5; i++) {
            new Thread() {
                @Override
                public void run() {
                    try {
                        print(Thread.currentThread()+"'s worker has comed");
                        cyclic.await();//会等BarrierRun执行完毕再继续向下执行
                        flag = true ;
                        print(Thread.currentThread()+"wait you five so long!");
                        cyclic.await();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }

                }
            }.start();
        }

    }
}
```
输出结果
Thread[Thread-1,5,main]'s worker has comed
Thread[Thread-2,5,main]'s worker has comed
Thread[Thread-3,5,main]'s worker has comed
Thread[Thread-0,5,main]'s worker has comed
Thread[Thread-4,5,main]'s worker has comed
we five are ready!
Thread[Thread-2,5,main]wait you five so long!
Thread[Thread-1,5,main]wait you five so long!
Thread[Thread-3,5,main]wait you five so long!
Thread[Thread-4,5,main]wait you five so long!
Thread[Thread-0,5,main]wait you five so long!
we five are ready again!
### <font color=red size=5>Semaphore</font>
一般用于控制同时访问特定资源的线程数量
```java
public class SemaphoreTest {    
 private final Semaphore sem = new Semaphore(5);   //声明5个许可数量        
 public void doBusiness() throws InterruptedExcep:on { 
   sem.acquire();             //获得一个许可（许可数量减一），没有许可就阻塞     
   try { 
     doExecute();       //单机同一时刻最多有5个线程能进入
   }  finally { 
     sem.release();     //归还一个许可
   } 
} 
```