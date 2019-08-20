---
title: Spring源码阅读(4)-基于FactoryBean的AOP实现原理
date: 2018-11-19 20:22:17
tags: Spring源码阅读
categories: Spring源码阅读
top : 96
---
上篇文章分析了FactoryBean的工作原理，本文以FactoryBean的实现ProxyFactoryBean为切入点分析AOP的实现原理。
<!-- more -->
# Demo
```java
public interface MyService {
    void test();
}
public class MyServiceImpl implements MyService {
    public void test() {
        System.out.println("i am service test()");
    }
}
public class MyAfterAdvice implements AfterReturningAdvice {
    public void afterReturning(Object returnValue, Method method, Object[] args, Object target) throws Throwable {
        System.out.println("after method advice");
    }
}
public class MyBeforeAdvice implements MethodBeforeAdvice {
    public void before(Method method, Object[] args, Object target) throws Throwable {
        System.out.println("before method advice");
    }
}
```
```xml
<bean id="myService" class="com.techlab.spring.aop.MyServiceImpl"/>

<bean id="beforeAdvice" class="com.techlab.spring.aop.MyBeforeAdvice"/>

<bean id="afterAdvice" class="com.techlab.spring.aop.MyAfterAdvice"/>

<bean id="testAop" class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="interfaces" value="com.techlab.spring.aop.MyService"/>
    <property name="target">
        <ref bean="myService"/>
    </property>
    <property name="interceptorNames">
        <list>
            <value>beforeAdvice</value>
            <value>afterAdvice</value>
        </list>
    </property>
</bean>
```
```java
public class EntranceApp {
    public static void main(String[] args) throws Exception {
        ApplicationContext context =
                new ClassPathXmlApplicationContext("beans.xml");
        MyService b = (MyService)context.getBean("testAop");
        b.test();
    }
}
```
控制台输出
```java
before method advice
i am service test()
after method advice
```
关于这个配置文件有几个地方要说明，这里的interceptorNames属性，既可以配置advice也可以配置advisor，如果只配置advice的情况下，那么默认的pointCut就是匹配所有的类所有的方法。如果想实现自定义的方法匹配，需要自己实现advisor。
interceptorNames还支持通配符，这样会找到所有匹配的advice and advisor。
既然是FactoryBean那么它最核心的方法就是getObject()方法了
# ProxyFactoryBean#getObject()
```java
@Override
public Object getObject() throws BeansException {
    initializeAdvisorChain();
    if (isSingleton()) {
        return getSingletonInstance();
    }
    else {
        if (this.targetName == null) {
            logger.warn("Using non-singleton proxies with singleton targets is often undesirable. " +
                    "Enable prototype proxies by setting the 'targetName' property.");
        }
        return newPrototypeInstance();
    }
}
```
先来概览下上面这个方法，首先是初始化AdvisorChain；接着返回代理对象。
## initializeAdvisorChain
```java
private synchronized void initializeAdvisorChain() throws AopConfigException, BeansException {
    if (this.advisorChainInitialized) {
        return;
    }
    if (!ObjectUtils.isEmpty(this.interceptorNames)) {
        //...
        for (String name : this.interceptorNames) {
            //...
            Object advice;
            if (this.singleton || this.beanFactory.isSingleton(name)) {
                // Add the real Advisor/Advice to the chain.
                advice = this.beanFactory.getBean(name);
            }
            //...
            addAdvisorOnChainCreation(advice, name);
        }
    }
    this.advisorChainInitialized = true;
}
```
这个方法简化后如上所示，主要就是遍历interceptorNames，利用BeanFactory找到每个advice的实例。之后调用<code>addAdvisorOnChainCreation(advice, name)</code>。 

Ps 这里的参数名字有点意思interceptorNames，无论配置的是advice还是advisor最后都会被包装成匹配的interceptor，实际上最开始即使配置了advice也会先包装成advisor，定位到具体的类和方法后，会卸去包装还原为advice，然后再将advice包装成interceptor。

