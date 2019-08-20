---
title: 读写锁
date: 2018-09-27 11:54:00
tags: 多线程
categories: 多线程
---
今天看到一个词，『锁降级』，于是就Google了下，然后顺带着把之前没搞懂的一个知识点搞清楚了，在这里记录下。
简单地讲，就是获取写锁后对数据进行变更，再获取读锁，再释放写锁，平滑地完成锁降级，以实现同一个线程的写读一致
<!-- more -->
先上例子（不得不说，官方的例子真是简洁优雅没任何废话）
```java
class CachedData {
   Object data;
   volatile boolean cacheValid;
   final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();

   void processCachedData() {
     rwl.readLock().lock();
     if (!cacheValid) {
        // Must release read lock before acquiring write lock
        rwl.readLock().unlock();
        rwl.writeLock().lock();
        try {
          // Recheck state because another thread might have
          // acquired write lock and changed state before we did.
          if (!cacheValid) {
            data = ...
            cacheValid = true;
          }
          // Downgrade by acquiring read lock before releasing write lock
          rwl.readLock().lock();
        } finally {
          rwl.writeLock().unlock(); // Unlock write, still hold read
        }
     }

     try {
       use(data);
     } finally {
       rwl.readLock().unlock();
     }
   }
 }
```
几个值得注意的点
1）当存在读锁时，获取写锁的线程会被阻塞（防止读的时候数据被修改）
2）读锁是共享锁，所以当存在读锁时，获取读锁的线程不会被阻塞（极大提升了系统吞吐量）
3）锁降级的目的，保证了一个『原子性』，写-读原子性（如果先放弃写锁再获取读锁，可能会被其他线程抢占写锁导致数据变更）;主要是为了保证当前线程，写读的一致性
4）当前线程持有写锁时，可以获取读锁；而当前线程持有读锁时，无法获取写锁。很容易理解，当前线程持有写锁时，其他线程不会获取到读/写锁，所以可以直接获取读锁，不会对其他执行读的线程产生影响；而当前线程持有读锁时，其他线程可能也持有读锁，所以在获取写锁时要先放弃读锁