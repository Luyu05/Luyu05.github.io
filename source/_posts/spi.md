---
title: spi机制与策略模式
date: 2018-09-14 19:11:43
tags: Java 进阶
categories: Java 进阶
---
Outline
一、啥是SPI？
二、啥是策略模式？
三、SPI和策略模式有啥关系？
<!--more-->
##  一、啥是SPI？
入职初期经常听大佬说SPI，也不知道是干啥的，最近终于有所领悟。
Java程序员应该对『面向接口』编程不陌生，我们要说的SPI和面向接口编程是紧密关联的。

简单地讲，如果一个接口由`调用方`来定义，而接口的实现由`提供方`来实现，这个就是SPI。而如果接口的定义和实现都由`提供方`来完成，就是我们常说的API。
这样说可能还是比较抽象，不如来看一个例子。
src.zip/rt-jar内定义了一个接口Driver，在这个例子中src包扮演的角色是调用方
```java
public interface Driver {
    Connection connect(String url, java.util.Properties info)
        throws SQLException;
        ...省略
}
```
既然src定义了接口，那么他肯定要用啊，具体细节忽略我们只关心下面这行代码
```java
public class DriverManager { 
    ...
     ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
     ....
      try{
            while(driversIterator.hasNext()) {
                driversIterator.next();
            }
        } catch(Throwable t) {
        // Do nothing
        }
}
```
小结：<br>
* 调用方提供了接口Driver
* 调用方面向Driver接口进行了编程 

下面看看提供方做了啥
首先接口提供方mysql-connector-java登场，我们重点关注两个地方
1）接口的具体实现
```java
public class Driver extends NonRegisteringDriver implements java.sql.Driver {
    public Driver() throws SQLException {
    }
    static {
        try {
            DriverManager.registerDriver(new Driver());
        } catch (SQLException var1) {
            throw new RuntimeException("Can't register driver!");
        }
    }
}
```
2）名为java.sql.Driver的配置文件
```xml
com.mysql.jdbc.Driver
```
小结：
* 实现了src中定义的接口
* 定义了一个不知道干啥的配置文件

我们再回到方法调用方src中（注意src是需要引入mysql-connector-java这个包的），有这样一行代码
<code>ServiceLoader.load(Driver.class);</code>
ServiceLoader读取mysql-connector-java包中的那个配置文件中定义的实现类的名字，通过反射获取对应类实例~
这样我们的接口就被指定具体的实现了~接下来我们来看策略模式
##  二、啥是策略模式？
在阎宏博士的《JAVA与模式》一书中开头是这样描述策略（Strategy）模式的：
策略模式属于对象的行为模式。其用意是针对一组算法，将每一个算法封装到具有共同接口的独立的类中，从而使得它们可以相互替换。策略模式使得算法可以在不影响到客户端的情况下发生变化。

我们还是通过一个例子来看
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
在Test类的doCalculate方法中并不关心接口的具体实现，这就是一『面向接口』编程的例子，我们看到接口的具体实现在main方法也就是接口的调用处来指定实现，这个思想和IOC的思想又是类似的~
那么看我们的最后一个问题，SPI和策略模式有啥关系
##  三、SPI和策略模式有啥关系？
聪明的小伙伴应该已经看出端倪
SPI模式和策略模式都是面向接口编程的典范，而且接口的具体实现由调用方来指定。策略模式的例子如上，对于第一个SPI例子来说，假如我们在开发一个java项目，使用的数据库是oracle，那么我们引入oracle的jar包即可，显然该jar包中一定包含一个Driver的实现类以及执行实现类的配置文件，这个时候我们的java程序就可以『无痕』地使用oracle提供的Driver类了
这里再多说一句，所有的java项目都会依赖src包的~