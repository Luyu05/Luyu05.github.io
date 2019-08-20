---
title: Spring源码阅读(1)-IOC之容器启动阶段
date: 2018-11-14 20:07:19
tags: Spring源码阅读
categories: Spring源码阅读
top : 99
---
契机：之前写了很多宏观分析Spring的文章，很多细节都不是很清楚，所以想开一个源码阅读的专栏，希望能够坚持下去。
本来不打算写关于IOC的文章的，后来看了下之前写的文章，都相对浅薄没有深入源码分析，于是做本文。
IOC部分打算分为两篇文章来分别讲解，本文是该系列之一『容器启动阶段』
<!-- more -->
# 入口点
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
```java
public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
        throws BeansException {
    super(parent);
    setConfigLocations(configLocations);
    if (refresh) {
        refresh();
    }
}
```


上面这个例子我们看到是在main方法中直接触发的Spring，而我们日常Web开发中是在哪儿&由谁来执行上面这部分代码的呢？
我们进入Java Web项目的web.xml中会发现这样一段代码
```xml
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```
而上述xml配置是基于Spring的Web项目必备的，我们看看这个ContextLoaderListener干了啥？
```java
public class ContextLoaderListener extends ContextLoader implements ServletContextListener {
   @Override
	public void contextInitialized(ServletContextEvent event) {
		initWebApplicationContext(event.getServletContext());
	}
	/**
	 * Close the root web application context.
	 */
	@Override
	public void contextDestroyed(ServletContextEvent event) {
		closeWebApplicationContext(event.getServletContext());
		ContextCleanupListener.cleanupAttributes(event.getServletContext());
	}
}

```
该类实现了ServletContextListener接口，并且覆写了contextInitialized方法和contextDestroyed方法。前者会在容器启动阶段被调用，且在任何servlet or filter执行之前；后者会在容器销毁阶段被调用，且在全部servlet and filter被销毁后执行。
在容器启动阶段，会执行ContextLoaderListener#contextInitialized方法，而该方法会调用下面这段逻辑
```java
// wac is instance of ConfigurableWebApplicationContext
wac.refresh();
```

我们发现最终也执行到了大名鼎鼎的refresh方法中

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);

            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            initMessageSource();

            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            onRefresh();

            // Check for listener beans and register them.
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            finishRefresh();
        }

        catch (BeansException ex) {
            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }
    }
}
```
本文主要分析的部分是<code>invokeBeanFactoryPostProcessors(beanFactory)</code>和它之前的部分
1）准备重启(refresh)
2）<font color=red>获得BeanFactory</font>(这里获得的BeanFactory已经持有注册好的BeanDefination了)
3）准备BeanFactory
4）对获得到的BeanFactory进行后处理
5）<font color=red>调用BeanFactoryPostProcessors</font>

本文的重点在2，3，5
# obtainFreshBeanFactory()
## refreshBeanFactory()
```java
@Override
protected final void refreshBeanFactory() throws BeansException {
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    try {
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        beanFactory.setSerializationId(getId());
        customizeBeanFactory(beanFactory);
        loadBeanDefinitions(beanFactory);
        synchronized (this.beanFactoryMonitor) {
            this.beanFactory = beanFactory;
        }
    }
    catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
    }
}
```
首先，判断当前是否已经有BeanFactory了，如果有那么要把beans全部销毁并关闭BeanFactory。
接着，创建BeanFactory
最后，解析BeanDefination并注册到容器中

我们重点看后两个步骤
### createBeanFactory()
```java
protected DefaultListableBeanFactory createBeanFactory() {
    return new DefaultListableBeanFactory(getInternalParentBeanFactory());
}
public AbstractAutowireCapableBeanFactory() {
    //this constructor just return instance
    super();
    ignoreDependencyInterface(BeanNameAware.class);
    ignoreDependencyInterface(BeanFactoryAware.class);
    ignoreDependencyInterface(BeanClassLoaderAware.class);
}
```
代码最后追到上面那里，首先调用父类的构造函数（空实现），接着将上面三个XXXAware类放到Set中，在该Set中的类的Xxx属性会由容器注入，<font color=red>且在依赖检查的时候会被忽略。</font>依赖检查已经被Spring抛弃了。所以不必过于纠结这里

延伸知识：
In Spring,you can use dependency checking feature to make sure the required properties have been set or injected.
none – No dependency checking.
simple – If any properties of primitive type (int, long,double…) and collection types (map, list..) have not been set, UnsatisfiedDependencyException will be thrown.
objects – If any properties of object type have not been set, UnsatisfiedDependencyException will be thrown.
all – If any properties of any type have not been set, an UnsatisfiedDependencyException
will be thrown.

Tips：
在Spring中名为xxxAware的bean，一般都会调用setXxx()以实现目标的注入。
### loadBeanDefinitions(beanFactory)
```java
@Override
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
    // Create a new XmlBeanDefinitionReader for the given BeanFactory.
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

    // Configure the bean definition reader with this context's
    // resource loading environment.
    beanDefinitionReader.setEnvironment(this.getEnvironment());
    beanDefinitionReader.setResourceLoader(this);
    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

    // Allow a subclass to provide custom initialization of the reader,
    // then proceed with actually loading the bean definitions.
    initBeanDefinitionReader(beanDefinitionReader);
    loadBeanDefinitions(beanDefinitionReader);
}
```
这里首先为BeanFactory新建了一个BeanDefinationReader（此时Reader持有BeanFactory）；
接着填充Reader的一些属性；
最后调用<code>loadBeanDefinitions(beanDefinitionReader)</code>，构建document、解析并注册BeanDefination
```java
//XmlBeanDefinationReader.class
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
    //...
    Document doc = doLoadDocument(inputSource, resource);
    return registerBeanDefinitions(doc, resource);
    //...
}
```
#### doLoadDocument(inputSource, resource)
根据输入流封装的inputSource以及resource去构建document，这里具体的代码就不去分析了，重点看这个。
Ps：以后有document解析的需求可以参考这里的代码
#### registerBeanDefinitions(doc, resource)
```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
    documentReader.setEnvironment(this.getEnvironment());
    int countBefore = getRegistry().getBeanDefinitionCount();
    documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
    return getRegistry().getBeanDefinitionCount() - countBefore;
}

