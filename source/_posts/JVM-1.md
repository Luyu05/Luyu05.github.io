---
title: JVM
date: 2018-08-20 19:56:31
tags: JVM
categories: JVM
---
Outline
0.最近在看《深入理解JVM》对全书做了一个思维导图，也方便以后的回顾（提纲挈领嘛），看不清楚的小伙伴可以download到本地，图片还是很清晰的
1.读书过程中的一些想法与体会也顺带记录了下，欢迎大家批评指正
<!-- more -->
### <font color=red size=5>一个*.java到真正执行都经历了什么？</font>
1. <font color=blue>首先*.java文件通过javac编译器生成字节码文件*.class</font>
Ps：Java号称一次编写，到处运行，这里的*.class是和平台无关的，这个时候需要不同平台的JVM对同一份*.class文件进行翻译了
在生成了class文件后，虽然可以根据class文件的内容去阅读具体的细节，但是比较『反人类』，这个时候可以使用javap去『去伪存真』，比如一些语法糖，让我们可以阅读到离机器最近的code，有时候方便排查一些『莫名其妙』的问题（乱用成语的习惯。。。）
javap -verbose xx.class 除了代码外还可以查看常量池的信息
2. <font color=blue>加载</font>
理论上来讲类的加载是讲究时机的，具体可以参考导图中初始化的触发时机（初始化发生前一定会先执行加载操作）。
当类被加载的时候，会做三件事情
1）通过类的全限定名获取此类的二进制流
2）将这个字节流所代表的静态存储结构转换为方法区运行时的数据结构，包括class中的常量池保存到JVM内存的运行时常量池、Class&Method&Field的信息保存到方法区
Ps：
简单的说类的全部信息都保存在方法区中，包括类的版本、字段、方法等的描述信息。
再看看一些细节的东西
将class文件中保存的一些内容存储到真正的运行时的内存中，比如class文件中的常量池，包括字面量和符号引用，要存储到JVM运行时的常量池中；将类的版本信息、方法、字段等的描述信息，存储到方法区中。
3）在堆中生成这个类的class对象，作为访问方法区中该类信息的入口（通过反射去拿类的信息）
3. <font color=blue>验证</font>
这步没啥好说的，就是对class文件进行校验，防止class会损害虚拟机
4. <font color=blue>准备</font>
两件事，第一，为类变量分配内存并赋默认值；第二，为final修饰的变量进行赋值操作（非默认值）
5. <font color=blue>解析</font>
严格意义上讲，解析可以在初始化之前也可以在其之后。
这步骤的工作就是将常量池中的符号引用转为直接引用比如上面的object（符号引用）转为堆上的直接引用
对于初始化前的方法解析主要针对一些运行期间不可变的方法比如final、private、static等
而对于初始化后的方法解析主要针对一些运行期绑定的方法（指的是真正运行时候，而不是说初始化之后立刻去解析）
还是举个例子吧
```java
private void fun(Father f) {
    f.fun1();
}
```
  假设调用上面这个方法，而入参为Father的子类Son且Son也覆写了fun1方法，那么程序最终执行的父类的方法还是子类的方法是不得而知的，只有在程序真正运行时候才能确定，称之为运行时绑定
6. <font color=blue>初始化</font>
执行clinit方法，该方法包含类变量的赋值操作（非final）和static语句块中的操作
7. <font color=blue>至此该类已经可以正常使用了</font>

### <font color=red size=5>内存回收实战</font>
这里是笔者公司内部的一个case，已经隐去了具体的涉密信息，展示一个框架给大家看，但是也能说明问题了。
```java
for(int id : idList) {
    ExecutorService threadPool = Executors.newSingleThreadExucutor();
    threadPool.execute(new Runnable(){
        @Override
        public void run() {
            doBusiness(id);
        }
    });
}
```
  这段代码会引起频繁的YGC，和OOM（具体OOM原因视JVM配置参数而定）
  显然是因为在for循环了创建了太多线程池，但是又没有执行shutdown操作，导致有很多活跃的核心线程，最终导致了栈的OOM（也可能是堆OOM视配置而定），而YGC是因为线程池内部的一些对象没有被回收，看过线程池内部原理的同学应该知道Worker之类的对象

### <font color=red size=5>内存分配的策略</font>
1.对象优先分配到Eden区，Eden区满了会触发一次Young GC(新生代对象朝生夕死 Young GC非常频繁并且回收速度较快 Old GC速度一般是前者的10倍)
2.大对象（阈值可配）直接进入老年代，避免在Eden去和Survivor区间进行大量的copy
3.长期存活的对象进入老年代，Survivor区每熬过一次Young GC年龄就+1，增加到一定数值，就会晋升到老年带中
4.发生Young GC时，虚拟机如果发现之前每次晋升到老年代的平均大小大于老年代的剩余空间，就执行一次Full GC；如果小于查看是否允许担保失败，如果允许那么久执行Young GC；反之执行Full GC
5.担保机制：让Survivor中无法容纳的对象直接晋升老年带

