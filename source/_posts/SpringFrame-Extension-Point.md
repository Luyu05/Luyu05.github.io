---
title: Spring Framework常见扩展点及其应用
date: 2019-06-26 19:04:35
tags: Spring
categories: Spring
top : 100
---
本文主要介绍Spring Framework常见扩展点及其在工程上的应用
<!-- more -->
# <font color=green>综述</font>
Spring Framework包含两个核心概念，IOC(Inversion of Control)和AOP(Aspect Oriented Programming)

<code>Spring Framework的设计理念：解耦</code>
## <font color=#0099ff>为什么引入IOC？</font>
先来看一个例子
``` java
class Programer {
    Computer computer = new Mac2015();
    private void work() {
        computer.help();
    }
}
```
此时有一个问题就是programer和computer耦合在一起，这个programer不具备扩展性（它只会用mac2015），如果此时公司换了一批电脑Mac2016，那么需要换一批新的程序员，这显然是不合理的。
从设计的角度来讲，<font color=red>类本身就是定义的一个模板亦或是一种抽象</font>，如果抽象和具体的实现绑定或者耦合在一起，那么就不是纯粹的抽象了。这也违背了设计模式中的don't call me法则。

所以这个时候要把computer和programer解耦。解耦的方法很简单，computer的具体实现由调用方指定就ok，而不是在类内部自行指定。那么类就需要暴露一些点供外界实现注入功能。

常见的方法有三种
1)构造函数注入
2)set注入
3)接口注入

这就是Spring 依赖注入的基本思路
那么选择何种注入方式以及需要注入哪些属性是怎么通知到Spring的呢？

常见的方法有两种
1)注解（@resource/@autowire）
2)配置文件（xml/properties）

举个例子
```java
@Service
class TestService {
    @Resource //注解方式通知Spring 1.通过set注入方式 2.注入userService属性
    private UserService userService;
}
```
```xml
<bean id='testService' class='com.luyu.TestService'>
    <property name="userService" ref="userService"> //配置文件方式通知Spring 通过set注入方式 注入userService属性
<bean />
```

<code>IOC：实现了当前类和所依赖类的具体实现间的解耦</code>

## <font color=#0099ff>为什么引入AOP？</font>
AOP旨在将辅助逻辑（日志、安全、监控等）从业务主体逻辑中进行剥离，实现关注点分离，以提高程序的模块化程度

举个例子，下面是一个向db提交任务的简单方法，该方法在提交到db前需要放到blocking queue中，后台启动线程将queue中的任务插入es中
```java
public long addTask(DispatchTaskEntity taskEntity) {
    //...
    taskDao.addTask(entity);
    //...
}
```
```xml
<bean id="myIntercepter" class="com.MyIntercepter"/>
<!-- 新增任务aop -->
<bean id="addOfdispatchTaskAdvisor" class="org.springframework.aop.aspectj.AspectJExpressionPointcutAdvisor"> 
  <property name="advice" ref="myIntercepter" /> 
  <property name="expression" value="execution(* com.TaskDaoImpl.addTask(..))" /> 
</bean> 
```
```java
public class MyIntercepter implements MethodInterceptor{
  
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        Method method = invocation.getMethod();
        String methodName = method.getName();
        if("addTask".equalsIgnoreCase(methodName)){
            return dealAdd(invocation);
        }
        //origin method
        return invocation.proceed();
    }
  
  private Object dealAdd(MethodInvocation invocation) throws Throwable {
        Object result = invocation.proceed();
        Long taskId = (Long) result;
        if(taskId > 0){
            TaskDo taskEntity = taskDao.loadTaskById(taskId);
            //异步插入es，通过先插入bq实现
            EsHelper.recordDetail(taskEntity, MonitorType.Add);
        }
        return result;
    }
  
}
```
<code>AOP：实现了业务逻辑与辅助逻辑之间的解耦</code>

