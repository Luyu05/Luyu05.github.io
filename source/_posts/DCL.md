---
title: 深入理解happen-before原则
date: 2018-07-06 12:39:48
tags: 多线程
categories: 多线程
---
happen-before 这个概念在最早接触并发编程的时候就知道，但是一直不得要领，不知道想表达的是什么。最近拜读了一些博客，还算有所领悟，于是写下来。
先把教科书上的内容贴一下，最后结合几个例子彻底搞懂它。
<!-- more -->
### <font color=red size=5>happens before</font>
概念：<code>如果存在hb(a,b)，那么操作a及a之前在内存上面所做的操作（如赋值操作等）都对操作b可见，即操作a影响了操作b</code>
Ps:hb(a,b) presents "a happens before b"
Ps:先行发生是一个逻辑上的概念，并非真实的执行的先后顺序

1）<font color=blue>程序次序规则</font>（Program Order Rule） <code>在一个线程内</code>，按照程序代码顺序，书写在前面的操作Happens-Before书写在后面的操作

线程中上一个动作及之前的所有写操作在该线程执行下一个动作时对该线程可见（也就是说，同一个线程中前面的所有写操作对后面的操作可见），在同一个线程内，即使发生了指令重排序，书写在前面的代码也是先行发生于书写在后面的代码的

2）<font color=blue>线程锁定规则</font>（Monitor Lock Rule） An unlock on a monitor happens-before every subsequent lock on that monitor.

如果线程1解锁了monitor a，接着线程2锁定了a，那么，线程1解锁a及其之前的（写）操作都对线程2可见（线程1和线程2可以是同一个线程）。

3）<font color=blue>volatile变量规则</font>（volatile Variable Rule） A write to a volatile field happens-before every subsequent read of that volatile. 

如果线程1写入了volatile变量v（这里和后续的“变量”都指的是对象的字段、类字段和数组元素），接着线程2读取了v，那么，线程1写入v及之前的写操作都对线程2可见（线程1和线程2可以是同一个线程）。

4）<font color=blue>线程启动规则</font>（Thread Start Rule） Thread对象的start()方法Happens-Before此线程的每一个动作。

5）<font color=blue>线程终止规则</font>（Thread Termination Rule） 线程中的所有操作都Happens-Before对此线程的终止检测。

6）<font color=blue>线程中断规则</font>（Thread Interruption Rule） 对线程interrupt()方法的调用Happens-Before被中断线程的代码检测到中断事件的发生，可以通过Thread.interrupt()方法检测到是否有中断发生。

7）<font color=blue>对象终结规则</font>（Finalizer Rule） 一个对象的初始化完成（构造函数执行结束）Happens-Before它的finalize()方法的开始。

8）<font color=blue>传递性</font>（Transitivity） 偏序关系的传递性：如果已知hb(a,b)和hb(b,c)，那么我们可以推导出hb(a,c)，即操作a Happens-Before 操作c。

很重要的特性，很多地方利用基本的先行发生保证了可见性

---
<code>时间上的先后顺序和先行发生原则没有太大的关系</code>
```java
int i = 1;
int j = 2;
```
同一个线程中显然有hb(i=1, j=2) 但是有可能发生指令重排序导致j=2先执行了，所以hb(i=1, j=2)并不代表在时间上也是i=1先于j=2发生

```java
private int value = 0;
public void setValue(int value) {
    this.value = value;
}
public int getValue() {
    return value;
}
```
假设线程1先执行了set方法，线程2才执行get方法，从时间上看set方法先于get发生，但是set和get并没有先行发生的关系，所以一个操作在时间上先于另一个操作发生也不能代表前者先行发生于后者

---
<code>看几个利用happen before规则的例子</code>
例子1：
以CopyOnWriteArrayList的set和get方法为例，我们看看是如何保证的hb(set, get)。
在这个类的javadoc上有这样一段声明
```java
* Memory consistency effects: As with other concurrent
* collections, actions in a thread prior to placing an object into a
* {@code CopyOnWriteArrayList}
* happen-before actions subsequent to the access or removal of that element from
* the {@code CopyOnWriteArrayList} in another thread.
```
简单地说就是对并发容器的写先行发生于后续对它的读和remove操作，那这个如何保证的呢？
```java
 public E set(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        E oldValue = get(elements, index);

        if (oldValue != element) {
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len);
            newElements[index] = element;
            setArray(newElements);
        } else {
            // Not quite a no-op; ensures volatile write semantics
            setArray(elements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
final void setArray(Object[] a) {
    array = a;
}
```
```java
public E get(int index) {
    return get(getArray(), index);
}
final Object[] getArray() {
    return array;
}
```
代码中的array是volatile类型的，而根据happen before原则，对array的写会先行发生于对它的读。
根据happen before法则有hb(set之前的代码,setArray),hb(getArray,get之后的代码)根据传递性有hb(set,get)

