---
title: Spring bean循环依赖
date: 2019-02-16 10:55:40
tags: Spring
categories: Spring
---
1.两种bean的循环依赖形式
2.通过源码分析Spring的处理方式
<!-- more -->
# 两种循环依赖
其实就是Spring中bean的两种注入方式，一种是构造器注入，一种是属性注入
## 构造器注入:
```java
public class A {
    public A(B b) {}
}
```
```java
public class B {
    public B(A b) {}
}
```
```xml
<bean id="testA" class="com.spring.A">
    <constructor-arg ref="testB"/>
</bean>

<bean id="testB" class="com.spring.B">
    <constructor-arg ref="testA"/>
</bean>
```
此时容器启动会报如下错误
```xml
org.springframework.beans.factory.BeanCurrentlyInCreationException: <br>
Error creating bean with name 'testA': Requested bean is currently in creation: <br>
Is there an unresolvable circular reference?
```

## 属性注入:
```java
public class C {
    D d;
    public void setD(D d) {
        this.d = d;
    }
}
```
```java
public class D {
    C c;
    public void setC(C c) {
        this.c = c;
    }
}
```
```xml
<bean id="testC" class="com.spring.C">
    <property name="d" ref="testD"/>
</bean>

<bean id="testD" class="com.spring.D">
    <property name="c" ref="testC"/>
</bean>
```
此时容器可以正常启动，不会报错

---
小结：
这里我们先抛出结论，Spring Bean的循环依赖分为构造器和属性两种，其中前者会报错而后者不会报错，下面我们通过源码具体看看Spring是怎么处理这两种情况的

---
# 构造器注入源码分析
因为之前已经分析过Spring bean构造的全过程了[Spring源码阅读(2)-IOC之bean的实例化](https://luyu05.github.io/2018/11/15/Spring0-5/). 
所以接下来不会逐步分析每个细节，只会对之前文章中未提及的一些点进行剖析。这里使用了idea的debug来查看调用栈
显然，容器会先实例化testA这个bean，而testA又依赖了testB，我们利用条件断点定位到获取testB的位置，观察下调用栈
<img src='001.jpg'>
从下向上看，大部分内容我们之前都已经分析过了，之前阅读ioc源码主要看的是一般情况，一些特殊情况和细节当时并未关注。所以之前对于bean的实例化只是看了默认的无参数构造函数的实例化，对于这种构造器注入的方式并未关注，那接下来就从autowireConstructor这里开始。
```java
public BeanWrapper autowireConstructor(
			final String beanName, final RootBeanDefinition mbd, Constructor<?>[] chosenCtors, final Object[] explicitArgs) {
    //...
    ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
    resolvedValues = new ConstructorArgumentValues();
    //在这个方法中对bean进行了解析，在本例中就是对testB进行了解析
    minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
}
private int resolveConstructorArguments(
			String beanName, RootBeanDefinition mbd, BeanWrapper bw,
			ConstructorArgumentValues cargs, ConstructorArgumentValues resolvedValues) {
    //...
    Object resolvedValue = valueResolver.resolveValueIfNecessary("constructor argument", valueHolder.getValue());      
    //...
}
public Object resolveValueIfNecessary(Object argName, Object value) {
    //...
    return resolveReference(argName, ref);
}
private Object resolveReference(Object argName, RuntimeBeanReference ref) {
    //终于抓到了，也就是在这里进行了对testB这个bean的获取
    Object bean = this.beanFactory.getBean(refName);
    this.beanFactory.registerDependentBean(refName, this.beanName);
    return bean;
}
```
好我们已经看到了在对testA实例化过程中，触发了对testB的实例化，同样地testB也会触发testA的实例化，那么Spring会如何处理呢？我们继续debug
<img src='002.jpg'>
仿佛是一个轮回我们看到果然又一次触发了testA的实例化，代码来到了doGetBean中的getSingleton方法中
```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    //...
    beforeSingletonCreation(beanName);
    //执行createBean(beanName, mbd, args);
    singletonObject = singletonFactory.getObject();
    afterSingletonCreation(beanName);
    //...
}
```
我们看看<code>beforeSingletonCreation(beanName)</code>这个方法
```java
protected void beforeSingletonCreation(String beanName) {
    if (!this.inCreationCheckExclusions.contains(beanName) &&
            !this.singletonsCurrentlyInCreation.add(beanName)) {
        throw new BeanCurrentlyInCreationException(beanName);
    }
}
```
这里主要看<code>this.singletonsCurrentlyInCreation.add(beanName)</code>。
我们知道上述代码是每个bean实例化前都要执行的，也就是说testA也执行过（从调用栈也能看到最开始的getSingleton方法），这意味着singletonsCurrentlyInCreation这个set中有testA。那么当testB通过构造器注入时触发testA的实例化过程后，testA会再次执行这段代码，而这个时候set中已经存有一份testA了，此时add操作会返回false，进而执行<code>throw new BeanCurrentlyInCreationException(beanName)</code>于是会将错误抛出。

显然在完成实例化后，<code>afterSingletonCreation(String beanName)</code>会将beanName从set中移除
```java
protected void afterSingletonCreation(String beanName) {
    if (!this.inCreationCheckExclusions.contains(beanName) &&
            !this.singletonsCurrentlyInCreation.remove(beanName)) {
        throw new IllegalStateException("Singleton '" + beanName + "' isn't currently in creation");
    }
}
```
至此，构造器形式的循环依赖已经分析完毕。

---
小结：
1)当testA在实例化前会先将自己的beanName存放到一个正在实例化的beanName的set中
2)testA实例化过程触发了testB的实例化
3)testB也将自己加入正在实例化的beanName的set中
4)testB实例化又触发了testA的实例化
5)而这个时候再将testA加入set中会返回false，意味着产生了循环依赖
6)于是抛出Exception