# <font color=green>Bean的生命周期</font>
Bean作为Spring Framework的核心，我们有必要研究下它的生命周期
## <font color=#0099ff>结论</font>
简单地将Bean的生命周期分为五个主干流程
1.容器启动阶段（严格来讲这个不属于Bean的生命周期）
2.Bean（单例非懒加载）的实例化阶段
3.Bean的属性注入阶段
4.Bean的初始化阶段
5.Bean的销毁阶段
下图是整个Spring容器的启动流程，可以看到除了上述五个主干流程外，Spring还提供了很多扩展点（这部分将在第三节讲述）
<img src='001.jpg'>
## <font color=#0099ff>验证</font>
通过一个实际的例子验证上图
首先定义一个业务Bean，实现了诸多上述的扩展点接口
```java
public class Car implements BeanFactoryAware, BeanNameAware,
        InitializingBean, DisposableBean {
    private String carName;
    private BeanFactory beanFactory;
    private String beanName;
    public Car() {
        System.out.println("bean的构造函数");
    }
    public void setCarName(String carName) {
        System.out.println("属性注入");
        this.carName = carName;
    }
    // 这是BeanFactoryAware接口方法
    public void setBeanFactory(BeanFactory arg0) throws BeansException {
        System.out.println("BeanFactoryAware.setBeanFactory()");
        this.beanFactory = arg0;
    }
    // 这是BeanNameAware接口方法
    public void setBeanName(String arg0) {
        System.out.println("BeanNameAware.setBeanName()");
        this.beanName = arg0;
    }
    // 这是InitializingBean接口方法
    public void afterPropertiesSet() throws Exception {
        System.out.println("InitializingBean.afterPropertiesSet()");
    }
    // 这是DiposibleBean接口方法
    public void destroy() throws Exception {
        System.out.println("DiposibleBean.destory()");
    }
    // 通过<bean>的init-method属性指定的初始化方法
    public void myInit() {
        System.out.println("<bean>的init-method属性指定的初始化方法");
    }
    // 通过<bean>的destroy-method属性指定的初始化方法
    public void myDestory() {
        System.out.println("<bean>的destroy-method属性指定的初始化方法");
    }
}
```
自定义了一个特殊的BeanPostProcessor类型的Bean
```java
public class MyBeanPostProcessor implements BeanPostProcessor {
    public MyBeanPostProcessor() {
        super();
        System.out.println("BeanPostProcessor构造函数");
    }
    public Object postProcessAfterInitialization(Object arg0, String arg1)
            throws BeansException {
        //可以针对指定的Bean做一些操作
        if (arg1.equals("car")) {
            System.out.println("BeanPostProcessor.postProcessAfterInitialization()");
        }
        return arg0;
    }
    public Object postProcessBeforeInitialization(Object arg0, String arg1)
            throws BeansException {
        if (arg1.equals("car")) {
            System.out.println("BeanPostProcessor.postProcessBeforeInitialization()");
        }
        return arg0;
    }
}
```
自定义了一个特殊的BeanFactoryPostProcessor类型的Bean
```java
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    public MyBeanFactoryPostProcessor() {
        super();
        System.out.println("BeanFactoryPostProcessor构造函数");
    }
    public void postProcessBeanFactory(ConfigurableListableBeanFactory arg0)
            throws BeansException {
        System.out.println("BeanFactoryPostProcessor.postProcessBeanFactory()");
    }
}
```
自定义了一个特殊的InstantiationAwareBeanPostProcessor类型的Bean
```java
public class MyInstantiationAwareBeanPostProcessor extends
        InstantiationAwareBeanPostProcessorAdapter {
    public MyInstantiationAwareBeanPostProcessor() {
        super();
        System.out.println("InstantiationAwareBeanPostProcessor的构造函数");
    }
    // 接口方法、实例化Bean之前调用
    @Override
    public Object postProcessBeforeInstantiation(Class beanClass, String beanName) throws BeansException {
        if(beanName.equals("car")) {
            System.out.println("InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation()");
        }
        return null;
    }
    // 接口方法、实例化Bean之后调用
    @Override
    public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
        if(beanName.equals("car")) {
            System.out.println("InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation()");
        }
        return true;
    }
    // 接口方法、设置某个属性时调用
    @Override
    public PropertyValues postProcessPropertyValues(PropertyValues pvs,
                                                    PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {
        if(beanName.equals("car")) {
            System.out.println("InstantiationAwareBeanPostProcessor.postProcessPropertyValues()");
        }
        return pvs;
    }
}
```
将上述1个业务Bean和3个特殊的Bean 配置到xml中
```xml
<bean id="beanPostProcessor" class="test.MyBeanPostProcessor"/>
<bean id="instantiationAwareBeanPostProcessor" class="test.MyInstantiationAwareBeanPostProcessor"/>
<bean id="beanFactoryPostProcessor" class="test.MyBeanFactoryPostProcessor"/>
<bean id="car" class="test.Car" init-method="myInit" destroy-method="myDestory" scope="singleton" p:carName="BMW"/>
```
下面来见证下整个流程
```java
public static void main(String[] args) {
     ClassPathXmlApplicationContext factory = new ClassPathXmlApplicationContext("spring/applicationContext.xml");
     factory.destroy();
}
```
控制台输出如下
```java
BeanFactoryPostProcessor构造函数
BeanFactoryPostProcessor.postProcessBeanFactory()
InstantiationAwareBeanPostProcessor的构造函数
BeanPostProcessor构造函数
InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation()
bean的构造函数
InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation()
InstantiationAwareBeanPostProcessor.postProcessPropertyValues()
属性注入
BeanNameAware.setBeanName()
BeanFactoryAware.setBeanFactory()
BeanPostProcessor.postProcessBeforeInitialization()
InitializingBean.afterPropertiesSet()
<bean>的init-method属性指定的初始化方法
BeanPostProcessor.postProcessAfterInitialization()
DisposableBean.destory()
<bean>的destroy-method属性指定的初始化方法
```
我们可以看到验证与预期相符合。
值得注意的是而在整个Bean的生命周期中除了上文提到的五大核心流程外，还穿插了很多Spring FrameWork提供的扩展点，这也是我们下文讨论的重点。
# <font color=green>Spring Framework扩展点及其应用</font>
<code>日常开发中几乎用不到这些『屠龙技』，但是当阅读Java中间件源码的时候经常会见到它们</code>。
我们接下来会按照下图中的顺序逐一介绍这些扩展点。

