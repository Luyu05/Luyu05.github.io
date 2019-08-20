---
title: 深入分析ConcurrentHashMap
date: 2018-07-11 17:47:18
tags: 多线程
categories: 多线程
---
本文重点分析JDK1.7的ConcurrentHashMap
说实话挺复杂的，硬着头皮啃吧~
通过和1.7进行比较，简要分析了1.8中的ConcurrentHashMap
Ps有任何问题请在下方评论，或者邮箱联系我luyucareer@163.com
<!-- more -->
### <font color=red size=5>1.7中的ConcurrentHashMap</font>
<font color=blue>1.整体认识</font>
先来看concurrentHashMap的数据结构，从整体上对它有个认识
<img src='structure.png' />
从图中可以看出，concurrentHashMap在hashMap的基础上抽象了一层
这一层就是Segment[],看下Segment这个类
```java
static final class Segment<K,V> extends ReentrantLock implements Serializable {
    //volatile类型的HashEntry数组
    transient volatile HashEntry<K,V>[] table;
    //当前Segment中元素的数量
    transient int count;
    //当前Segment结构变更的次数（put、remove）
    transient int modCount;
    //当count>threashold时候，Segment内部进行rehash
    //table.length*loadFactor
    transient int threshold;
    //负载因子
    final float loadFactor;
}
```
可以看到Segment继承自ReentrantLock，可以方便的进行加锁
每个Segment可以视为一个hashMap，ConcurrentHashMap正是通过Segment实现的同时进行多个读写操作
<font color=blue>2.初始化</font>
直接撸代码
```java
public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (concurrencyLevel > MAX_SEGMENTS)
            concurrencyLevel = MAX_SEGMENTS;
        // Find power-of-two sizes best matching arguments
        int sshift = 0;
        int ssize = 1;
        //concurrencyLevel默认是16，即有16个Segment
        //此时sshift=4,ssize=16
        while (ssize < concurrencyLevel) {
            ++sshift;
            ssize <<= 1;
        }
        //segmentShift=28,segmentMask=15
        this.segmentShift = 32 - sshift;
        this.segmentMask = ssize - 1;
        //initialCapacity表示整个map中容纳kv的数量
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        //对每个segment取均值
        int c = initialCapacity / ssize;
        if (c * ssize < initialCapacity)
            ++c;
        //默认每个槽的大小为2，确保最终每个segment中table的数量为2的n次方
        int cap = MIN_SEGMENT_TABLE_CAPACITY;
        while (cap < c)
            cap <<= 1;
        // create segments and segments[0]
        // segment的threshold=cap*loadFactor
        // HashEntry[]数组长度也为cap
        Segment<K,V> s0 =
            new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                             (HashEntry<K,V>[])new HashEntry[cap]);
        Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
        UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
        //将segments赋给当前concurrentHashMap，注意setment[0]已经初始化
        this.segments = ss;
    }
```
小结：
1）默认情况下Segment数组长度为16，不可扩容
2）假如initialCapacity=64,那么c=4,cap=4 则Segment[0]的thresHold=4*0.75=3
<font color=blue>3.安全并发put</font>
```java
public V put(K key, V value) {
        Segment<K,V> s;
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key);
        //找到当前key所属的segment
        int j = (hash >>> segmentShift) & segmentMask;
        //如果segment[j]为空，对segment[j]进行初始化
        if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
             (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
            //注意这里还没有进行加锁，所以可能多个线程同时对一个segment[i]执行初始化操作
            s = ensureSegment(j);
        return s.put(key, hash, value, false);
    }
```
先来看ensureSegment操作，相对简单直接写注释了
```java
private Segment<K,V> ensureSegment(int k) {
        final Segment<K,V>[] ss = this.segments;
        //可以理解为定位标识
        long u = (k << SSHIFT) + SBASE; 
        Segment<K,V> seg;
        //如果ss[u]==null 才能继续 否则表示已经初始化过了 直接return即可
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
            //初始化的时候只初始了一个segment[0]，其实是为了给其他segment做标杆
            //只有segment[i]用到的时候才赋值
            //此时segment[0]可能早就扩容过了
            Segment<K,V> proto = ss[0]; // use segment 0 as prototype
            int cap = proto.table.length;
            float lf = proto.loadFactor;
            int threshold = (int)(cap * lf);
            HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
            //segment[u]是否真的未初始化
            if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                == null) { // recheck
                Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
                //通过CAS操作，对segment[u]进行初始化
                while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                       == null) {
                    if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                        break;
                }
            }
        }
        return seg;
    }
```
Segment的put操作
```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
        //对当前segment加锁
        //scanAndLockForPut方法后续分析
        HashEntry<K,V> node = tryLock() ? null :
            scanAndLockForPut(key, hash, value);
        V oldValue;
        try {
            HashEntry<K,V>[] tab = table;
            //定位到segment中的HashEntry<K,V>[] tab
            int index = (tab.length - 1) & hash;
            //保证获取到的链表头是最新的Unsafe.getObjectVolatile
            HashEntry<K,V> first = entryAt(tab, index);
            for (HashEntry<K,V> e = first;;) {
                //表头非空，不断遍历，如果有相同的就覆盖，如果没有继续遍历直到为null
                if (e != null) {
                    K k;
                    //== 和 equals & hash
                    if ((k = e.key) == key ||
                        (e.hash == hash && key.equals(k))) {
                        oldValue = e.value;
                        //如果并未指定"不存在才插入"，那么正常覆盖即可
                        //并且记录segment结构变化的modCount居然++了？
                        if (!onlyIfAbsent) {
                            e.value = value;
                            ++modCount;
                        }
                        break;
                    }
                    e = e.next;
                }
                else {
                    //如果node不为null，将node置于链表头部
                    if (node != null)
                        node.setNext(first);
                    else//如果node为空，也将node置于链表头部
                        node = new HashEntry<K,V>(hash, key, value, first);
                    int c = count + 1;
                    //如果超过了threshold并且tab的长度没超过最大值，进行扩容操作
                    //rehash后续分析
                    if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                        rehash(node);
                    else//node置为tab的表头,下次再获取first的时候就是node
                        setEntryAt(tab, index, node);
                    ++modCount;
                    count = c;
                    oldValue = null;
                    break;
                }
            }
        } finally {
            unlock();
        }
        return oldValue;
    }
```
言出必行，接下来分析scanAndLockForPut和rehash
首先看看scanAndLockForPut
```java
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
        //通过segment和hash值定位到tab[i]
        HashEntry<K,V> first = entryForHash(this, hash);
        HashEntry<K,V> e = first;
        HashEntry<K,V> node = null;
        int retries = -1; // negative while locating node
        //自旋操作获取锁
        while (!tryLock()) {
            HashEntry<K,V> f; // to recheck first below
            if (retries < 0) {
                if (e == null) {
                    if (node == null) // speculatively create node
                        node = new HashEntry<K,V>(hash, key, value, null);
                    retries = 0;
                }
                else if (key.equals(e.key))
                    retries = 0;
                else
                    e = e.next;
            }
            //重试次数过多，那么不重试了，直接进入bq
            //就是reentrant.lock
            else if (++retries > MAX_SCAN_RETRIES) {
                lock();
                break;
            }
            //链表的头部是新的node，那么重新执行循环
            else if ((retries & 1) == 0 &&
                        (f = entryForHash(this, hash)) != first) {
                e = first = f; // re-traverse if entry changed
                retries = -1;
            }
        }
        return node;
}
```
简单地说就是，通过自旋获取锁，这里new了一个node私以为没啥大用
接下来看rehash方法，在hashmap中并发时候resize经常会出问题，但是在concurrentHashMap则不存在这个问题，因为只能有一个线程执行rehash方法
```java
private void rehash(HashEntry<K,V> node) {
    HashEntry<K,V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    int newCapacity = oldCapacity << 1;
    threshold = (int)(newCapacity * loadFactor);
    HashEntry<K,V>[] newTable =
        (HashEntry<K,V>[]) new HashEntry[newCapacity];
    int sizeMask = newCapacity - 1;
    for (int i = 0; i < oldCapacity ; i++) {
        HashEntry<K,V> e = oldTable[i];
        if (e != null) {
            HashEntry<K,V> next = e.next;
            int idx = e.hash & sizeMask;
            if (next == null)   //  Single node on list
                newTable[idx] = e;
            else { // Reuse consecutive sequence at same slot
                //找到最后一个在新table位置不同的node
                //换句话说在它后面都是在当前table的node
                //这个其实就是随缘，很大可能最后一个才是甚至最后一个也不是
                HashEntry<K,V> lastRun = e;
                int lastIdx = idx;
                for (HashEntry<K,V> last = next;
                        last != null;
                        last = last.next) {
                    int k = last.hash & sizeMask;
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                //位置不动的node直接放到新的table但是索引相同
                newTable[lastIdx] = lastRun;
                // Clone remaining nodes
                //从头遍历lastRun前面的链，放到属于它的table
                //1.7还有点笨，还在老老实实的重新计算hash值
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    V v = p.value;
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                }
            }
        }
    }
    //put操作引发的rehash，rehash之后再put，放到链表头部
    int nodeIndex = node.hash & sizeMask; // add the new node
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = node;
    table = newTable;
}
```
<font color=blue>4.get操作</font>
```java
public V get(Object key) {
    Segment<K,V> s; // manually integrate access methods to reduce overhead
    HashEntry<K,V>[] tab;
    int h = hash(key);
    //定位segment
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        //定位在table中的位置,找到所属的entry/node
        //通过UNSAFE.getObjectVolatile保证刚插入的node也能被看到
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                    (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
                e != null; e = e.next) {
            K k;
            //== 和 equals & hash
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```
<font color=blue>5.remove操作</font>
这个方法很简单
```java
final V remove(Object key, int hash, Object value) {
    //当前segment加锁，失败则自旋
    if (!tryLock())
        scanAndLock(key, hash);
    V oldValue = null;
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;
        HashEntry<K,V> e = entryAt(tab, index);
        HashEntry<K,V> pred = null;
        //简单地说就是把目标从链上摘除
        while (e != null) {
            K k;
            HashEntry<K,V> next = e.next;
            if ((k = e.key) == key ||
                (e.hash == hash && key.equals(k))) {
                V v = e.value;
                if (value == null || value == v || value.equals(v)) {
                    if (pred == null)
                        setEntryAt(tab, index, next);
                    else
                        pred.setNext(next);
                    ++modCount;
                    --count;
                    oldValue = v;
                }
                break;
            }
            pred = e;
            e = next;
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```
<font color=blue>6.size()操作</font>
有意思的一个方法~吃过饭回来撸
```java
public int size() {
    // Try a few times to get accurate count. On failure due to
    // continuous async changes in table, resort to locking.
    final Segment<K,V>[] segments = this.segments;
    int size;
    boolean overflow; // true if size overflows 32 bits
    long sum;         // sum of modCounts
    long last = 0L;   // previous sum
    int retries = -1; // first iteration isn't retry
    try {
        for (;;) {
            //如果重试的次数超过3，那么对所有segment加锁
            //而且如果此时segment还未初始化，先对其初始化
            if (retries++ == RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    ensureSegment(j).lock(); // force creation
            }
            sum = 0L;
            size = 0;
            overflow = false;
            //遍历segment，将每个segment的modCount累加到一起
            //modCount表示当前segment数据结构变动的次数
            for (int j = 0; j < segments.length; ++j) {
                Segment<K,V> seg = segmentAt(segments, j);
                if (seg != null) {
                    sum += seg.modCount;
                    int c = seg.count;
                    if (c < 0 || (size += c) < 0)
                        overflow = true;
                }
            }
            //如果本次循环统计的modeCount和上次循环统计的modCount相等
            //表示这期间并未发生任何变动
            //那么我们可以认为本次统计的size就是实际的size
            if (sum == last)
                break;
            last = sum;
        }
    } finally {
        //释放segment的锁
        if (retries > RETRIES_BEFORE_LOCK) {
            for (int j = 0; j < segments.length; ++j)
                segmentAt(segments, j).unlock();
        }
    }
    return overflow ? Integer.MAX_VALUE : size;
}
```
简单地说，就是先不加锁，因为这个方法加锁的代价相当高昂。先去在循环中统计modCount，如果两次循环之间modCount未发生变动，那么将当前的size返回
小结：
1.put的时候分段加锁，如果加锁失败会有一个自旋的过程
2.put操作node是插入到entry的头部
3.在put操作触发rehash的时候（resize）因为put本身已经保证了线程安全，所以resize的时候也是线程安全的，不会出现hashmap的环形链(resize+get死循环)
<font color=blue>7.并发问题的分析</font>
1)牛X的Unsafe包
我们可以简单的把unsafe包中的操作，视为对地址的直接操作，类似C++和C中的指针
保证了多线程间的可见性
2）get操作
我们可以看到get操作是没有加锁的，那么是怎么保证线程安全的？
a)get和put并发
先get再put，因为put操作是直接插到表头的，所以不影响正常遍历原链表
先put再get，通过unsafe找到segment，而segment中的table是volatile的，所以能够保证执行get的时候可以看到新增的kv
b)get和remove并发
如果remove的节点，已经被get遍历过了，那么没啥问题
反之，有分成两种情况，如果remove的是头节点，通过unsafe保证了可见；而如果remove的是中间段的节点，这里是通过Entry的volatile修饰的next保证可见的
### <font color=red size=5>1.8中的ConcurrentHashMap</font>
1)和1.7不同，1.8中的ConcurrentHashMap的数据结构和其HashMap完全相同，单单通过代码保证了线程安全性
2)对每个table[i]通过synchronized关键字加锁，也无须自己实现自旋操作
3)和1.8的hash相同，当链表长度太长的时候，会变为红黑树
4)涉及的红黑树的内容暂时搁置，未去分析
### <font color=red size=5>ConcurrentHashMap自问自答</font>
1.为何get方法不需要加锁？
在HashEntry类中，key，hash域都被声明为final的，value，next域被volatile所修饰，
因此HashEntry对象几乎是不可变的，这是ConcurrentHashmap读操作并不需要加锁的一个重要原因