```java
private void addAdvisorOnChainCreation(Object next, String name) {
    // We need to convert to an Advisor if necessary so that our source reference
    // matches what we find from superclass interceptors.
    Advisor advisor = namedBeanToAdvisor(next);
    if (logger.isTraceEnabled()) {
        logger.trace("Adding advisor with name '" + name + "'");
    }
    addAdvisor(advisor);
}
```
<font color=red>我们可以简单地理解为 Advisor = Advice(逻辑+相对方法的方位) + POINTCUT（定位），而POINTCUT=ClassFilter（类定位）+ MethodMatcher（方法定位）</font>
首先，将advice类型的bean包装成advisor，如果本身就是advisor那么不需要包装。
接着，将advisor放到advisor list中。第二步骤很简单，我们重点看第一个步骤。
```java
private Advisor namedBeanToAdvisor(Object next) {
    //...
    try {
        return this.advisorAdapterRegistry.wrap(next);
    }
}

public Advisor wrap(Object adviceObject){
    //如果是advisor那么无须包装
    if (adviceObject instanceof Advisor) {
        return (Advisor) adviceObject;
    }
    //...
    return new DefaultPointcutAdvisor(advice);
}

public DefaultPointcutAdvisor(Advice advice) {
    this(Pointcut.TRUE, advice);
}

public DefaultPointcutAdvisor(Pointcut pointcut, Advice advice) {
    this.pointcut = pointcut;
    setAdvice(advice);
}
```
可以看到如果我们只传advice进来，那么包装成advisor的时候，默认的Pointcut就是Pointcut.TRUE，表示匹配所有类的所有方法。而接下来的方法很简单了，就是保存pointcut和advice。包装成Advisor后，塞到Advisor链中，至此第一个步骤就结束了。

---
小结：
（1）根据xml中配置的interceptorNames的list属性，去利用BeanFactory.getBean()找到每个对应的实例
（2）如果bean本身就是advisor类型的，那么直接加入到advisor的list中
（3）如果bean是advice类型的，那么还需要对它进行包装，包装为advisor（Ps此时配套的PointCut是匹配所有的方法的）
（4）包装结束后，同样加入到advisor的list中，这个list由ProxyFactoryBean持有

---
## getSingletonInstance()
```java
private synchronized Object getSingletonInstance() {
    //...
    return getProxy(createAopProxy());
}
```
这里又分为两个步骤
### createAopProxy()
```java
protected final synchronized AopProxy createAopProxy() {
    if (!this.active) {
        activate();
    }
    return getAopProxyFactory().createAopProxy(this);
}

public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
        Class<?> targetClass = config.getTargetClass();
        if (targetClass == null) {
            throw new AopConfigException("TargetSource cannot determine target class: " +
                    "Either an interface or a target is required for proxy creation.");
        }
        if (targetClass.isInterface()) {
            return new JdkDynamicAopProxy(config);
        }
        return new ObjenesisCglibAopProxy(config);
    }
    else {
        return new JdkDynamicAopProxy(config);
    }
}
```
第一个方法中我们看到<code>createAopProxy</code>的入参是this，这里this表示的就是ProxyFactoryBean（持有advisorChain）
第二个方法中，如果目标类未实现任何接口，那么通过cglib创建动态代理；反之，通过jdk默认的方式创建动态代理。这里就是简单的new了一个代理对象JdkDynamicAopProxy,并且把一些配置参数（config=proxyFactoryBean）一起传给Proxy。
### getProxy(AopProxy)
```java
protected Object getProxy(AopProxy aopProxy) {
    return aopProxy.getProxy(this.proxyClassLoader);
}

public Object getProxy(ClassLoader classLoader) {
    //...
    Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised);
    findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
    return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```
我们看到了经典的<code>Proxy.newProxyInstance(classLoader, proxiedInterfaces, this)</code>，注意这里的this，此时表示的是JdkDynamicAopProxy实例。
我们看过了getObject方法，下面要看看代理类在调用方法的过程中，具体的执行逻辑也就是invoke方法