<img src='002.jpg'>
## <font color=#0099ff>BeanFactoryPostProcessor#postProcessBeanFactory</font>
背景：由于历史原因，团队有些service项目很臃肿，整个工程中bean的数量有上百个，而大部分单测依赖都是整个工程的xml，导致单测执行时需要很长时间（大部分时间耗费在xml中数百个单例非懒加载的bean的实例化及初始化过程）
解决方法：利用Spring提供的扩展点将xml中的bean设置为懒加载模式，省去了Bean的实例化与初始化时间
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"a.xml","b.xml"})
public abstract class AbstractLightTest {}
```
```java
//a.xml
<bean id="myBeanFactoryPostProcessor" class="com.luyu.LazyBeanFactoryProcessor"/>
```
```java
public class LazyBeanFactoryProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        DefaultListableBeanFactory fac = (DefaultListableBeanFactory) beanFactory;
        Map<String, AbstractBeanDefinition> map = (Map<String, AbstractBeanDefinition>) ReflectionTestUtils.getField(fac, "beanDefinitionMap");
        for (Map.Entry<String, AbstractBeanDefinition> entry : map.entrySet()) {
            //设置为懒加载
            entry.getValue().setLazyInit(true);
        }
    }
}
```
提速非常明显，原来启动测试要1min多钟，懒加载的话只要十几秒就好了
Ps：这个问题建议通过xml环境隔离解决，这里只是提供一个通过Spring扩展点解决的思路
## <font color=#0099ff>InstantiationAwareBeanPostProcessor#postProcessPropertyValues</font>
```xml
Tips：
在容器启动阶段，除了实例化容器，还会针对指定的xml中的bean进行注册
（所谓注册就是以BeanDefination的形式放到容器的BeanDefinationMap中）
流程大体可以概括为：1）解析xml中的元素 2）注册Bean
对于常规的配置项<bean/>  Spring会有默认的解析器对其进行解析并注册
而非常规的配置项比如，<context:component-scan base-package="com.luyu" />
 <aop:aspectj-autoproxy/> Spring也提供了与之对应的特殊解析器
