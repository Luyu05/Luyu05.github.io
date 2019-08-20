---
title: Spring-AOP专题
date: 2018-07-28 11:22:07
tags: Spring
categories: Spring
---
Spring AOP专题
<code>AOP解决程序开发里事务性，和核心业务无关的问题，但这些问题对于业务场景的实现是很有必要的。</code>
<!-- more -->
传统的编程方式是垂直化的，不同的模块相同的逻辑虽然可以抽象出来，但是还是会对原有的代码有一定的侵入，而且这部分相同的逻辑（下文称为统计代码）大多数情况和业务代码无关，那么能不能有一种方法可以<font color=blue>对业务代码0侵入，而且又能实现应有的功能？</font>
<img src='aop.png'>
### <font color=red size=5>代理模式</font>
<code>没有什么是通过抽象不能解决的，如果有那就抽象两次--奥雷连诺-刘能</code>
在业务代码上抽象出一层代理层，持有原始类的引用，在调用处前后加入相应的统计代码。
优点：
1.业务代码无感知
2.对同一个接口的实现类来讲只需要一个代理类
缺点：
1.在统计代码相同的情况下，不同接口需要开发不同的代理类
2.而且统计代码和业务代码仍然存在一定程度的耦合，比如现在想把某个方法的统计功能去掉or某个没有统计功能的代码加上统计功能，那么仍然需要修改Proxy类

既然有缺点那就一条一条解决，首先解决代理类膨胀的问题，这个很简单通过动态代理就能解决 Ps这部分自己不常用，所以写一下代码

```java
public class MonitorHandler implements InvocationHandler {
    private Object obj;
    public MonitorHandler(Object obj){
        this.obj = obj;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) {
        if(needLog()){
            logbefore();
            Object result = method.invoke(obj,args);
            logafter();
            return result;
        }
        return method.invoke(obj,args);
    }
}
public static void main(String[] args) {
    InterfaceA interfaceA = new InterfaceAImpl();
    InterfaceB interfaceB = new InterfaceBImpl();
    InterfaceA proxyA = (InterfaceA) Proxy.newProxyInstance(MonitorHandler.class.getClassLoader(), new Class<?>[]{InterfaceA.class},new MonitorHandler(interfaceA));
    InterfaceA proxyB = (InterfaceB) Proxy.newProxyInstance(MonitorHandler.class.getClassLoader(), new Class<?>[]{InterfaceB.class},new MonitorHandler(interfaceB));
    proxyA.doWork();
    proxyB.doWork();
}
```
对了这里还要提一句如果被代理的类没实现接口此时需要使用CGLIB
好，这样看已经解决了，代理类膨胀的问题；那么统计代码和业务代码耦合的问题如何解决？这个时候就要引出aop了
Ps Spring AOP的底层实现就是通过JDK动态代理和CGLIB动态代理技术实现的
### <font color=red size=5>AOP</font>
我觉得理解下面三个术语就ok了
1. advisor
advisor = pointcut(定位到具体方法) + advice(方法的具体织入位置和逻辑)
2. 切点pointcut
定义统计逻辑的切入位置，目前只支持方法级别的切入
<code>切点只定位到某个方法上</code>
3. advice
advice = 逻辑 + 方位
统计代码写到这里，由不同类型表示目标切入方法的具体方位，包括前置、后置、异常、最终、环绕
BeforeAdvice/AfterAdvice/ThrowAdvice

简单例子《Spring 3.x》，这个例子已经是aspectJ了~
```java
public class NaiveWaiter implements Waiter{
    public void greetTo(String name){
        System.out.println("greet to"+name);
    }
    public void serverTo(String name){
        System.out.println("serve to"+name);
    }
}

@Aspect
public class PreGreetingAspect{
    @Before("execution(* greetTo(...))")
    public void beforeGreeting(){
        System.out.println("How are you");
    }
}

public static void main(String[] args) {
    AspectJProxyFactory fac = new AspectJProxyFactory();
    fac.setTarget(new NaiveWaiter);
    fac.addAspect(PreGreetingAspect.class);
    Waiter proxy = fac.getProxy();
    proxy.greetTo("Luyu");
    proxy.serverTo("Luyu");
}
or
<aop:aspectj-autoproxy />
<bean id="waiter" class="com.luyu.NaiveWaiter" />
<bean class="com.luyu.PreGreetingAspect">
```