@Override
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
    this.readerContext = readerContext;
    logger.debug("Loading bean definitions");
    Element root = doc.getDocumentElement();
    doRegisterBeanDefinitions(root);
}

protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate){
    if (delegate.isDefaultNamespace(root)) {
        NodeList nl = root.getChildNodes();
        for (int i = 0; i < nl.getLength(); i++) {
            Node node = nl.item(i);
            if (node instanceof Element) {
                Element ele = (Element) node;
                //常规bean的解析操作
                if (delegate.isDefaultNamespace(ele)) {
                    parseDefaultElement(ele, delegate);
                }
                //自定义注解的解析，eg:<aop>
                else {
                    delegate.parseCustomElement(ele);
                }
            }
        }
    }
    else {
        delegate.parseCustomElement(root);
    }
}
```
我们层层递进追到了上面这个方法中，我们在分析自动化aop的时候发现就是在这里的<code>delegate.parseCustomElement(ele)</code>对<code>&lt;aop:aspectj-autoproxy/&gt;</code>进行解析的。我们先来看默认的情况
```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
    if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
        importBeanDefinitionResource(ele);
    }
    else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
        processAliasRegistration(ele);
    }
    else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
        processBeanDefinition(ele, delegate);
    }
    else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
        // recurse
        doRegisterBeanDefinitions(ele);
    }
}
```
我们直接进入到<code>processBeanDefinition(ele, delegate)</code>
```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if (bdHolder != null) {
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
            // Register the final decorated instance.
            BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
        }
        catch (BeanDefinitionStoreException ex) {
            getReaderContext().error("Failed to register bean definition with name '" +
                    bdHolder.getBeanName() + "'", ele, ex);
        }
        // Send registration event.
        getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
}
```
这里主要分为了两个步骤，一个是BeanDefination的真正解析；一个是BeanDefination的注册。具体的解析步骤这里就不进行分析了。而BeanDefination的注册其实就是将beanName和beanDefination的映射关系保存在map中，<code>this.beanDefinitionMap.put(beanName, beanDefinition);</code>
Ps:显然beanDefinitionMap是被BeanFactory持有的
## getBeanFactory()
这个步骤就是简单地将上一步骤中实例化以及初始化过得beanFactory返回。<code>return this.beanFactory;</code>

# prepareBeanFactory
```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // Tell the internal bean factory to use the context's class loader etc.
    beanFactory.setBeanClassLoader(getClassLoader());
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver());
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // Configure the bean factory with context callbacks.
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    //...
}
```
这里我们看到添加了一个BeanPostProcessor-ApplicationContextAwareProcessor；并且同样忽略了很多以Aware结尾的接口。关于ApplicationContextAwareProcessor我们会在Bean的实例化阶段再去分析。
# invokeBeanFactoryPostProcessors(beanFactory)
```java
public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
    //...
    List<BeanFactoryPostProcessor> regularPostProcessors = new LinkedList<BeanFactoryPostProcessor>();
    for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
        regularPostProcessors.add(postProcessor);
    }
    invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
}

private static void invokeBeanFactoryPostProcessors(
        Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory) {
    for (BeanFactoryPostProcessor postProcessor : postProcessors) {
        postProcessor.postProcessBeanFactory(beanFactory);
    }
}
```
对每个普通的BeanFactoryProcessor都调用了postProcessBeanFactory方法。
至此容器的启动阶段结束。

---
小结：
1）如果当前存在BeanFactory那么销毁所有Beans并关闭BeanFactory
2）实例化BeanFactory，与此同时指定了一些不会被依赖检查的类型的bean
3）将xml解析为Document（xml->inputsource&resource）
4）利用BeanDefinitionParserDelegate对Document进行解析，解析的结果是BeanDefination（对特殊注解比如aspectJ aop的解析也在这里）
5）对解析出来的BeanDefination进行注册（其实就是放到了一个ConcurrentHashMap中，该map被BeanFactory持有）
6）回调BeanFactoryPostProcessor类型的bean的postProcessBeanFactory方法

以上です

---