正是通过这些特殊的解析器才使得对应的配置项能够生效
Ps：xml中的每一个配置项都有与之对应的解析器
```
举个例子
```java
@Service
class MyServiceImpl implements MyService {
	@Resource
    private ThirdService thirdService;
}
```
我们知道当使用@Service以及@Resource注解时候，需要在xml中配置如下项
```xml
<context:component-scan base-package="com.luyu" />
```
而针对这个特殊注解的解析器为 ComponentScanBeanDefinitionParser
在这个解析器(parser)的解析方法(parse)中，注册了很多特殊的Bean，下面代码只截取了针对@Resource和@Autowire注解的bean
```java
public BeanDefinition parse(Element element, ParserContext parserContext) {
    //...
    registerComponents(parserContext.getReaderContext(), beanDefinitions, element);
    //...
    return null;
}

public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
			BeanDefinitionRegistry registry, Object source) {
    Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<BeanDefinitionHolder>(4);
    //...
    //@Autowire
    if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
    //@Resource
    if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        //特殊的Bean
        RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
    }
    //...
    return beanDefs;
}
```
我们这里以@Resource为例，看看这个特殊的bean做了啥
```java
public class CommonAnnotationBeanPostProcessor extends InitDestroyAnnotationBeanPostProcessor
		implements InstantiationAwareBeanPostProcessor, BeanFactoryAware, Serializable {
    public PropertyValues postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds, 
    Object bean, String beanName) throws BeansException {
        InjectionMetadata metadata = findResourceMetadata(beanName, bean.getClass());
        try {
        //属性注入
        metadata.inject(bean, beanName, pvs);
        }
        catch (Throwable ex) {
        throw new BeanCreationException(beanName, "Injection of resource dependencies failed", ex);
        }
        return pvs;
    }
}
```
我们看到在postProcessPropertyValues方法中，进行了属性注入
## <font color=#0099ff>invokeAware</font>
这里截取了一段Spring启动过程中的代码，可以看到包含了我们上面3~6四个步骤
```java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
    //③
    invokeAwareMethods(beanName, bean);
    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
    //④
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }
    try {
    //⑤
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    if (mbd == null || !mbd.isSynthetic()) {
    //⑥
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }
    return wrappedBean;
}
```
这里关注下invokeAwareMethods方法
```java
private void invokeAwareMethods(final String beanName, final Object bean) {
    if (bean instanceof Aware) {
        //...
        if (bean instanceof BeanFactoryAware) {
            ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
        }
    }
}
```
举个例子：
```java
@Bean
class BeanFactoryHolder implements BeanFactoryAware{
    private static BeanFactory beanFactory;
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
    }
}
```
实现BeanFactoryAware接口的类，会由容器执行setBeanFactory方法将当前的容器BeanFactory注入到类中
## <font color=#0099ff>BeanPostProcessor#postProcessBeforeInitialization</font>
实现ApplicationContextAware接口的类，会由容器执行setApplicationContext方法将当前的容器applicationContext注入到类中，而该方法调用处位于ApplicationContextAwareProcessor#postProcessBeforeInitialization中
```java
@Bean
class ApplicationContextAwareProcessor implements BeanPostProcessor {
    private final ConfigurableApplicationContext applicationContext;
    public ApplicationContextAwareProcessor(ConfigurableApplicationContext applicationContext) {
      this.applicationContext = applicationContext;
    }
    @Override
    public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
      //...
      invokeAwareInterfaces(bean);
      return bean;
    }
    private void invokeAwareInterfaces(Object bean) {
        if (bean instanceof ApplicationContextAware) {
          ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
        }
    }
}
```
我们看到正是在BeanPostProcessor的postProcessBeforeInitialization中进行了setApplicationContext方法的调用
```java
class ApplicationContextHolder implements ApplicationContextAware{
    private static ApplicationContext applicationContext;
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}
```
## <font color=#0099ff>afterPropertySet() / init-method</font>
目前很多Java中间件都是基本Spring Framework搭建的，而这些中间件经常把入口放到afterPropertySet或者自定义的init中，比如大众点评开源的rpc框架pigeon入口就是配置的bean的init 方法，即ProxyBeanFactory#init（如下）
Ps:https://github.com/dianping/pigeon
```xml
<bean id="remoteService" class="com.luyu.ProxyBeanFactory"
       init-method="init">
     <property name="serviceName" value="xxx" />
     <property name="iface" value="xxx" />
     <property name="serialize" value="xxx" />
     <property name="callMethod" value="xxx" />
     <property name="timeout" value="1000" />