AOP真是博大精深，各种网上的文章都只是简单地介绍如何使用，而不深入剖析内部原理（可能和相对复杂也有关系），想要理解AOP还是要深入去看
### <font color=red size=5>深入分析AOP原理</font>
<code>关于aop的使用，有两种方式一种是配置的方式；另一种是注解的方式（@Aspect）</code>
配置的方式是基础，主要分析这种方式
给出一个简单的基于注解的配置文件
``` xml
<bean id="testAdvisor" class=com.techlab.advisor.TestAdvisor/>
<bean id="myTestAop" class="org.springframework.aop.ProxyFactoryBean">
    <property name="proxyInterfaces">
        <value>
            com.test.AbcInterface
        <value/>
    </property>
    <property name="target">
        <bean class="com.techlab.bean.TestBean"/>
    </property>
    <property name="interceptorNames">
        <list><value>testAdvisor</value></list>
    </property>
</bean>
```
从上面对配置文件易知aop的入口点应该是ProxyFactoryBean，<code>也是通过这个bean实现了IOC和AOP的对接。</code>
ProxyFactoryBean本身也是一个FactoryBean，所以对于这种类型的bean实例化的时候，返回的对象是由其内部getObject()方法决定的，这个就是我们分析的入口点。
开车~
``` java
getObject()
    //初始化advisor链
    1.initializeAdvisorChain()
        //遍历每个配置的interceptorNames获取实例bean
        advice = this.beanFactory.getBean(name)
        //将advice包裹为advisor并放到链中
        addAdvisorOnChainCreation(advice,name)
    //返回单例的代理对象
    2.return getSingletonInstance()
        //AopProxy有两个子类实现，一个是Cglib，一个是JdkDynamicProxy
        //根据目标对象是否实现接口来决定采用哪种方式进行动态代理
        AopProxy proxy = createAopProxy())
        proxy.getProxy(this.proxyClassLoader)
            Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
            //重点看下this里面的invoke()方法，这是最核心的方法，完成了功能增强
            AopProxy#invoke()
                //获取拦截器链(这里的拦截器和mvc中的不同，这个拦截器是advice的子类)
                List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method,targetClass)
                    //根据第一步中的advisor去生成interceptor
                    registry.getInterceptors(advisor)
                    //将和方法匹配的拦截器放到拦截器链中
                    interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor,mm))
                //如果没有任何拦截器
                AopUtils.invokeJoinpointUsingReflection(target,method,args)
                    method.invoke(target,args)
                //如果有拦截器
                //proceed方法会逐个运行拦截器的方法,判断拦截器是否和当前bean匹配
                //如果运行到了链的末尾，那么直接调用目标对象的方法；否则继续执行匹配的拦截器方法
                new ReflectviceMethodInvocation(proxy,target,method,args,targetClass,chain).proceed()
                    //如果拦截器调用完毕
                    invokeJoinpoint()
                    //如果当前方法和advisor的pointcut匹配
                    interceptor.invoke(this)
                    //如果当前拦截器不匹配，递归调用proceed
                    proceed()
```
大概整个流程就是这样了，有一些细节要提一下。
之前一直想不通，假如某个方法既有前置增强又有后置增强，如果遍历拦截器（对advisor的包裹）怎么实现的先执行前置再执行方法本身最后执行后置的？看了拦截器的源码就懂了（结合proceed方法）
前置advisor与后置advisor配置的先后顺序也不会影响最终的执行结果
```java
@Override
public Object proceed() throws Throwable {
    //如果拦截器遍历完毕，那么调用方法本身的方法
    if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        return invokeJoinpoint();
    }
    //注意这里的++
    Object interceptorOrInterceptionAdvice =
            this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
    if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
        InterceptorAndDynamicMethodMatcher dm =
                (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
        //如果拦截器和当前方法匹配，那么调用拦截器的方法
        if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
            return dm.interceptor.invoke(this);
        }
        //如果拦截器不匹配，那么递归调用proceed方法
        else {
            return proceed();
        }
    }
    ...省略
}
```
```java
public class AfterReturningAdviceInterceptor implements MethodInterceptor, AfterAdvice, Serializable {
    ...省略
    @Override
	public Object invoke(MethodInvocation mi) throws Throwable {
        //先去执行其他拦截器的方法or本身的方法
		Object retVal = mi.proceed();
		this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());
		return retVal;
	}
}
```
```java
public class MethodBeforeAdviceInterceptor implements MethodInterceptor, Serializable {
    ...省略
    @Override
	public Object invoke(MethodInvocation mi) throws Throwable {
        //先去执行advice的前置方法
		this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis() );
		return mi.proceed();
	}
}
```
小结：
终于撸完了一个简单的AOP全过程，之前如果有人问题AOP的原理，我就只能说动态代理和CGLIB
现在我知道了ProxyFactoryBean将IOC和AOP连接在一起
以及为啥前置>方法>后置，通过不同advice的interceptor实现

