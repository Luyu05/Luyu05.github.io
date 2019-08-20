---
title: JAVA中的设计模式
date: 2018-06-24 16:39:03
tags: Java 进阶
categories: Java 进阶
---
记录日常开发中常用的一些设计模式以及自己的一些体会
<!-- more -->
### FACADE模式
门面模式是对象的结构模式，外部与一个子系统的通信必须通过一个统一的门面对象进行。门面模式提供一个高层次的接口，使得子系统更易于使用。-------《JAVA与模式》
优点：
* 屏蔽细节
* 客户端调用优雅
* 保护某些不希望被客户端访问的public方法

```java
Before:
//小猫管理者会喂猫，又会写情书，别的管理者都需要请求他来写情书，所以该方法开放
class CatKeeper {
	public void writeLoveLetter(){}
	public void feedCat(){}
}
class DogKeeper {
	public void feedDog(){}
}
//此时动物园需要和每个动物管理员打交道，而且可能会调用写情书的方法这显然是不合理的
class Zoo {
	CatKeeper ck = new CatKeeper();
	DogKeeper dk = new DogKeeper();
	public void feedAnimals(){
		ck.feedCat();
		dk.feedDog();
	}
}
After:
class ZooKeeper {
	CatKeeper ck = new CatKeeper();
	DogKeeper dk = new DogKeeper();
	public void feedAnimals(){
		ck.feedCat();
		dk.feedDog();
	}
}
//此时动物园可以通过统一的zooKeeper饲养动物，同时也不会调用写情书的方法，而小猫管理员可以继续帮别的管理员写情书了
class Zoo {
	ZooKeeper zk = new ZooKeeper();
	zk.feedAnimals();
}
```
Ps: HttpServlet中的方法doGet/doPost入参为HttpServletRequest和HttpServletResponse，这两个参数是从tomcat中传递而来，实际上是对原始的request和response进行了一层封装，也就是facade模式，之后才传递到servlet中，，目的就是屏蔽request和response中某些不希望被外部访问的方法
### 模板方法模式
常用于解决某些通用的部分中嵌有特异的部分
通用的抽象，特异的定义抽象方法由各个子类实现
```java
Before:
class Luyu{
	public void visitZoo() {
		buyTicket();
		seeLion();
		seeHawk();
		goHome();
	}	
}
class HH{
	public void visitZoo() {
		buyTicket();
		seeCat();
		seeDolphin();
		goHome();
	}	
}
After:
abstract class Visitor{
	abstract void seeMyFavorateAnimals();
	public void visitZoo() {
		buyTicket();
		seeMyFavorateAnimals();
		goHome();
	}	
}
class Luyu extends Visitor {
	@Override
	public void seeMyFavorateAnimals() {
		seeLion();
		seeHawk();
	}
}
class HH extends Visitor {
	@Override
	public void seeMyFavorateAnimals() {
		seeCat();
		seeDolphin();
	}
}
```
Ps 在spring中，对于抽象类的bean定义需要设置其abstract="true"
<bean id="visitor" class="com.luyu.Visitor" abstract="true"/>
### 单例模式
参考文章：https://www.cnblogs.com/java-my-life/archive/2012/03/31/2425631.html
**饿汉模式**
```java
public class EagerSingleton {
	private static EagerSingleton instance = new EagerSingleton();
	private EagerSingleton(){}
	public static EagerSingleton getInstance() {
		return instance;
	}
}
```
EagerSingleton类被加载的时候，会初始化静态变量，由JVM保证线程安全
优点：简单无须同步
缺点：不需要的时候也实例化，浪费空间
Ps:说点题外话，这里说是饿汉其实也没有那么饿，其实真正new实例是在触发初始化的时候，而不是一上来就new
类的生命周期：加载、验证、准备、解析、初始化、使用和卸载
其中在准备阶段为类变量分配内存并设置类变量初始值，而初始化阶段才会进行类变量的赋值操作，常见的触发初始化的条件是new、get/put static fields
当多个线程触发该类初始化时，由JVM保证类初始化的线程安全
**懒汉模式**
```java
public class LazySingleton {
    private static LazySingleton instance = null;
    private LazySingleton(){}
    public static synchronized LazySingleton getInstance(){
        if(instance == null){
            instance = new LazySingleton();
        }
        return instance;
    }
}
```
优点：在需要的时候才去实例化且线程安全
缺点：每次调用getInstance方法获取实例时，都需要同步性能较差
**双重检查加锁**
```java
public class Singleton {
    private volatile static Singleton instance = null;
    private Singleton(){}
    public static Singleton getInstance(){
        if(instance == null){
            synchronized (Singleton.class) {
                if(instance == null){
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```
很经典的一个关于volatile关键字使用的例子，如果去掉该关键字，那么getInstance方法变为线程不安全
很可能Thread1在执行instance = new Singleton()的时候，发生了指令重排，真正的实例化尚未完成，但是instance已经不为null
这时候TThread2在第一重检查的时候通过了，直接返回了一个中间态的实例给到调用方，这是我们不能接受的。
优点：不需要每次都同步且线程安全
缺点：使用了volatile关键字使得虚拟机的一些优化手段被禁用，可能会对性能有影响
**静态内部类模式**
```java
public class Singleton {
    private Singleton(){}
    /**
     *    类级的内部类，也就是静态的成员式内部类，该内部类的实例与外部类的实例
     *    没有绑定关系，而且只有被调用到时才会装载，从而实现了延迟加载。
     */
    private static class SingletonHolder{
        /**
         * 静态初始化器，由JVM来保证线程安全
         */
        private static Singleton instance = new Singleton();
    }
    public static Singleton getInstance(){
        return SingletonHolder.instance;
    }
}
```
优点：懒汉模式且由JVM保证保证线程安全
缺点：引入了静态内部类，代码稍显复杂
1）上述方法都存在一个问题，如果客户端通过反射使得私有构造函数变为可访问，之后实例化，这个时候会产生多个"单例"
此时一个方法是在构造函数中，对于创建的第二个实例抛出异常（AtomInteger可以保证）
2）单例在序列化的时候相对复杂，首先实现Serializable接口，每个域都要置为transient，还需要提供一个readResolve方法 
**枚举模式**
```java
Enum Singleton {
	INSTANCE;
	//类的方法
	public void myMethod(){}
}
```
简洁优雅，有效抵挡反射、序列化攻击
### 代理模式-静态VS动态
优雅地修改某个类中的既有方法且不修改原有代码
**静态代理**
PART1
```java
interface ISubject {
    void request();
}
```
```java
class SubjectImpl implements ISubject {
    @Override
    public void request(){
        System.out.print("origin request");
    }
}
```
```java
class SubjectProxy implements ISubject {
    private ISubject subject;
    public SubjectProxy(ISubject s){
        this.subject = s;
    }
     @Override 
    public void request(){ 
        System.out.print("before origin request"); 
        System.out.print("origin request"); 
        System.out.print("after origin request"); 
    }
}
```
上面就是一个标准的代理模式，但是这样可能会有一些问题，如果系统内还有其他接口也有request方法，而我们也想对其代理相同的逻辑，这时候需要再去创建一个代理类
换句话说有多少个接口，就需要创建多少个代理类，那这样我们的系统就显得过于臃肿了，这时候我们引出动态代理
**动态代理模式**
PART1 & PART2同上，被代理的对象一定要实现接口
```java
class RequestInvocationHandler implements InvocationHandler {
    private Object target;
    public RequestInvocationHandler(Object target) {
        this.target = target;
    }
    public Object invoke(Object proxy,Method method,Object[] args) thorws Throwable {
        //只对request方法进行代理
        if(method.getName.equals("request")){
            System.out.print("before origin request");   
            method.invoke(obj,args); 
            System.out.print("after origin request");
        } 
    }
}
```
这样上面这个handler就具有一定的通用性了，可以供所有接口使用。
```java
ISubject subject = new SubjectImpl();
RequestInvocationHandler handler = new RequestInvocationHandler(subject);
ISubject proxy = (ISubject)Proxy.newProxyInstance(subject.getClass().getClassLoader(),subject.getClass().getInterfaces(),handler);
proxy.request();
```
使用动态代理最大的问题就是只能对实现了接口的类进行代理，使用广度有些受限
**CGLIB**
动态字节码：对目标对象进行继承扩展，为其生成相应的子类，而子类可以通过覆写来扩展父类的行为，只要将横切逻辑的实现放到子类中，然后让系统使用扩展后的目标对象的子类，就可以达到与代理模式相同的效果。
```java
class RequestCallback implements MethodInterceptor {
    public Obeject intercept(Obeject object, Method method, Object[] args, MethodProxy proxy) {
        //只对request方法进行代理 
        if(method.getName.equals("request")){ 
            System.out.print("before origin request");    
            method.invoke(obj,args);  
            System.out.print("after origin request"); 
        } 
        return null;
    }
}
```
使用方式
```java
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(SubjectImpl.class);
enhancer.setCallback(new RequestCallback());
SubjectImpl proxy = (SubjectImpl)enhancer.create();
proxy.request();
```
使用CGLIB对类进行扩展的唯一限制就是无法对final方法进行覆写
### 责任链模式-管道模型
多条流水线，相似度较高，但仍有部分不同的模块或者流水线的顺序不同
before:
```java
class processLine0 {
	public void process(){
		module0();
		module1();
		module3();
	}
}
class processLine1 {
	public void process(){
		module1();
		module5();
		module3();
		module0();
	}
}
```
after:
```java
//框架层面代码
//阀接口
public interface Valve {
  public void invoke(UniParam uni);
}
//管道接口
public interface Pipeline {
	public void addValve(Valve valve);
	public void doInvoke(UniParam uni);
}
public class MyPipeline implements PipeLine {
	private List<Valve> valves = Lists.newArrayList();
	private Valve getFirst(){
		Collections.isEmpty(valves) ? throw Exception : return valves.get(0);
	}
	@Override
	public void addValve(Valve valve){
		valves.add(valve);
	}
	@Override
	public void doInvoke(UniParam uni){
		int cur = 0;
		while(cur <= valves.size()){
			valves.get(cur++).invoke(uni);
		}
	}
}
//业务层面 自己实现
//X=0,1,3,5
public class ModuleX implements Valve {
	@Override
	public void invoke(UniParam uni){
		//module自己的逻辑
		doMyThing(uni);
	}
}
class processLine0 {
	public void process(){
		PipeLine pl = new PipeLine();
		Uniparam uni = new Uniparam();
		pl.add(new Module0());
		pl.add(new Module1());
		pl.add(new Module3());
		pl.doInvoke(uni);
	}
}
class processLine1 {
	public void process(){
		PipeLine pl = new PipeLine();
		pl.add(new Module1());
		pl.add(new Module5());
		pl.add(new Module3());
		pl.add(new Module0());
		pl.doInvoke(uni);
	}
}
```
是不是灵活了许多
缺点是valve的invoke方法必须要是同一个模板（入参返回值相同）

### 策略模式
```java
interface Calculator {
	int doExecute(int a, int b);
}
class AddCalculator implements Calculator{
	public int doExecute(int a, int b){
		return a+b;
	}
}
class MinusCalculator implements Calculator{
	public int doExecute(int a, int b){
		return a-b;
	}
}
public class Test {
	Calculator cal;
	public void setCal(Calculator cal){
		this.cal = cal;
	}
	public void doCalculate(int a, int b){
		cal.doExecute(a, b);
	}
	public static void main(String[] args) {
		Test t = new Test();
		t.setCal(new AddCalculator());
		t.doCalculator(1,2);
		t.setCal(new MinusCalculator());
		t.doCalculator(2,1);
	}
}
```
Spring中的bean的实例化过程采用的就是策略模式（reflect or CGLIB），默认是CGLIB