总结：
Young GC经常发生 而且速度很快 触发条件是Eden区满
Full GC发生条件：
1.System.gc() 此方法建议执行full gc 但是未必会执行 一般禁用
2.发生Young GC时如果发现当前老年代剩余空间小于之前每次晋升对象的平均大小那么执行Full GC
3.发生Young GC时如果空间分配担保机制不允许失败那么执行Full GC

### JAVA内存泄露
定义：对象已经没有程序使用了但是gc无法移除，就是内存泄露
eg
1.顺序构造链表时候，可能会使用哨兵节点，而最后返回的是哨兵.next，此时哨兵就是内存泄露
2.LinkedList移除节点时候会将双向链表都断开也是为了释放内存
3.数组实现stack出栈操作 return element[--size] 此时element[size]内存泄露
4.循环中创建线程池 忘记调用shutdown
5.长生命周期对象（v）持有短生命周期的对象（o）
```java
Vector v = new Vector(10);
for (int i = 0; i < 100; i++) {
    Object o = new Object();
    v.add(o);
    o = null;
}
```
o = null本意是释放对象，但是由于v并未释放所以导致内存泄露

### 不同内存区域的OOM
1.栈
无限递归
2.堆
死循环向容器内塞大对象
3.方法区
3.1 常量池
String.valueOf(i++).intern 死循环执行这行代码
3.2 方法区
运行时产生大量的类去填充方法区   Ps： cglib or reflect

### <font color=red size=5>结合例子来看.class文件的常量池</font>
```java
public class TestJavap {
    private int n = 1;
    private String mstring = "123";
    public int c;
    public void f() {}
    private final int a = 4;

    public static void main(String[] args) {
        String d = "string";
        System.out.println(d);
    }
}
```
javap反编译后结果如下
```java
Classfile /Users/luyu/git/tech-lab/target/classes/com/Test/TestJavap.class
  Last modified 2018-9-17; size 799 bytes
  MD5 checksum bb7043f39135f0e9f9cfc397f6d96a65
  Compiled from "TestJavap.java"
public class com.Test.TestJavap
  minor version: 0
  major version: 49
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #10.#34        // java/lang/Object."<init>":()V
   #2 = Fieldref           #9.#35         // com/Test/TestJavap.n:I
   #3 = String             #36            // 123
   #4 = Fieldref           #9.#37         // com/Test/TestJavap.mstring:Ljava/lang/String;
   #5 = Fieldref           #9.#38         // com/Test/TestJavap.a:I
   #6 = String             #39            // string
   #7 = Fieldref           #40.#41        // java/lang/System.out:Ljava/io/PrintStream;
   #8 = Methodref          #42.#43        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #9 = Class              #44            // com/Test/TestJavap
  #10 = Class              #45            // java/lang/Object
  #11 = Utf8               n
  #12 = Utf8               I
  #13 = Utf8               mstring
  #14 = Utf8               Ljava/lang/String;
  #15 = Utf8               c
  #16 = Utf8               a
  #17 = Utf8               ConstantValue
  #18 = Integer            4
  #19 = Utf8               <init>
  #20 = Utf8               ()V
  #21 = Utf8               Code
  #22 = Utf8               LineNumberTable
  #23 = Utf8               LocalVariableTable
  #24 = Utf8               this
  #25 = Utf8               Lcom/Test/TestJavap;
  #26 = Utf8               f
  #27 = Utf8               main
  #28 = Utf8               ([Ljava/lang/String;)V
  #29 = Utf8               args
  #30 = Utf8               [Ljava/lang/String;
  #31 = Utf8               d
  #32 = Utf8               SourceFile
  #33 = Utf8               TestJavap.java
  #34 = NameAndType        #19:#20        // "<init>":()V
  #35 = NameAndType        #11:#12        // n:I
  #36 = Utf8               123
  #37 = NameAndType        #13:#14        // mstring:Ljava/lang/String;
  #38 = NameAndType        #16:#12        // a:I
  #39 = Utf8               string
  #40 = Class              #46            // java/lang/System
  #41 = NameAndType        #47:#48        // out:Ljava/io/PrintStream;
  #42 = Class              #49            // java/io/PrintStream
  #43 = NameAndType        #50:#51        // println:(Ljava/lang/String;)V
  #44 = Utf8               com/Test/TestJavap
  #45 = Utf8               java/lang/Object
  #46 = Utf8               java/lang/System
  #47 = Utf8               out
  #48 = Utf8               Ljava/io/PrintStream;
  #49 = Utf8               java/io/PrintStream
  #50 = Utf8               println
  #51 = Utf8               (Ljava/lang/String;)V
{
  public int c;
    descriptor: I
    flags: ACC_PUBLIC

  public com.Test.TestJavap();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: iconst_1
         6: putfield      #2                  // Field n:I
         9: aload_0
        10: ldc           #3                  // String 123
        12: putfield      #4                  // Field mstring:Ljava/lang/String;
        15: aload_0
        16: iconst_4
        17: putfield      #5                  // Field a:I
        20: return
      LineNumberTable:
        line 6: 0
        line 7: 4
        line 8: 9
        line 11: 15
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      21     0  this   Lcom/Test/TestJavap;

  public void f();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=0, locals=1, args_size=1
         0: return
      LineNumberTable:
        line 10: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       1     0  this   Lcom/Test/TestJavap;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=1
         0: ldc           #6                  // String string
         2: astore_1
         3: getstatic     #7                  // Field java/lang/System.out:Ljava/io/PrintStream;
         6: aload_1
         7: invokevirtual #8                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        10: return
      LineNumberTable:
        line 14: 0
        line 15: 3
        line 16: 10
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      11     0  args   [Ljava/lang/String;
            3       8     1     d   Ljava/lang/String;
}
SourceFile: "TestJavap.java"
```

