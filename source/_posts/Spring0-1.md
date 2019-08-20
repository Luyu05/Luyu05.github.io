---
title: Spring源码阅读(3)-IOC&AOP的桥梁FactoryBean
date: 2018-11-16 09:49:09
tags: Spring源码阅读
categories: Spring源码阅读
top : 97
---
IOC&AOP桥梁FactoryBean的实现
<!-- more -->
分析的入口点是refresh方法，从finishBeanFactoryInitialization(beanFactory)方法开始
```java
public void refresh() throws BeansException, IllegalStateException {
    //...
    finishBeanFactoryInitialization(beanFactory);
    //...
}

protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    //...
    beanFactory.preInstantiateSingletons();
}
```
重点看preInstantiateSingletons方法，注意本文不分析普通的Bean，本文的关注重点是FactoryBean
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
我们按照顺序来看，这个方法其实已经是IOC进行bean实例化的入口了，这里首先对BeanDefinationMap加锁，找到所有的beanNames，接着去遍历beanNames，针对每个非抽象&单例&非懒加载的bean执行实例化工作。
代码先来到<code>isFactoryBean(beanName)</code>
```java
@Override
public boolean isFactoryBean(String name) throws NoSuchBeanDefinitionException {
    //...
    return isFactoryBean(beanName, getMergedLocalBeanDefinition(beanName));
}

protected boolean isFactoryBean(String beanName, RootBeanDefinition mbd) {
    Class<?> beanType = predictBeanType(beanName, mbd, FactoryBean.class);
    return (beanType != null && FactoryBean.class.isAssignableFrom(beanType));
}
```
可以看到通过beanName以及对应的beanDefination来判断当前这个bean是否被定义为FactoryBean类型。
由于我们分析的就是FactoryBean所以代码进入if代码块。
```java
if (isFactoryBean(beanName)) {
    //如果是工厂bean 那么为其添加前缀&
    final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
}
```
一路跟踪代码直到下面，此时 name = & + beanName
```java
protected <T> T doGetBean(
			final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
			throws BeansException {
    //以&开头的name - & = beanName
    final String beanName = transformedBeanName(name);
    //...
    if (mbd.isSingleton()) {
        sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
            @Override
            public Object getObject() throws BeansException {
                try {
                    return createBean(beanName, mbd, args);
                }
                catch (BeansException ex) {
                    destroySingleton(beanName);
                    throw ex;
                }
            }
        });
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
    }
    //...
}
```
上面这段代码，首先这里的<code>transformedBeanName</code>，会将&+beanName去除掉&，去掉之后对真正的BeanFactory执行实例化操作，接着调用getSingleton，该方法是任何一个单例bean实例化都要经历的一个方法，这个方法很简单就是调用匿名类的getObject()方法<code>createBean</code>并获得对应的bean的实例
<font color=red>此时的sharedInstance是一个真正的BeanFactory实例!!!</font>，这意味着容器中该bean的缓存一直都是一个BeanFactory实例。并且我们发现无论是从缓存中(Object sharedInstance = getSingleton(beanName))获取到bean还是自己实例化bean，之后都会调用<code>getObjectForBeanInstance</code>这个方法，而该方法就是FactoryBean的核心。


<font color=red>在该方法中对于BeanFactory类型的bean，如果是&开头那么最终返回就是BeanFactory反之返回覆写的getObject方法的返回值，注意这里将name也一起传递到了方法中（用来标识是否&开头）</font>
```java
protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, RootBeanDefinition mbd) {

    //...
    //1.是FactoryBean并且name以&开头，直接return 2.非FactoryBean直接return
    if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
        return beanInstance;
    }

    Object object = null;
    if (mbd == null) {
        object = getCachedObjectForFactoryBean(beanName);
    }
    if (object == null) {
        FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
        if (mbd == null && containsBeanDefinition(beanName)) {
            mbd = getMergedLocalBeanDefinition(beanName);
        }
        boolean synthetic = (mbd != null && mbd.isSynthetic());
        object = getObjectFromFactoryBean(factory, beanName, !synthetic);
    }
    return object;
}
```
第一个if判断是说如果当前这个bean不是FactoryBean<font color=red>或者当前的name带有&</font>那就直接return当前的beanInstance。如果是FactoryBean且前缀无&那么代码继续向下走，调用<code>getObjectFromFactoryBean(factory, beanName, !synthetic)</code>
```java
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
    //...
    return doGetObjectFromFactoryBean(factory, beanName, shouldPostProcess);
}

private Object doGetObjectFromFactoryBean(
			final FactoryBean<?> factory, final String beanName, final boolean shouldPostProcess)
			throws BeanCreationException {
        
        Object object;
        //...
        object = factory.getObject();
		//...
		return object;
	}
```
到这里就真相大白了，调用了factoryBean这个bean的getObject方法。

Tips：
application启动的时候，放到<code>Map<String, Object> singletonObjects</code>中的bean是FactoryBean而不是getObject()返回的对象，当调用getBean()时候如果入参前缀为&此时会直接return map中存放的对象(即使有&也会被摘除)；反之，如果无前缀，这时候会return FactoryBean.getObject()方法。下面这段代码是关键的关键。
```java
protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, RootBeanDefinition mbd) {
   //...
    if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
        return beanInstance;
    }
    //...
}
```
就是通过<code>BeanFactoryUtils.isFactoryDereference</code>来决定是返回FactoryBean还是其getObject()方法的返回值。

在xml中是无法配置&开头的名字的bean的，也就是说在beanDefinationMap中FactoryBean类型的bean的beanName是没有&前缀的；前文所说的& + beanName是在人为调用getBean()时候完成的

再梳理下整个流程，当我们执行getBean("&MyFactory")时候，name=&MyFactory beanName=MyFactory
根据MyFactory获取到了缓存中的FactoryBean类型的实例，接着调用getObjectForBeanInstance方法，而该方法的入参除了beanName还有name，根据name是否有前缀决定下一步的行为，如果有那么直接return缓存中的FactoryBean实例，反之返回getObject()的返回值。

---
小结：
每次调用getBean，在获取到bean的实例之后（无论是从缓冲中拿还是自己实例化），都会执行getObjectForBeanInstance方法，该方法对于普通的bean以及(FactoryBean类型并且名字是&开头)的bean会直接return；反之，也就是说对于FactoryBean类型且bean的名字开头不是&的bean会调用其getObject方法。

<code>容器中只有FactoryBean类型的bean</code>

最后，写文章一定要多问自己几个为什么，就像文末说的那行代码决定是返回factoryBean还是其getObject()方法的返回值，这个如果不仔细思考可能就遗漏了，追代码追到getObject就结束了。本文分析了factoryBean，下篇文章去看看和factoryBean大有渊源的AOP。

---
