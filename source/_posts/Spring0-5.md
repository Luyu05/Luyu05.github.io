---
title: Spring源码阅读(2)-IOC之bean的实例化
date: 2018-11-15 10:03:17
tags: Spring源码阅读
categories: Spring源码阅读
top : 98
---
本文分析IOC的后半部分-bean的实例化
<!--more-->
# 概览
```java
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
```
上面是refresh方法后续的一些流程，我们先看看<code>registerBeanPostProcessors</code>。
registerBeanPostProcessors方法首先是对BeanFactory中BeanPostProcessor类型bean进行注册，这里首先找到对应的beanName，再调用getBean方法，最后保存到特定的list中。
重点看下<code>finishBeanFactoryInitialization</code>。
# finishBeanFactoryInitialization
从名字就能看出来，这个方法是用来完成对BeanFactory的初始化的。
```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    //...
    // Instantiate all remaining (non-lazy-init) singletons.
    beanFactory.preInstantiateSingletons();
}
```
```java
@Override
public void preInstantiateSingletons() throws BeansException {
    //...
    List<String> beanNames;
    synchronized (this.beanDefinitionMap) {
        beanNames = new ArrayList<String>(this.beanDefinitionNames);
    }
    for (String beanName : beanNames) {
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            if (isFactoryBean(beanName)) {
                final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
                boolean isEagerInit;
                if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                    isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
                        @Override
                        public Boolean run() {
                            return ((SmartFactoryBean<?>) factory).isEagerInit();
                        }
                    }, getAccessControlContext());
                }
                else {
                    isEagerInit = (factory instanceof SmartFactoryBean &&
                            ((SmartFactoryBean<?>) factory).isEagerInit());
                }
                if (isEagerInit) {
                    getBean(beanName);
                }
            }
            else {
                getBean(beanName);
            }
        }
    }
}
```
（1）首先对beanDefinitionMap加锁，找到所有的beanName；
（2）遍历每个beanName找到对应的beanDefination；
（3）对于非抽象&单例&非懒加载的bean执行实例化操作，对于FactoryBean类型的bean我们在关于FactoryBean那篇文章[Spring源码阅读(3)-IOC&AOP的桥梁FactoryBean](https://luyu05.github.io/2018/11/16/Spring0-1/)中已经分析过了，我们本文只关心普通类型的bean。
```java
@Override
public Object getBean(String name) throws BeansException {
    return doGetBean(name, null, null, false);
}
```
```java
protected <T> T doGetBean(
            final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
            Object bean;
			throws BeansException {
                //1.根据beanName获取对应的bean（单例只会创建一次）
                Object sharedInstance = getSingleton(beanName);   
                if (sharedInstance != null && args == null) {
                    bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
                }
                else{
                    final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
                    checkMergedBeanDefinition(mbd, beanName, args); 
                    String[] dependsOn = mbd.getDependsOn();
                    //2.优先实例化dependOn的bean
                    if (dependsOn != null) {
                        for (String dependsOnBean : dependsOn) {
                            if (isDependent(beanName, dependsOnBean)) {
                                throw new BeanCreationException("Circular depends-on relationship between '" +
                                        beanName + "' and '" + dependsOnBean + "'");
                            }
                            registerDependentBean(dependsOnBean, beanName);
                            getBean(dependsOnBean);
                        }
                    }
                    if (mbd.isSingleton()) {
                        sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
                            @Override
                            public Object getObject() throws BeansException {
                                try {
                                    //3.实例化bean
                                    return createBean(beanName, mbd, args);
                                }
                            }
                        });
                        bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
                    }
                }
                //...
                return (T) bean;
            }
```
即使删除了一些边角料代码还是相对冗长，我们将代码分成3个部分
（1）检测单例类型的bean是否已经被实例化，如果有直接return
（2）否则，优先实例化depend-on属性指定的bean
（3）接下来开始真正实例化当前的bean，调用的方法是createbean

在深入createBean之前我们先看看下面这个入口方法
```java
public Object getSingleton(String beanName, ObjectFactory singletonFactory) {
    //...
    beforeSingletonCreation(beanName);
    singletonObject = singletonFactory.getObject();
    afterSingletonCreation(beanName);
    //...
}
```
在该方法中执行bean的实例化前，<code>beforeSingletonCreation(beanName)</code>会将当前beanName存放到一个创建ing的bean的set中，如果对set的add操作失败则表示产生了循环依赖直接抛出错误；而<code>afterSingletonCreation(beanName)</code>方法会将创建ing的beanName从set中移除。
```java
@Override
protected Object createBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
        throws BeanCreationException {
    //...
    // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
    Object bean = resolveBeforeInstantiation(beanName, mbd);
    if (bean != null) {
        return bean;
    }
    //...
    Object beanInstance = doCreateBean(beanName, mbd, args);
    return beanInstance;
}
```
这里我们会遇到bean实例化阶段的第1个扩展点，InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation方法。该方法在resolveBeforeInstantiation方法中被回调。
接下来根据beanDefination创建bean的实例。注意接下来这个方法是及其重要的一个方法，涉及bean的实例化、注入以及初始化
```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
    //...
    //实例化
    BeanWrapper instanceWrapper = createBeanInstance(beanName, mbd, args);
    final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
    Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);
    //...
    Object exposedObject = bean;
    try {
        //注入
        populateBean(beanName, mbd, instanceWrapper);
        if (exposedObject != null) {
            //初始化
            exposedObject = initializeBean(beanName, exposedObject, mbd);
        }
    }
    //...
    return exposedObject;
}
```
我们逐一去分析这三个步骤
## 实例化
我们只看最普通的情况，其他的代码全部省略。
```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
    //...
    return instantiateBean(beanName, mbd);
}
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
    //...
    beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
    BeanWrapper bw = new BeanWrapperImpl(beanInstance);
    initBeanWrapper(bw);
    return bw;
}
```
我们可以看到在bean实例化的时候，采用的是策略模式，默认选择的是CglibSubclassingInstantiationStrategy，也就是说cglib的模式。我们跟进方法看到，instantiate这个方法是位于父类的（SimpleInstantiationStrategy），该方法首先判断是否有method的重写（look-up等）如果没有的话，那么默认采用反射的方式实例化bean；反之采用子类cglib的方式实例化bean。接下来将Bean包装成BeanWrapper供执行注入操作。
## 注入
```java
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
    PropertyValues pvs = mbd.getPropertyValues();
    //...
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                    //...
                }
            }
        }
    }

    if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
            mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

        // Add property values based on autowire by name if applicable.
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mbd, bw, newPvs);
        }

        // Add property values based on autowire by type if applicable.
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
            autowireByType(beanName, mbd, bw, newPvs);
        }

        pvs = newPvs;
    }

    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

    if (hasInstAwareBpps || needsDepCheck) {
        PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        if (hasInstAwareBpps) {
            for (BeanPostProcessor bp : getBeanPostProcessors()) {
                if (bp instanceof InstantiationAwareBeanPostProcessor) {
                    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                    pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                    if (pvs == null) {
                        return;
                    }
                }
            }
        }
        if (needsDepCheck) {
            checkDependencies(beanName, mbd, filteredPds, pvs);
        }
    }

    applyPropertyValues(beanName, mbd, bw, pvs);
}
```
这个方法也是很重要的，
（1）这里首先引出了实例化阶段Spring扩展点的第2个回调InstantiationAwareBeanPostProcessor的postProcessAfterInstantiation方法；
（2）接着如果bean的定义上有autowire_by_name或autowire_by_type属性，那么除了用户显式配置的property外，还要自动去寻找bean的类中『隐藏』的一些property（只针对bean），找到之后此时并未执行注入，而是简单地把propertyName和对应的bean放到最后一起注入使用的容器MutablePropertyValues pvs中。
（3）接着来到了bean实例化阶段Spring扩展点的第3个回调InstantiationAwareBeanPostProcessor的postProcessPropertyValues方法，Ps对于@Resource,@Autowire注解都是在这里生效的(会提前执行inject操作)
（4）最后，执行真正的注入操作。最终利用反射将属性注入到Bean中。

之前对最后一步的理解有问题，之前认为Bean内注入的Bean是一份深拷贝。刚才写了代码测试了下，对于单例的bean整个容器有且只有一份。
## 初始化
```java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
    //...
    invokeAwareMethods(beanName, bean);
    //...
    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }
    //...
    try {
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    //...
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }
    return wrappedBean;
}
```
我们看到这个方法也包含了很多的内容
（1）如果当前的bean是和BeanFactory相关的Aware接口，那么调用setXxx方法，还记得在容器启动阶段ignoreDependencyInterface的那几个类吗？就是在这里调用对应的方法的。（BeanNameAware,BeanClassLoaderAware,BeanFactoryAware）
<font color=red>这里之前理解的有问题，这里要先判断当前的bean是否是指定的那几种类型，如果是则调用setXxx方法，而不是无脑调用

</font>

（2）接下来是Spring扩展点的第4个回调BeanPostProcessor的postProcessBeforeInitialization方法，在refresh方法中的第三个步骤prepareBeanFactory(beanFactory)中将ApplicationContextAwareProcessor注入到了BeanFactory中，而该类就是在这里执行对应的postProcessBeforeInitialization方法。这个方法调用了ApplicationContext特色的各种XxxAware的类的setXxx方法，比如ApplicationContextAware,ApplicationEventPublisherAware,ResourceLoaderAware
<font color=red>这里也是一样，只有当前bean是对应类型的接口，才调用对用的setXxx方法；和上面invokeAware方法不同的是，这里仍然要遍历容器内的所有BeanPostProcessor并执行对应的postProcessBeforeInitialization方法（类似渔网，有些鱼能过去，有些鱼会被捉住）

</font>

（3）调用bean的初始化方法
```java
protected void invokeInitMethods(String beanName, final Object bean, RootBeanDefinition mbd)
			throws Throwable {
    boolean isInitializingBean = (bean instanceof InitializingBean);
    if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
        //...
        ((InitializingBean) bean).afterPropertiesSet();
    }

    if (mbd != null) {
        String initMethodName = mbd.getInitMethodName();
        if (initMethodName != null && !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
                !mbd.isExternallyManagedInitMethod(initMethodName)) {
            invokeCustomInitMethod(beanName, bean, mbd);
        }
    }
}
```
如果bean的类型是InitializingBean，那么优先调用afterPropertiesSet方法；随后调用自定义的init方法。
（4）接下来是Spring扩展点的第5个回调BeanPostProcessor的postProcessAfterInitialization方法

至此，Bean已经实例化、注入、初始化完成。

---
最后祭出一张图。
<img src='001.jpg'>
<img src='002.jpg'>

---