简单地说，就是对每个目标对象创建一个BeanFactory的实例，通过getObject对实现进行改造，该方法内部通过动态代理 or CGLIB的方式对当前方法进行代理，而代理的invoke方法中会先去遍历当前target配置的advisor，然后找到和当前method匹配的advisor，之后再将匹配的advisor和method融合到一起，实现aop~

这种配置的方法其实很蛋疼，每一个需要被代理的类都要配置为一个proxyFactoryBean
那么如果不这么配置的话咋实现对每个可能的目标类的代理呢？
我们想到了Spring预留的BeanPostProcessor接口，该方法可以做到对每个符合代理要求的bean进行偷梁换柱
具体逻辑位于postProcessAfterInitialization中，也就是执行过初始化方法之后，对符合
pointCut的类进行代理，返回给getBean的调用处完成『偷梁换柱』

### <font color=red size=5>关于AOP的几问</font>
<font color=blue>AOP的底层是通过什么实现的？</font>
Spring AOP底层使用JDK动态代理和CGLIB来实现的
这里多提一句JDK动态代理相当于new了一个代理所实现接口的实例，接下来再对其方法进行覆写,写到这里引出一个问题，如果代理类本身就有的一些方法能否同样被代理？哈哈，一下就暴露了对动态代理的不熟悉,
(TargetInterface)Proxy.newProxyInstance,注意前面那个强制转换，此时是无法调用非接口方法的。
缺点：
* JDK动态代理需要代理类实现至少一个接口
* 且只能对接口中的方法进行代理

当代理类未实现任何接口或者想要代理目标的所有方法而不止是实现自接口的方法，此时需要使用CGLIB代理，CGLIB相当于new了一个代理类的子类，对子类的方法进行覆写以支持代理功能
缺点：
* 但是这样有一个问题就是无法覆写代理类本身生命为final的方法
* CGLib在创建代理对象时所花费的时间却比JDK多得多

关于JDK和CGLIB的总结
* 如果目标对象实现了接口，默认情况使用JDK的动态代理实现AOP
* 如果目标对象实现了接口，可以强制使用CGLIB，设置方式aop:aspectj-autoproxy proxy-target-class="true"
* 如果目标对象没有实现接口，必须采用CGLIB库

<font color=blue>基于aspectJ注解的动态代理是如何偷梁换柱的？</font>
这里描述一些源码细节，AOP代理对象是通过AnnotationAwareAspectJAutoProxyCreator创建的，而该类实现了BeanPostProcessor方法，在该接口的postProcessAfterInitialization方法中实现了偷梁换柱，这部分可以参考下Bean的生命周期  https://luyu05.github.io/2018/07/26/Spring-1/

<font color=blue>AOP的应用？</font>
最常用的一个地方就是数据库事务，除了本身的业务逻辑代码只需一个注解就ok。建立在 AOP 的基础之上的。其本质是对方法前后进行拦截，然后在目标方法开始之前创建或者加入一个事务，在执行完目标方法之后根据执行情况提交或者回滚事务。
