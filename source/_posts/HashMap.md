---
title: 深入分析HashMap
date: 2018-07-10 15:25:51
tags: 多线程
categories: 多线程
---
带着问题深入分析HashMap
Ps有任何问题请在下方评论，或者邮箱联系我luyucareer@163.com
<!-- more -->
<font color=blue>Q:HashMap的数据结构</font>
Node[] table
```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
}
```
这里想要多说几句关于final
1)final修饰类，表示该类不希望被继承
2)final修饰方法，两个作用，第一不希望该方法在子类被覆写；第二在早起java版本中final修饰的方法会转为内嵌调用，可以提升效率
3)final修饰变量，如果是基本数据类型的变量，则其数值一旦在初始化之后便不能更改；如果是引用类型的变量，则在对其初始化之后便不能再让其指向另一个对象。but指向的对象内部可以随意修改
而且还有一点final修饰的变量要么在定义的时候就赋值，要么在static代码块中赋值
<font color=blue>Q:HashMap的threshold/factor/length/size这些参数的含义</font>
size:最常用的属性，表示map中k/v的个数
length:表示table的长度，默认是16
facotr:负载因子，默认为0.75
threshold:等于length*factor,当map的size超过threshold时进行扩容(2倍)
<font color=blue>Q:1.7和1.8 HashMap的区别</font>
1)hash方式
在JDK1.8的实现中，优化了高位运算的算法，通过hashCode()的高16位异或低16位实现的：(h = k.hashCode()) ^ (h >>> 16)，主要是从速度、功效、质量来考虑的，这么做可以在数组table的length比较小的时候，也能保证考虑到高低Bit都参与到Hash的计算中，同时不会有太大的开销。
JDK1.7中hash计算相对复杂这里不做分析
Ps这里实际上1.8的hash更为简化了，因为此时即使发生碰撞也会有红黑树保证效率
2)红黑树的引入
JDK1.7中，当NODE变得特别长的时候，get和put操作复杂度最差会变为O(n)
JDK1.8中，当NODE长度超过设定值（默认8）时候，会调用treeifyBin方法，将链表转为红黑树，最差的情况时间复杂度为O(log n)
3)扩容时的优化 这里要多说几句之前理解的有些问题
我们看1.7的代码可以发现，1.7的hash(key)这个方法和当前的table的容量有关系，所以扩容的时候一方面可能（注意这里说的是可能）需要重新计算每个key的hash值（私以为这也是1.7hash方法的缺点）；另一方面需要对每个key的hash值跟新的capacity-1进行与操作
而1.8中hash方法已经完全和capacity解耦了，一方面在resize的时候一定不需要再去对key进行rehash了；另一方面扩容后只需要和capacity进行与操作，如果结果=0那么该元素仍然位于老的table[i]中；反之，该元素将位于table[i+oldCapacity]中
<font color=blue>Q:以1.8为例，简要说明HashMap put操作和get操作的流程</font>
put操作:
1)根据key计算hash值，再对bucket的length取模（&操作）,定位到bucket的位置i
2)判断bucket[i]是否是null，如果是执行resize进行扩容
3)定位到bucket，之后遍历node，如果有key相同的则覆盖
4)如果没有key相同的node，此时判断该bucket中node的个数，如果个数大于等于8，那么对链表进行树化并插入到树中；反之直接插入到链表末尾
5)插入成功后，判断size是否大于threshold，如果大于进行resize操作
get操作：
1）根据key计算hash值，再对bucket的length取模（&操作）,定位到bucket的位置i
2）如果直接命中，那么直接返回即可
3）未命中，如果是树则在树中查找当前key（O(log n)）;如果是链表那么在链表中查找当前key(O(n))
4）如果没找到返回null
<font color=blue>Q:为什么HashMap的bucket数目要取2^n?而不是对hash更为友好的素数？</font>
对bucket的数量取模的操作，当bucket数量为2^n时候，可以转化为bucket&(2^n-1),对计算机来说&比%具有更高的效率
<font color=blue>Q:为什么说hashMap是线程不安全的</font>
问题表现为
多线程一起执行put操作，另外的线程执行get操作可能会导致死循环及使用迭代器时的fast-fail上（参考之前写过的一篇文章:foreach循环中为什么不要进行remove/add操作）
这个要从hashMap的resize操作说起,resize操作最核心的方法是transfer方法 JDK1.7
```java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}
```
先来看最简单的单线程情况
<img src='resize.png' />
再来看多线程的情况
假设线程1执行到next = e.next，放弃了cpu时间，轮到线程2执行
线程2执行完毕，线程1继续执行，此时循环链表形成,执行对应的get时候将无法退出方法
<img src='loop.png'/>
<font color=blue>Q:hashMap有一个线程写其他线程读会有什么问题吗？</font>
会有可见性问题，hashMap中table/size等都不是volatile的

参考链接：
http://www.cnblogs.com/chengdabelief/p/7419776.html
https://www.cnblogs.com/andy-zhou/p/5402984.html
http://www.jasongj.com/java/concurrenthashmap/

<div id="container"></div>
<link rel="stylesheet" href="https://imsun.github.io/gitment/style/default.css">
<script src="https://imsun.github.io/gitment/dist/gitment.browser.js"></script>
<script>
var gitment = new Gitment({
  id: '深入分析HashMap',
  owner: 'Luyu05',
  repo: 'luyu.net',
  oauth: {
    client_id: '66302af0aad3978dc224',
    client_secret: '2f69936866c0b9a0905542e613096d964aa35b20',
  },
})
gitment.render('container')
</script>

