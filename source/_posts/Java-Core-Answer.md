---
title: Java核心技术拓展
date: 2019-07-26 09:09:34
tags: Java 进阶
categories: Java 进阶
---
读极客时间《Java核心技术36讲》有感
<!-- more -->
# <font color=green>解释执行和编译执行？</font>
解释执行可以理解为一个中国人和外国人对话，解释器相当于翻译人员，解释执行就是中国人说一句，翻译人员就翻译一句，告诉外国人
而编译执行，翻译人员直接讲全文翻译成英文，再把翻译文件给外国人看

对于Java这种解释和编译共存的语言就类似于：

而日常对话中有很多常用语句，翻译人员已经翻译无数次了，她想到了巧妙的方式，索性把常用语句直接写到白板上，这样当下次中国人再想说常用语句时候直接举白板就ok了
上述例子，中国人：高级语言（Java） 外国人：计算机 翻译：JVM 解释器 中国话：字节码 外国话：机器码 
显然编译执行很快，但是需要生成中间文件（目标程序），不适合内存紧张的场景；而解释执行，无需中间文件（目标程序），解释一句执行一句

是不是可以理解为，对计算机来讲输入都是一样的『机器码』，而解释执行是在内存中生成机器码用后即删，而编译执行会持久化到本地反复使用

# <font color=green>Java异常体系</font>
## Error VS Exception
Error更为严重，一般是JVM报的错误，比如OOM，SOF等，一般无法恢复；Exception一般都是可以通过编码避免的，分为两个类型checked和unchecked的，换言之就是编译器异常和运行期异常，前者包括sleep、io等操作时候必须catch的异常InterruptException、IOException等；后者包括NPE、ArrayIndexOutOfBoundException

1）除非有特殊用途否则不要catch Error 或者 Throwable
2）catch到的异常一般不推荐使用printStackTrace,最好使用日志组件，将日志统一打印到一个地方
3）敏感信息不要打印到日志中

## ClassNotFoundException VS NoClassDefFoundError
ClassNotFoundException相对简单，它是一个编译期异常，当我们使用Class.forName时候就会要求我们检查该异常，举个例子
```java
class C {
    public static void main(String[] args) throws ClassNotFoundException {
        test();
    }
    public static void test() throws ClassNotFoundException {
        Class.forName("Foo");
    }
}
```
输出： java.lang.ClassNotFoundException: B

而后者可能模拟起来相对困难，看下面的例子
```java
public class MockNoClassDefFoundError {
    public static void main(String[] args) throws InterruptedException {
        new Thread(new Runnable() {
            public void run() {
                new B();
            }
        }).start();
        new B();
    }
}

class B {
    static {
      int a = 1/0;
    }
}
```
输出：java.lang.NoClassDefFoundError: Could not initialize class com.luyu.B

这里为啥要使用一个多线程呢？
因为如果不用多线程的话，主线程直接就会抛java.lang.ExceptionInInitializerError，而使用多线程其中一个线程用来触发B的clinit方法，此时类初始化失败，而另一个线程再去使用B的时候就报NoClassDefFoundError了

# <font color=green>Java中的引用</font>
我们知道Java中共有四种引用，强软弱虚。为啥需要这么多引用呢，如果只有可回收与不可回收两种状态，那么对于一些食之无味弃之可惜的对象就没办法表示了，所以引用不能非黑即白。
1）强引用，即使发生OOM也不会回收的对象
2）软引用，垃圾回收一般不会回收，但是发生OOM前会将软引用对象回收
3）弱引用，垃圾回收会收走  Ps：Guava Cache/ThreadLocal都是该类型，和软引用一样适用于内存敏感情况下的缓存情景
4）虚引用，在执行过finalize之后处于虚引用状态，唯一存在的目的就是在对象被回收时可以发送一个系统通知