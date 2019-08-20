---
title: Spring-IOC骨架
date: 2018-09-01 14:49:08
tags: Spring
categories: Spring
---
试着把自己看源码对一些理解写下来
从相对宏观的角度去分析整个过程码
<!-- more -->
还是从经典的例子开始入手
```java
private void applicationEntrance(){
    ApplicationContext context = new FileSystemXmlApplicationContext("classpath:applicationContext.xml");
    context.getBean("account");
}

public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent){
    super(parent);
    setConfigLocations(configLocations);
    if (refresh) {
        refresh();
    }
}
```
要开车了，坐稳了，我们逐步进行分析<br>
首先调用new FileSystemXmlApplicationContext(xml)方法，最后发现方法来到了refresh()方法，这个可以一个大名鼎鼎的方法，凡是看过或者尝试看过spring源码对同学应该都对这个方法不陌生，这是一个标准的模板方法模式，由父类指定整个代码的执行逻辑，由子类去决定每个方法的具体实现。额，还是上代码吧重要的地方会写注释
```java
refresh()
    //applicaitonContext其实就是高配版本的Beanfactory所以很多工作还是要BeanFactory来做
    //去初始化beanFactory,包括(1）读取xml (2）将xml中的bean转为beanDefination (3）并注册到容器中
     1)obtainFreshBeanFactory()
        refreshBeanFactory()
            //创建BeanFactory
            createBeanFactory()
            loadBeanDefinitions(beanFactory)
                loadBeanDefinitions(demo.xml)
                    1.读取配置文件
                        //将配置文件抽象为resource
                        resourceLoader.load(String confLocation)
                        容器读取resource
                    2.将配置文件中的bean转为spring中的格式BeanDefination并注册到容器中
                        //将resource转为document
                        Document doc = doLoadDocument(inputSource, resource)
                            //获取xml的验证模式
                            getValidationModeForResource(resource)
                                判断是DTD or XSD（根据是否具有DOCTYPE）
                            //对xml进行解析获取document（根据验证模式）
                            this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,getValidationModeForResource(resource),isNamespaceAware());
                        //注册beanDefination  
                        registerBeanDefinitions(doc, resource)  
                            将document转为beanDefination
                                根据element进行构造
                            对beanDefination进行校验
                            //beanName-BeanDefinition存放在Map<String, BeanDefinition> beanDefinitionMap
                            this.beanDefinitionMap.put(beanName, beanDefinition);
    //大佬在这里被调用，唯一可以插手容器启动阶段的一个bean，可以对beanDefinition进行修改
    //对该类型bean会提前执行BeanFactory.getBean()
    2)invokeBeanFactoryPostProcessors()
    //注册BeanPostProcessor
    //真正干活的还是beanFactory = =!
    3)registerBeanPostProcessors(beanFactory)
    //实例化所有单例非懒加载beans
    4)finishBeanFactoryInitialization(beanFactory) 
        遍历 beanDefinitionMap找到单例非抽象非懒加载的bean调用BeanFactory的getBean(beanName)   
            单例bean先从缓冲中读取，如果没有才去创建；反之直接返回
            获取depend-on到bean先去对这些bean调用getBean()
            //以单例bean为例子
            createBean()
                //这里要和postProcessBeforeInitialization区分
                InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation,如果这里返回的bean非空那么直接return（短路了）
                //核心类 AbstractAutowireCapableBeanFactory
                //核心方法，创建bean->填充->初始化
                doCreateBean(beanName,beanDefination,args)
                    //创建bean的实例并用BeanWrapper包裹
                    1、createBeanInstance(beanName, mbd, args) 
                        instantiateBean(beanName, mbd)
                            //策略模式决定如何实例化bean cglib or reflect
                            //反射策略居然是cglib策略的父类= =！
                            //对于look-up之类的功能需要cglib来支持
                            getInstantiationStrategy().instantiate(mbd, beanName, parent)
                            BeanWrapper bw = new BeanWrapperImpl(beanInstance);
                            initBeanWrapper(bw);
                            return bw
                    //对BeanWrapper进行属性填充
                    2、populateBean(beanName, mbd, instanceWrapper)
                        先去从BeanDefination中获取property属性放到pv中
                        InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation
                        代码来到了byName byType
                            //注意beanDefination中的pv仅仅表示配置的属性，而非全属性
                            以byName为例，先去遍历已经实例化的beanWrapper中的set/is开头的方法
                            再去判断这个属性是否是一个bean，如果是将对应的bean放到pv中稍后一起填充
                        InstantiationAwareBeanPostProcessor#postProcessPropertyValues
                        //属性填充
                        applyPropertyValues(beanName, mbd, bw, pvs)
                    //初始化bean
                    3、initializeBean(beanName, exposedObject, mbd)
                        //这里对实现各种aware结尾接口的相关属性进行注入
                        invokeAwareMethods(beanName, bean)
                        //beanPostProcessor的前置方法
                        applyBeanPostProcessorsBeforeInitialization
                        //调用bean的初始化方法
                        invokeInitMethods(beanName, wrappedBean, mbd)
                            先执行InitializingBean类型的bean都afterPropertiesSet()方法
                            再执行<bean init-method>
                        //beanPostProcessor的后置方法
                        applyBeanPostProcessorsAfterInitialization
                    //注册DisposableBean和<destroy-bean>
                    4、registerDisposableBeanIfNecessary
```

<code>AbstractAutowireCapableBeanFactory这个类是很重要的一个类</code>