</bean>
```
此外，值得一提的是Pigeon的ProxyBeanFactory类本身是FactoryBean接口的一个实现，而FactoryBean也是Spring提供的一个扩展点，实现该接口的类在作为bean被实例化的时候会返回该接口getObject方法返回的值
```java
public interface FactoryBean<T> {
  	T getObject() throws Exception;
}
```
熟悉aop的同学应该知道，aop底层是通过动态代理实现的，不知道大家有没有思考过动态代理是如何在调用方无感知情况下替换原始对象的？
这里提供Spring解决该问题的两种思路，思路1：通过覆写FactoryBean的getObject方法实现的，该方法返回对应的动态代理对象，该过程对调用方全部透明；思路2见3.6
举个例子（思路1）
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
        ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
        //此时b是ProxyFactoryBean#getObject方法返回的动态代理对象
        MyService b = (MyService)context.getBean("testAop");
        b.test();
    }
}
```
控制台输出
before method advice
i am service test()
after method advice
## <font color=#0099ff>BeanPostProcessor#postProcessAfterInitialization</font>
我们知道当配置了&lt;aop:aspectj-autoproxy/&gt;时候，默认开启aop功能，相应地调用方需要被aop织入的对象也需要替换为动态代理对象，和上文显式地通过FactoryBean偷梁换柱不同，此时通过BeanPostProcessor的postProcessAfterInitialization实现『偷梁换柱』
在3.2节中提到了针对&lt;aop:aspectj-autoproxy/&gt; Spring也提供了特殊的解析器，和其他的解析器类似，在核心的parse方法中注册了特殊的bean，这里是一个BeanPostProcessor类型的bean
```java
class AspectJAutoProxyBeanDefinitionParser implements BeanDefinitionParser {
	@Override
	public BeanDefinition parse(Element element, ParserContext parserContext) {
        //注册特殊的bean
		AopNamespaceUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(parserContext, element);
		extendBeanDefinition(element, parserContext);
		return null;
    }
}
```
我们看到该扩展点的方法的返回值时Object，只要将于当前bean对应的动态代理对象返回即可，该过程对调用方全部透明
```java
public class AnnotationAwareAspectJAutoProxyCreator extends AspectJAwareAdvisorAutoProxyCreator {
		public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (bean != null) {
          Object cacheKey = getCacheKey(bean.getClass(), beanName);
          if (!this.earlyProxyReferences.containsKey(cacheKey)) {
            //如果该类需要被代理，返回动态代理对象；反之，返回原对象
            return wrapIfNecessary(bean, beanName, cacheKey);
          }
        }
        return bean;
	}
}
```
正是利用Spring的这个扩展点实现了动态代理对象的替换
## <font color=#0099ff>destroy() / destroy-method</font>
bean生命周期的最后一个扩展点，该方法用于执行一些bean销毁前的准备工作，比如将当前bean持有的一些资源释放掉

以上です
# <font color=green>资料推荐</font>
官网：https://docs.spring.io/spring/docs/4.3.18.RELEASE/spring-framework-reference/htmlsingle/
《Spring 3.x企业应用与开发实战》
《Spring揭秘》
《Spring技术内幕》
《Spring源码深度解析》
Ps：建议阅读顺序从上至下