1）final修饰的a，它的值4也在常量池，4就是所谓的字面量
2）字符串常量"123"也在常量池，"123"也是所谓的字面量
3）基本上类中出现的符号都会在常量池中，比如n、c、a、mstring、d也都在常量池中
4）除此之外还有域的类型，方法的返回值类型、入参类型、入参名字
5）方法内部的"string"并不在常量池中
6）我们可以理解为类（父类信息）、域（修饰符等）、方法（修饰符等）的实际信息都存放在方法区中，而那些符号和字面量存在常量池。
<code>终于搞懂了字面量和符号引用的意思，实践是检验整理的唯一标准，古人诚不欺我</code>

### <font color=red size=5>再来宏观看下jvm运行时的数据区</font>
<img src='001.png' />
重点关注方法区（jdk8改为metaspace），包括
1）运行时常量池(Runtime Constant Pool)，包括class文件中的常量池和运行时的string.intern()
2）类信息(Class & Field & Method data)
3）即时编译器编译后的代码(Code)等等

### <font color=red size=5>什么是动态连接</font>
如何理解动态连接？我们知道Class文件的常量池中存有大量的符号引用，在加载过程中会被原样的拷贝到内存里先放着，到真正使用的时候就会被解析为直接引用 (直接引用包含：直接指向目标的指针、相对偏移量、能间接定位到目标的句柄等)。有些符号引用会在类的加载阶段或者第一次使用的时候转化为直接引用，这种转化称为静态解析(private/final)，而有的将在运行期间转化为直接引用，这部分称为动态连接(多态)。

### <font color=red size=5>常用的垃圾收集器</font>
|适用周期 | 名称 | 算法 | 特征描述
|- | - | - | ---
|新生代 | <font color=red>Serial</font> | 复制  | 1.垃圾收集时STW<br><br>2.只有一个活跃的线程去进行GC<br><br>3.可以和老年代的CMS一起使用
|新生代 | <font color=red>ParNew</font>  | 复制  | 1.Serial的多线程版本<br><br>2.垃圾收集时STW<br><br>3.多个活跃的线程进行GC<br><br>4.可以和老年代的CMS一起使用
|新生代 | <font color=red>Parallel Scavenge</font>  | 复制  | 1.多个活跃的线程进行GC<br><br>2.吞吐量=运行用户代码时间/（用户代码时间+GC时间），关注点比较奇特是吞吐量，而不是用户线程的停顿时间<br><br>3.停顿时间短用户体验较好，而吞吐量高可以最高效率利用CPU时间，适合在后台运算而不许太多交互的任务<br><br>4.STW
|老年代 | <font color=red>Serial Old</font> |  标记-整理  | 1.Serial老年代版本<br><br>2.只有一个活跃的线程去进行GC<br><br>3.垃圾收集时STW
|老年代 | <font color=red>Parallel Old</font> |  标记-整理  | 1.Parallel Scavenge老年代版本<br><br>2.多个活跃的线程进行GC<br><br>3.与Parallel Scavenge一起形成了吞吐量优先的闭环<br><br>4.STW
|老年代 | <font color=red>CMS(Concurrent Mark Sweep)</font> |  标记-清除  | 1.以获取最短回收停顿时间为目标<br><br>2.分为四个步骤，初始标记（STW耗时较短）、并发标记（并发标记和用户线程并行，耗时较长）、重新标记（STW耗时较短）、并发清除（并发清除和用户线程并行，耗时较长）<br><br>3.耗时较长的过程用户线程都能工作，所以用户体验较好<br><br>4.缺点：CPU敏感，可能很多CPU用来处理GC；无法处理浮动垃圾（清理过程，工作线程产生的垃圾）；标记清除算法的缺点----空间碎片
|老年代 | <font color=red>G1</font> |  标记-整理  | 1.非常精确地控制停顿，指定xx毫秒停顿xx毫秒<br><br>2.极力避免全区域垃圾收集，将堆划分为多个独立区域，优先回收垃圾最多的区域（G1=Garbage First）