---
# 属性注入源码分析
既然是循环依赖，显然C实例化过程也会触发D的实例化我们看看这时候的调用栈
<img src='003.jpg'>
我们看到此时testC已经执行到了applyPropertyValues这里，而这里是执行属性注入的地方，这意味着testC已经完成了实例化+属性注入。这里还有一些没有展示出来的细节我们关注下
在doCreateBean方法中执行了<code>addSingletonFactory</code>
```java
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(singletonFactory, "Singleton factory must not be null");
    synchronized (this.singletonObjects) {
        if (!this.singletonObjects.containsKey(beanName)) {
            this.singletonFactories.put(beanName, singletonFactory);
            this.earlySingletonObjects.remove(beanName);
            this.registeredSingletons.add(beanName);
        }
    }
}
```
注意此时还未执行bean的属性注入，我们看到这里将bean存放到了singletonFactories中，此时的bean仅仅是一个空壳（其属性并未完成填充）
显然testD实例化过程也会触发testC的实例化，我们debug也看到了确实是这样
<img src='004.jpg'>
接着代码进入到了doGetBean中，首先会从容器中看看是否已经有了对应bean的缓存
```java
protected <T> T doGetBean(
			final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
			throws BeansException {
    //...
    //对于构造器循环依赖，执行到这里返回值都是null
    Object sharedInstance = getSingleton(beanName);
    //...
    //注意区分这两个getSingleton，前者单纯地从缓存中捞取数据
    //而后者主要目的是执行createBean操作
    if (mbd.isSingleton()) {
        sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
            public Object getObject() throws BeansException {
                    return createBean(beanName, mbd, args);
            }
        });
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
    }        
}
```
这个方法注意和构造器依赖中的方法做区分，二者都在doGetBean中被调用
```java
public Object getSingleton(String beanName) {
    return getSingleton(beanName, true);
}

protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject();
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
```
我们看到上面这个方法，首先从singletonObjects中查找，而singletonObjects存放的是已经彻底完成实例化的bean；接着从earlySingletonObjects中获取，如果再获取不到就去singletonFactories中获取，获取到后会将bean存放到earlySingletonObjects中并将其从singletonFactories中移除。

testC->testD->testC  当第二次触发testC时候执行getSingleton时候，就会在singletonFactories中获取到并未填充属性的testC的实例，从而完成testD的实例化与属性注入接着完成testC的注入。

---
小结：
属性注入的循环依赖的解决其实是基于Java的引用传递，bean完成实例化后会将自己的引用暴露出来以供其他bean注入使用

---

---
总结：
1）一个bean在<font color=red>实例化之前</font>会将自己存放到名为singletonsCurrentlyInCreation的set中表征自己正处于创建状态
2）如果set执行add返回false表征发生了构造器类型的循环依赖，此时会抛出错误
3）在实例化之后会将自己从singletonsCurrentlyInCreation中移除
4）一个bean在<font color=red>实例化后属性注入前</font>会将自己存放到名为singletonFactories的map中
5）当bean2触发bean1实例化时，会先去singletonObjects中获取bean1，获取不到会去earlySingletonObjects中获取bean1，再获取不到就会去singletonFactories中获取。如果在singletonFactories中获取到，就将bean1从singletonFactories中移除，并存放到earlySingletonObjects中

以上です