# 动态代理的invoke方法
代码来到，JdkDynamicAopProxy#invoke
```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    MethodInvocation invocation;
    Object oldProxy = null;
    boolean setProxyContext = false;

    TargetSource targetSource = this.advised.targetSource;
    Class<?> targetClass = null;
    Object target = null;

    try {
        //如果代理接口中没有覆写hashCode和equals方法，那么直接返回当前类覆写的对应方法
        //并且不会执行代理逻辑
        if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
            // The target does not implement the equals(Object) method itself.
            return equals(args[0]);
        }
        if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
            // The target does not implement the hashCode() method itself.
            return hashCode();
        }
        if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
                method.getDeclaringClass().isAssignableFrom(Advised.class)) {
            // Service invocations on ProxyConfig with the proxy config...
            return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
        }

        Object retVal;

        if (this.advised.exposeProxy) {
            // Make invocation available if necessary.
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }

        // May be null. Get as late as possible to minimize the time we "own" the target,
        // in case it comes from a pool.
        target = targetSource.getTarget();
        if (target != null) {
            targetClass = target.getClass();
        }

        // Get the interception chain for this method.
        List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

        // Check whether we have any advice. If we don't, we can fallback on direct
        // reflective invocation of the target, and avoid creating a MethodInvocation.
        if (chain.isEmpty()) {
            // We can skip creating a MethodInvocation: just invoke the target directly
            // Note that the final invoker must be an InvokerInterceptor so we know it does
            // nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
            retVal = AopUtils.invokeJoinpointUsingReflection(target, method, args);
        }
        else {
            // We need to create a method invocation...
            invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
            // Proceed to the joinpoint through the interceptor chain.
            retVal = invocation.proceed();
        }

        // Massage return value if necessary.
        Class<?> returnType = method.getReturnType();
        if (retVal != null && retVal == target && returnType.isInstance(proxy) &&
                !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
            // Special case: it returned "this" and the return type of the method
            // is type-compatible. Note that we can't help if the target sets
            // a reference to itself in another returned object.
            retVal = proxy;
        } else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
            throw new AopInvocationException("Null return value from advice does not match primitive return type for: " + method);
        }
        return retVal;
    }
    finally {
        if (target != null && !targetSource.isStatic()) {
            // Must have come from TargetSource.
            targetSource.releaseTarget(target);
        }
        if (setProxyContext) {
            // Restore old proxy.
            AopContext.setCurrentProxy(oldProxy);
        }
    }
}
```
我们还是人为的分为两个步骤，1）根据class和method获取匹配的interceptorChain；2）调用方法with interceptorChain chain
先来看步骤一
## 根据class和method获取匹配的interceptorChain
```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, Class<?> targetClass) {
    MethodCacheKey cacheKey = new MethodCacheKey(method);
    List<Object> cached = this.methodCache.get(cacheKey);
    if (cached == null) {
        cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
                this, method, targetClass);
        this.methodCache.put(cacheKey, cached);
    }
    return cached;
}
```
可以看到，会针对与当前class & method匹配的chain进行缓存。我们观察无缓存的情况
```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
			Advised config, Method method, Class<?> targetClass) {
    //...
    List<Object> interceptorList = new ArrayList<Object>(config.getAdvisors().length);
    //...
    for (Advisor advisor : config.getAdvisors()) {
        if (advisor instanceof PointcutAdvisor) {
            // Add it conditionally.
            PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
            if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(targetClass)) {
                MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
                MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
                if (MethodMatchers.matches(mm, method, targetClass, hasIntroductions)) {
                    if (mm.isRuntime()) {
                        // Creating a new object instance in the getInterceptors() method
                        // isn't a problem as we normally cache created chains.
                        for (MethodInterceptor interceptor : interceptors) {
                            interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
                        }
                    }
                    else {
                        interceptorList.addAll(Arrays.asList(interceptors));
                    }
                }
            }
        }
        //...
    }
    return interceptorList;
}
```
遍历每个advisor（包括本身配置的以及advice被包裹成的advisor），找到和当前class、method匹配的advisor，在这中间调用了一个很重要的方法<code>registry.getInterceptors(advisor)</code>该方法会将advisor包裹为对应类型的interceptor
```java
//DefaultAdvisorAdapterRegistry.class
public DefaultAdvisorAdapterRegistry() {
    registerAdvisorAdapter(new MethodBeforeAdviceAdapter());
    registerAdvisorAdapter(new AfterReturningAdviceAdapter());
    registerAdvisorAdapter(new ThrowsAdviceAdapter());
}
    
public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
    List<MethodInterceptor> interceptors = new ArrayList<MethodInterceptor>(3);
    Advice advice = advisor.getAdvice();
    if (advice instanceof MethodInterceptor) {
        interceptors.add((MethodInterceptor) advice);
    }
    //判断属于哪种类型的advisor，前置、后置、异常 并构造对应interceptor
    for (AdvisorAdapter adapter : this.adapters) {
        if (adapter.supportsAdvice(advice)) {
            interceptors.add(adapter.getInterceptor(advisor));
        }
    }
    if (interceptors.isEmpty()) {
        throw new UnknownAdviceTypeException(advisor.getAdvice());
    }
    return interceptors.toArray(new MethodInterceptor[interceptors.size()]);
}
```
我们看到类的构造方法中，先是将3个Adapter放到了list中。以MethodBeforeAdviceAdapter为例，我们看看这个类
```java
class MethodBeforeAdviceAdapter implements AdvisorAdapter, Serializable {

	@Override
	public boolean supportsAdvice(Advice advice) {
		return (advice instanceof MethodBeforeAdvice);
    }
    
	@Override
	public MethodInterceptor getInterceptor(Advisor advisor) {
		MethodBeforeAdvice advice = (MethodBeforeAdvice) advisor.getAdvice();
		return new MethodBeforeAdviceInterceptor(advice);
    }
    
}
```
可以看到这个类还是很简单的，第一个方法就是用来判断当前的advice是否是MethodBeforeAdvice类型的；第二个方法是将advice封装到intreceptor中。
Ps：注意这里已经定位到了具体的类和方法，此时Advisor可以甩下累赘，只用advice来包装成interceptor了
```java
public class MethodBeforeAdviceInterceptor implements MethodInterceptor, Serializable {

	private MethodBeforeAdvice advice;


	/**
	 * Create a new MethodBeforeAdviceInterceptor for the given advice.
	 * @param advice the MethodBeforeAdvice to wrap
	 */
	public MethodBeforeAdviceInterceptor(MethodBeforeAdvice advice) {
		Assert.notNull(advice, "Advice must not be null");
		this.advice = advice;
	}

	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis() );
		return mi.proceed();
	}

}
```
这里先留意下，invoke方法，我们看到先是调用了advice的before方法，再调用了proceed，这个proceed方法我们后续会去分析。至此，已经将匹配的advice包裹成了标准的interceptor，并存放在interceptors这个list中（这个就是interceptor chain）
Tips：一个advisor(advice),可能被封装为多个interceptor,因为advisor(advice)可能同时实现了多个接口。
## 调用方法with interceptor chain
```java
if (chain.isEmpty()) {
    // We can skip creating a MethodInvocation: just invoke the target directly
    // Note that the final invoker must be an InvokerInterceptor so we know it does
    // nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
    retVal = AopUtils.invokeJoinpointUsingReflection(target, method, args);
}
else {
    // We need to create a method invocation...
    invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
    // Proceed to the joinpoint through the interceptor chain.
    retVal = invocation.proceed();
}
```
如果interceptorChain是空的，那么很简单直接通过反射执行方法就ok了；反之，代码来到了proceed
```java
public Object proceed() throws Throwable {
    //	We start with an index of -1 and increment early.
    if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        return invokeJoinpoint();
    }

    Object interceptorOrInterceptionAdvice =
            this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
    if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
        // Evaluate dynamic method matcher here: static part will already have
        // been evaluated and found to match.
        InterceptorAndDynamicMethodMatcher dm =
                (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
        if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
            return dm.interceptor.invoke(this);
        }
        else {
            // Dynamic matching failed.
            // Skip this interceptor and invoke the next in the chain.
            return proceed();
        }
    }
    else {
        // It's an interceptor, so we just invoke it: The pointcut will have
        // been evaluated statically before this object was constructed.
        return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
    }
}
```
这里的interceptorsAndDynamicMethodMatchers其实就是刚刚的interceptorChain
这里的index是从-1开始的，所以最开始代码的意思是如果这个array被遍历光了，那么就可以执行本身的方法了；如果并未遍历结束，那么获取下一个interceptor并调用其invoke方法。
我们通过通过Demo中的例子来看看这个方法，假设在chain中的顺序是{MyAfterInterceptor,MyBeforeAdviceInterceptor}，此时先执行after的invoke方法
```java
public Object invoke(MethodInvocation mi) throws Throwable {
    Object retVal = mi.proceed();
    this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());
    return retVal;
}
```
我们看到此时会继续调用proceed方法，此时遍历到了MyBeforeAdvice，执行它的invoke方法
```java
public Object invoke(MethodInvocation mi) throws Throwable {
    this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis() );
    return mi.proceed();
}
```
显然代码的执行顺序是，before->method->afterReturning。这和我们的预期也是相符的。

