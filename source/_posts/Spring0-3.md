---
title: Spring源码阅读(5)-基于@Aspect的AOP实现原理
date: 2018-11-19 20:23:17
tags: Spring源码阅读
categories: Spring源码阅读
top : 95
---
上篇文章分析了比较原始的AOP的实现方式，最后我们也分析了那种方式的弊端，本文分析一种更为炫酷的AOP的实现方式。
<!-- more -->
# 自动化Demo
```java
@Aspect
public class AspectJTest {

    @Pointcut("execution(* *.test(..))")
    public void test() {}

    @Before("test()")
    public void beforeTest() {
        System.out.println("before method advice");
    }

    @After("test()")
    public void afterTest() {System.out.println("after method advice");}

}

public class MyServiceImpl implements MyService {
    public void test() {
        System.out.println("i am service test()");
    }
}
```
```xml
<aop:aspectj-autoproxy/>

<bean id="myService" class="com.techlab.spring.aop.MyServiceImpl"/>

<bean class="com.techlab.spring.aop.AspectJTest"/>
```
```java
public class EntranceApp {
    public static void main(String[] args) throws Exception {
        ApplicationContext context =
                new ClassPathXmlApplicationContext("beans.xml");
        MyService b = (MyService)context.getBean("myService");
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
# 源码分析
## AnnotationAwareAspectJAutoProxyCreator的注册
诶，我们看到配置文件清爽了很多，而且我们不需要针对每一个代理对象引入一个ProxyFactoryBean了。显然是这段代码起的作用<code>&lt;aop:aspectj-autoproxy/&gt;</code>
这里要普及一个知识点，对于自定义的注解（annotation），一定有与之相匹配的解析器（parser）。
那么我们就可以去寻找和当前注解匹配的解析器，我们全局搜索了下这个注解找到了与之相匹配的解析器。

---
Tips：
在容器启动阶段，1)需要根据xml生成相应的resource；2)接着根据resource去获取document；3)最后对document进行解析来生成一个个beanDefination并注册到容器中。
而对<code>&lt;aop:aspectj-autoproxy/&gt;</code>的解析就在第三步骤中，找到了对应的parser并调用parse方法，该方法内部会将一个特殊的BeanPostProcessor以BeanDefination形式注册到容器中。
同理针对@Resource以及@Autowire注解也有对应的AutowiredAnnotationBeanPostProcessor,CommonAnnotationBeanPostProcessor，这两个类一般是通过postProcessPropertyValues方法来发挥作用。而这两个BeanPostProcessor的BeanDefination注册也是通过对&lt;context:annotation-config/&gt;的解析实现的。 Ps 这两个BeanPostProcessor都是InstantiationAwareBeanPostProcessor的实现
果然不出老夫所料，是在ComponentScanBeanDefinitionParser的parse方法中调用了AnnotationConfigUtils的registerAnnotationConfigProcessors方法，该方法直接new了很多相关BeanPostProcessor的defination。
Ps有点举一反三的能力了

---
```java
class AspectJAutoProxyBeanDefinitionParser implements BeanDefinitionParser {
	@Override
	public BeanDefinition parse(Element element, ParserContext parserContext) {
		AopNamespaceUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(parserContext, element);
		extendBeanDefinition(element, parserContext);
		return null;
    }
}
```
我们重点关注第一行代码，根据名字我们就知道这个方法是用来注册一个『支持AspectJ注解的自动代理创建器』，我们深入这个方法去瞅瞅
```java
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry, Object source) {
    return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
}