2.size()的原理？
会去统计每个segment的modCount和，如果下次循环统计的modCount和跟上次统计的一样，认为此时的size（每个segment的count和）就是一个准确的值
反之如果和上次统计的不一样，当循环执行到一定次数的时候会对每个segment进行加锁操作，再去统计size

3.rehash触发的条件？
默认segment为16（不会再变动），每个segment中的HashEntry数组是扩容的目标，当segment中的元素size > length(数组的长度)*factor
这个时候开始扩容，将length*2

4.一些细节
segment其实是懒加载的，只有用到的时候才会初始化，不过当调用size方法的时候会全部初始化
1.6中HashEntry的next是final修饰的，所以新增的时候只能在最前面新增且删除的时候比较笨重需要全量复制；1.7中改为了volatile修饰
在进行rehash过程中，不会被其他get方法读取到中间状态，因为是一个新的复制操作，只有老table和新table
put的时候分段加锁，如果加锁失败会有一个自旋的过程
put操作node是插入到entry的头部
在put操作触发rehash的时候（resize）因为put本身已经保证了线程安全，所以resize的时候也是线程安全的，不会出现hashmap的环形链

参考链接:
http://www.jasongj.com/java/concurrenthashmap/
http://www.importnew.com/28263.html

<div id="container"></div>
<link rel="stylesheet" href="https://imsun.github.io/gitment/style/default.css">
<script src="https://imsun.github.io/gitment/dist/gitment.browser.js"></script>
<script>
var gitment = new Gitment({
  id: '深入分析ConcurrentHashMap',
  owner: 'Luyu05',
  repo: 'luyu.net',
  oauth: {
    client_id: '66302af0aad3978dc224',
    client_secret: '2f69936866c0b9a0905542e613096d964aa35b20',
  },
})
gitment.render('container')
</script>