---
小结：
1.ProxyFactoryBean的getObject方法返回了一个代理对象到调用方
2.而在getObject方法中做了两件事，1）初始化advisor链，2）创建动态代理对象并返回（此时代理持有advisor链）
3.接着执行代理对象的方法会调用其invoke方法，而invoke方法也做了两件事，1）根据class和method找到匹配的advisor并将其封装为interceptor放到interceptor链中；2）调用proceed方法，该方法会按我们的预期的执行顺序对方法进行织入

再小结：
1.初始化advisorChain
2.返回代理对象（jdk or cglib）
3.调用方法时，根据方法&类找到匹配的interceptor chain
4.执行方法 with interceptor chain

这里尤其要注意Advice,Advisor,Interceptor之间的关系
Advice = 织入逻辑 + 相对方法的织入位置
Advisor = Advice + Pointcut
Pointcut = ClassFilter + MethodMatcher
Advice : Advisor : Interceptor = 1 : 1 : n，注意一个advice可能继承了很多接口，那么它对应的Interceptor就可能有多个，Interceptor保证了方法的有序调用。

本文分析了基于ProxyFactoryBean实现的AOP，所有aop的实现原理都大体是这样的。
但是这种方式的缺点是显而易见的，我们需要针对每个目标对象，给出他们各自对应的ProxyFactoryBean的配置，下篇文章我们看看有没有『智能』一些的方法解决这个问题。

---