private static BeanDefinition registerOrEscalateApcAsRequired(Class<?> cls, BeanDefinitionRegistry registry, Object source) {
    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
        BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
        if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
            int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
            int requiredPriority = findPriorityForClass(cls);
            if (currentPriority < requiredPriority) {
                apcDefinition.setBeanClassName(cls.getName());
            }
        }
        return null;
    }
    RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
    beanDefinition.setSource(source);
    beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
    beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
    registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
    return beanDefinition;
}
```
首先先去看看是否已经创建了自动代理创建器。如果创建了那么和当前的创建器进行优先级的比较，并将beanDefination替换为优先级较高的那个并返回；如果未创建，那么需要new一个BeanDefinition、填充部分属性并注册到容器中。至此，『支持AspectJ注解的自动代理创建器』已经注册到容器中了，我们接下来看看这个创建器到底是何方神圣。
## AnnotationAwareAspectJAutoProxyCreator详解
我们先来看看这个AnnotationAwareAspectJAutoProxyCreator的类结构图
<img src='001.jpg'>
我们定位到BeanPostProcessor这个接口，我们知道这个接口是容器的一个扩展点，可以在bean初始化的前后执行一些自定义的操作。而我们重点关注的就是AnnotationAwareAspectJAutoProxyCreator#postProcessAfterInitialization方法
```java
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (!this.earlyProxyReferences.contains(cacheKey)) {
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}
```
这里主要看下
```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    //...
    // Create proxy if we have advice.
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        Object proxy = createProxy(bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```
和ProxyFactoryBean类似，这里也是分成了两个步骤，首先是获取当前bean匹配的advisor list；之后创建代理对象并返回。
### 获取advisor list
```java
@Override
protected Object[] getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, TargetSource targetSource) {
    List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
    if (advisors.isEmpty()) {
        return DO_NOT_PROXY;
    }
    return advisors.toArray();
}
```
我们看到最终都是以advisor的形式返回给调用方的。
<font color=red>而且这里值得注意的是，如果当前没有匹配的advisor（说明无需代理）那么直接返回无需代理的标识。AnnotationAwareAspectJAutoProxyCreator默认对所有bean生效，但是对于不需要代理的对象会直接返回。</font>
```java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
    List<Advisor> candidateAdvisors = findCandidateAdvisors();
    List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
    extendAdvisors(eligibleAdvisors);
    if (!eligibleAdvisors.isEmpty()) {
        eligibleAdvisors = sortAdvisors(eligibleAdvisors);
    }
    return eligibleAdvisors;
}
```
#### findAdvisorBeans
步骤一：我们看第一行代码，根据方法名称猜测是找到所有的Advisor，注意这个时候是没有传入beanClass和beanName的
囧，这里之前跟代码居然跟错了，本来应该是调用子类的方法却跟到了父类中，<code>调试真乃阅读源码之神器</code>
```java
protected List<Advisor> findCandidateAdvisors() {
    // Add all the Spring advisors found according to superclass rules.
    List<Advisor> advisors = super.findCandidateAdvisors();
    // Build Advisors for all AspectJ aspects in the bean factory.
    advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
    return advisors;
}
```
首先调用父类的findCandidateAdvisors方法，而这个方法很简单核心就是这行代码<code>BeanFactoryUtils.beanNamesForTypeIncludingAncestors(this.beanFactory, Advisor.class, true, false)</code>，就是找到所有普通的Advisor。
接着添加了aspectJ定义的advisors，我们重点关注这种情况。
```java
public List<Advisor> buildAspectJAdvisors() {
    List<String> aspectNames = null;
    synchronized (this) {
        aspectNames = this.aspectBeanNames;
        if (aspectNames == null) {
            List<Advisor> advisors = new LinkedList<Advisor>();
            aspectNames = new LinkedList<String>();
            String[] beanNames =
                    BeanFactoryUtils.beanNamesForTypeIncludingAncestors(this.beanFactory, Object.class, true, false);
            for (String beanName : beanNames) {
                if (!isEligibleBean(beanName)) {
                    continue;
                }
                // We must be careful not to instantiate beans eagerly as in this
                // case they would be cached by the Spring container but would not
                // have been weaved
                Class<?> beanType = this.beanFactory.getType(beanName);
                if (beanType == null) {
                    continue;
                }
                if (this.advisorFactory.isAspect(beanType)) {
                    aspectNames.add(beanName);
                    AspectMetadata amd = new AspectMetadata(beanType, beanName);
                    if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
                        MetadataAwareAspectInstanceFactory factory =
                                new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
                        //这里一个Aspect类型的bean，是有可能包含多于一个的Advisor的
                        List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
                        if (this.beanFactory.isSingleton(beanName)) {
                            this.advisorsCache.put(beanName, classAdvisors);
                        }
                        else {
                            this.aspectFactoryCache.put(beanName, factory);
                        }
                        advisors.addAll(classAdvisors);
                    }
                    //...
                }
            }
            this.aspectBeanNames = aspectNames;
            return advisors;
        }
    }

    if (aspectNames.isEmpty()) {
        return Collections.emptyList();
    }
    List<Advisor> advisors = new LinkedList<Advisor>();
    for (String aspectName : aspectNames) {
        List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
        if (cachedAdvisors != null) {
            advisors.addAll(cachedAdvisors);
        }
        else {
            MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
            advisors.addAll(this.advisorFactory.getAdvisors(factory));
        }
    }
    return advisors;
}
```
方法有点长，我们一点点去分析。
（1）首先加了个锁，很简单因为List类型的aspectNames是非线程安全的
（2）接着代码进入到同步代码块中，在这里非常丧病的找到了当前容器中的所有beanName
（3）遍历每个beanName，根据beanName获取beanType，接着根据是否有注解去判断是否是Aspect类型的
（4）将Aspect类型的advisor一同放到advisors中（这里一个Aspect类型的bean，是有可能包含多于一个的Advisor的）
#### findAdvisorsThatCanApply
步骤二：接着从步骤一返回的Advisor中找到和当前bean匹配的Advisor Ps代码中忽略了introductionAdvisor
```java
public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
    //...
    for (Advisor candidate : candidateAdvisors) {
        //...
        if (canApply(candidate, clazz, hasIntroductions)) {
            eligibleAdvisors.add(candidate);
        }
    }
    //...
    return eligibleAdvisors;
}
public static boolean canApply(Advisor advisor, Class<?> targetClass, boolean hasIntroductions) {
    //..
    else if (advisor instanceof PointcutAdvisor) {
        PointcutAdvisor pca = (PointcutAdvisor) advisor;
        return canApply(pca.getPointcut(), targetClass, hasIntroductions);
    }
    else {
        // It doesn't have a pointcut so we assume it applies.
        return true;
    }
}
public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
    Assert.notNull(pc, "Pointcut must not be null");
    if (!pc.getClassFilter().matches(targetClass)) {
        return false;
    }

    MethodMatcher methodMatcher = pc.getMethodMatcher();
    //...
    Set<Class<?>> classes = new HashSet<Class<?>>(ClassUtils.getAllInterfacesForClassAsSet(targetClass));
    classes.add(targetClass);
    //这里只要advisor和当前bean or bean的父类的任意方法匹配，就会被加入到合格的advisor list中
    for (Class<?> clazz : classes) {
        Method[] methods = clazz.getMethods();
        for (Method method : methods) {
            if ((introductionAwareMethodMatcher != null &&
                    introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions)) ||
                    methodMatcher.matches(method, targetClass)) {
                return true;
            }
        }
    }

    return false;
}
```
我们截取的代码忽略了关于IntroductionAdvisor的相关代码，可以看到代码逻辑相对简单，遍历所有的advisor找到和当前对象相匹配的advisor们(利用Advisor的PointCut判断是否和当前类的任何方法相匹配)，最后将符合条件的advisor放到eligibleAdvisors中

Ps：在我们Demo中普通类型的Advisor数量是0；AspectJ类型的Advisor数量是2。
### 根据获取的增强去创建代理
```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    //...
    // Create proxy if we have advice.
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        Object proxy = createProxy(bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```
接着是这个方法
```java
protected Object createProxy(
        Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {

    ProxyFactory proxyFactory = new ProxyFactory();
    // Copy our properties (proxyTargetClass etc) inherited from ProxyConfig.
    proxyFactory.copyFrom(this);

    if (!shouldProxyTargetClass(beanClass, beanName)) {
        // Must allow for introductions; can't just set interfaces to
        // the target's interfaces only.
        Class<?>[] targetInterfaces = ClassUtils.getAllInterfacesForClass(beanClass, this.proxyClassLoader);
        for (Class<?> targetInterface : targetInterfaces) {
            proxyFactory.addInterface(targetInterface);
        }
    }

    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    for (Advisor advisor : advisors) {
        proxyFactory.addAdvisor(advisor);
    }

    proxyFactory.setTargetSource(targetSource);
    customizeProxyFactory(proxyFactory);

    proxyFactory.setFrozen(this.freezeProxy);
    if (advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
    }

    return proxyFactory.getProxy(this.proxyClassLoader);
}
```
将我们获得的与当前类匹配的Advisors放到ProxyFactory中，接着创建动态代理，<code>Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);</code>

至此，自动的AOP和半自动的AOP一样走到了同一个岔路口，可谓殊途同归。既然代理对象的创建的逻辑都一样，那么后续Proxy的invoke方法逻辑也就完全相同了。（包括根据class&method找到匹配的interceptor、调用方法with interceptor chain）

---
小结：
1）IOC容器启动阶段需要对document进行解析以生成一个个的BeanDefination并注册到容器中
2）在这个步骤中会对我们在xml中写的<code>&lt;aop:aspectj-autoproxy/&gt;</code>进行解析
3）解析的过程就是创建了一个<code>AnnotationAwareAspectJAutoProxyCreator.class</code>类型的BeanDefination并注册到了容器中
4）IOC流程继续执行的时候会调用<code>BeanPostProcessor.postProcessAfterInitialiization()</code>
5）而我们注册的AnnotationAwareAspectJAutoProxyCreator就是一个BeanPostProcessor，此时执行它对应的postProcessAfterInitialiization方法
6）该方法首先获取advisor list，包括普通的advisor以及aspectJ类型的advisor（只有和任意当前类以及当前类实现的接口的方法相匹配的advisor才会被加入到list中）
7）接着根据获取到的advisor创建动态代理并返回（此时获取到的bean已经是动态代理对象了）
8）当调用bean的方法时候，会触发动态代理对象的invoke方法，该方法会找到和当前class & method匹配的advisor并包装成interceptor，接着执行经典的proceed方法。

Tips：
和上文的半自动aop不同，最开始寻找advisor chain的时候只会找到和当前bean的任意接口任意方法匹配的advisor，如果找到才说明它需要被代理，而在真正执行invoke方法时，才会根据method进行进一步的甄别并封装成interceptor。而上文的半自动aop会找到配置的所有的advisor，并且一定会将配置的target包装为代理。这是两者的区别，请注意。
无论是哪种方式，最终都是要创建持有advisor的动态代理对象，而该对象内部的invoke方法会从advisor中找到和当前class method匹配的advisor并包装为interceptor，最后执行方法调用。

<img src='002.jpg'>

<img src='003.jpg'>

至此，AOP专题到此结束，源码分析过程主要分析了一些主干流程，很多也很重要的点被砍掉了。以后有机会对那些部分再深入研究。

That's all.
2018-11-19 20:06:01 上海

---