这里有一行注解很值得推敲，<code>// Not quite a no-op; ensures volatile write semantics</code>
我们看到在else中竟然也执行了一次set操作。其实这正是为了实现java doc中set happen before get而写的代码，不然当set一个相同的元素的时候就不能保证happen before了.而hb(set, get)这一特性可能会被其他地方使用以实现新的hb，所以即使相同元素的情况也要执行一次volatile的写。

例子2：
```java
/**
 * Created by luyu on 2019/1/28.
 */
public class Test0128 {

    //static volatile  int a = 1;

    static private boolean initialized = false;

    public static void main(String[] args) throws InterruptedException {

        new Thread(new Runnable() {
            public void run() {
                doWork();
            }
        }).start();

        Thread.sleep(1000);

        new Thread(new Runnable() {
            public void run() {
                init();
            }
        }).start();
    }

    public static void init() {
        initialized = true;
        //a = 2;
    }

    public static void doWork() {
        while (!initialized) {
            //int b = a;
        }
        System.out.println("end loop");
    }
}
```
这个例子是经典的说明volatile关键字用处的例子，这个例子运行之后会发现doWork一直处于死循环的状态。通常的做法是对initialized添加一个volatile修饰符，但我们这里换一种方式跳出死循环。以加深对hb原则的理解。

具体的方法就是将上面code中关于a的注释都取消，这个时候我们发现console会打印出"end loop"。
简单分析下原因，根据hb原则有hb(initialized = true, a = 2)，类似的hb(int b = a, while (!initialized))，而对volatile的写hb于对它的读，即 hb(a = 2, int b = a)，根据传递性最后推导出hb(initialized = true, while (!initialized))，所以这个时候可以跳出死循环。

例子3：
FutureTask的get与set（jdk6)，result并非volatile变量，怎么保证写对读的可见呢？
```java
void innerSet(V v) {
	for (;;) {
		int s = getState();
		if (s == RAN)
			return;
		if (s == CANCELLED) {
			// aggressively release to set runner to null,
			// in case we are racing with a cancel request
			// that will try to interrupt runner
			releaseShared(0);
			return;
		}
		if (compareAndSetState(s, RAN)) {
			result = v;
			releaseShared(0);
			done();
			return;
		}
	}
}
```
```java
V innerGet(long nanosTimeout) throws InterruptedException, ExecutionException, TimeoutException {
	if (!tryAcquireSharedNanos(0, nanosTimeout))
		throw new TimeoutException();
	if (getState() == CANCELLED)
		throw new CancellationException();
	if (exception != null)
		throw new ExecutionException(exception);
	return result;
}
```
因为 hb(result = v, releaseShared（对volatile的写)) 
hb(tryAcquireSharedNanos(对volatile的读), return result)
hb(releaseShared, tryAcquireSharedNanos)
所以 hb(result = v, return result)

### <font color=red size=5>线程不安全的DCL</font>
```java
public class Singleton {
    private static Singleton instance = null;
    private Singleton(){}
    public static Singleton getInstance(){
        if(instance == null){                     //①
            synchronized (Singleton.class) {      
                if(instance == null){             //②
                    instance = new Singleton();   //③
                }
            }
        }
        return instance;                          //④
    }
}
```
如果Thread1执行到③放弃了cpu时间，转到Therad2执行假设此时执行到①
这个时候就很有可能出现线程不安全的问题
先来分析下③这行代码都做了什么，大体可以分为三个步骤：
1）分配内存
2）初始化内存
3）将instance指向内存
当Thread1在执行过程中，CPU对这三个步骤进行了重排序，可能执行顺序为1)->3)->2)
因为从单个线程的角度来看，即使顺序变为1 3 2对最终的结果也没有任何影响，但是对cpu来讲效率or利用率变高了
但从多线程角度来看，如果cpu对③进行了重排序，Thread1在执行到3）时放弃了CPU时间，而Thread2执行到①，这个时候会返回一个false，直接将未完全初始化的instance返回给调用方（我理解这个时候已经不会再报NPE了，而是一个未经过初始化的内存可能会产生很多莫名其妙的问题），这显然是不行的
所以需要对这段代码进行改造，方法很简单只要在instance前面添加volatile关键字就ok
注意这里的volatile的作用是保证对其修饰的变量的构造不会发生重排序(对新对象的引用的写入操作不会与对象中各个域的写入操作重排序)