### <font color=red size=5>GC触发条件</font>
Minor GC：当Eden区满的时候，就会执行Minor GC
Full GC：
1）大对象直接进入老年代，如果此时老年代空间不够那么执行Full GC
2）调用System.gc时，系统建议执行Full GC，但是不必然执行
3）执行了jmap -histo:live pid命令 //这个会立即触发fullgc
4）在执行minor gc的时候进行的一系列检查 
如果开启空间担保机制，则JVM会检查老年代中最大连续可用空间是否大于了历次晋升到老年代中的平均大小，如果小于则执行改为执行Full GC。
5）由Eden区、From Space区向To Space区复制时，对象大小大于To Space可用内存，则把该对象转存到老年代（空间担保机制），且老年代的可用内存小于该对象大小

### <font color=red size=5>Eden/From/To</font>
<code>From和To是动态的</code>
在GC开始的时候，对象只会存在于Eden区和名为“From”的Survivor区，Survivor区“To”是空的。紧接着进行GC，Eden区中所有存活的对象都会被复制到“To”，而在“From”区中，仍存活的对象会根据他们的年龄值来决定去向。年龄达到一定值(年龄阈值，可以通过-XX:MaxTenuringThreshold来设置)的对象会被移动到年老代中，没有达到阈值的对象会被复制到“To”区域。经过这次GC后，Eden区和From区已经被清空。这个时候，“From”和“To”会交换他们的角色，也就是新的“To”就是上次GC前的“From”，新的“From”就是上次GC前的“To”。不管怎样，都会保证名为To的Survivor区域是空的。Minor GC会一直重复这样的过程，直到“To”区被填满，“To”区被填满之后，会将所有对象移动到年老代中。

### <font color=red size=5>代码的执行顺序</font>
```java
package com.Test;

/**
 * Created by luyu on 2018/9/26.
 */
public class OrderTest {

    static final int c = init2();

    private static int init2() {
        System.out.println("father static final variable");
        return 0;
    }

    int d = init3();

    private int init3(){
        System.out.println("father normal variable");
        return 0;
    }

    static int a = init();

    private static int init() {
        System.out.println("father static variable");
        return 1;
    }

    static {
        System.out.println("father static {}");
    }

    final int b = init1();

    private static int init1() {
        System.out.println("father final variable");
        return 0;
    }

    public OrderTest() {
        System.out.println("father constructor");
    }
}

class OrderTestSon extends OrderTest {

    static final int c = init2();

    private static int init2() {
        System.out.println("son staic final variable");
        return 0;
    }

    static {
        System.out.println("son static {}");
    }

    static int a = init();

    private static int init() {
        System.out.println("son static variable");
        return 1;
    }

    final int b = init1();

    private static int init1() {
        System.out.println("son final variable");
        return 0;
    }

    int d = init3();

    private int init3(){
        System.out.println("son normal variable");
        return 0;
    }

    public OrderTestSon() {
        System.out.println("son constructor");
    }

    public static void main(String[] args) {
        new OrderTestSon();
    }
}

```
输出结果：
father static final variable
father static variable
father static {}
son staic final variable
son static {}
son static variable
father normal variable
father final variable
father constructor
son final variable
son normal variable
son constructor

代码执行顺序为：父类clinit方法（包括类变量的赋值+静态代码块）、子类clinit方法、父类普通成员初始化（普通代码块）、父类构造函数、子类普通成员初始化（普通代码块）、子类构造函数
注意点：final static修饰的变量会优先赋值，前提是"="右侧是常量，如果通过方法赋值那么就是普通的clinit


### <font color=red size=5>《深入理解JVM》思维导图</font>
<img src='JVM-2.png'>