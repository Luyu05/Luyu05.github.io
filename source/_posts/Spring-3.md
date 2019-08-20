---
title: Spring官方文档拾遗
date: 2018-08-11 14:49:07
tags: Spring
categories: Spring
---
最近在读Spring官方文档，顺路对很多不清晰的地方进行了梳理
1.ApplicationContext
<!-- more -->
### <font color=red size=5>关于ApplicationContext</font>
都说ApplicationContext是高级版的BeanFactory，更加框架化
相比BeanFactory主要多了如下几个功能
<code>1）通过MessageResource接口提供了国际化支持</code>
```xml
# in exception.properties
exception = "又写bug了吧"
# in exception_en_GB.properties
exception = "find bug agian"
```
```xml
<bean id="messageSource" class="org.springframework.context.support.ResourceBundleMessageSource">
        <property name="basename" value="exception"/>
    </bean>
```
```java
public static void main(String[] args) {
    MessageSource resources = new ClassPathXmlApplicationContext("spring/applicationContext.xml");
    String message0 = resources.getMessage("exception",null,"又写bug了把", Locale.UK);
    String message1 = resources.getMessage("exception",null,"又写bug了把", Locale.CHINA);
    System.out.println(message0);
    System.out.println(message1);
}
```
输出为：
find bug agian
又写bug了吧
Ps原以为高大上的国际化其实很简单，只不过类似屠龙技不常用罢了
<code>2）通过ResourceLoader接口提供了资源访问功能</code>
一个Resource本质上是一个功能更丰富的JDK类java.net.URL版本，实际上，Resource的实现包装了一个java.net.URL的实例
 ApplicationContext可以方便地加载资源
<code>3）通过ApplicationEventPublisher接口提供了事件发布功能</code>
```java
public class BlackListEvent extends ApplicationEvent {

    private final String address;
    private final String test;

    public BlackListEvent(Object source, String address, String test) {
        super(source);
        this.address = address;
        this.test = test;
    }

    // accessor and other methods...
}
```
```java
public class EmailService implements ApplicationEventPublisherAware {

    private List<String> blackList;
    private ApplicationEventPublisher publisher;

    public void setBlackList(List<String> blackList) {
        this.blackList = blackList;
    }
    //这里注入的实际上是ApplicationContext本身
    public void setApplicationEventPublisher(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    public void sendEmail(String address, String text) {
        if (blackList.contains(address)) {
            BlackListEvent event = new BlackListEvent(this, address, text);
            publisher.publishEvent(event);
            return;
        }
        // send email...
    }
}
```
```java
public class BlackListNotifier implements ApplicationListener<BlackListEvent> {

    private String notificationAddress;

    public void setNotificationAddress(String notificationAddress) {
        this.notificationAddress = notificationAddress;
    }

    public void onApplicationEvent(BlackListEvent event) {
        // notify appropriate parties via notificationAddress...
    }
}
```
```xml
<bean id="emailService" class="example.EmailService">
    <property name="blackList">
        <list>
            <value>known.spammer@example.org</value>
            <value>known.hacker@example.org</value>
            <value>john.doe@example.org</value>
        </list>
    </property>
</bean>

<bean id="blackListNotifier" class="example.BlackListNotifier">
    <property name="notificationAddress" value="blacklist@example.org"/>
</bean>
```
当调用emailService的sendEmail()时，如果有email在黑名单中，这时候会会发布BlackListEvent，而BlackListNotifier会对这个事件进行监听，进行一些处理工作
Ps一个事件可以有多个监听者，多个监听者如果同时被触发，会同步地执行，这里的同步指的是一个一个的执行，而且publisher.publishEvent(event);方法也会阻塞知道监听者逐一执行完自己的逻辑
### <font color=red size=5>关于BeanPostProcessor</font>
1.BeanPostProcessor处理容器内所有符合条件的BeanDefinition Ps这里的符合条件与否在覆写的方法中自行判断if(beanName==xxx)

2.A bean post-processor typically checks for callback interfaces or may wrap a bean with a proxy. Some Spring AOP infrastructure classes are implemented as bean post-processors in order to provide proxy-wrapping logic.
AOP就是通过BeanPostProcessor实现偷梁换柱的

3.Classes that implement the BeanPostProcessor interface are special and are treated differently by the container. All BeanPostProcessors and beans that they reference directly are instantiated on startup, as part of the special startup phase of the ApplicationContext
实现该接口的bean是很特殊的，它以及它所依赖的bean需要优先启动，可以将其视为容器启动的一个阶段。
而且实现了该接口的bean（或者被依赖的beans）无法对其进行动态代理，因为动态代理本身就是通过BeanPostProcessor实现的，你可以理解为"大力士举不起自己"

### <font color=red size=5>关于BeanFactoryPostProcessor</font>
1.PropertyPlaceholderConfigurer
```xml
<bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
    <property name="locations" value="classpath:com/foo/jdbc.properties"/>
</bean>
or
<context:property-placeholder location="classpath:com/foo/jdbc.properties"/>

<bean id="dataSource" destroy-method="close"
        class="org.apache.commons.dbcp.BasicDataSource">
    <property name="driverClassName" value="${jdbc.driverClassName}"/>
    <property name="url" value="${jdbc.url}"/>
    <property name="username" value="${jdbc.username}"/>
    <property name="password" value="${jdbc.password}"/>
</bean>
```
``` properties
# jdbc.properties
jdbc.driverClassName=org.hsqldb.jdbcDriver
jdbc.url=jdbc:hsqldb:hsql://production:9002
jdbc.username=sa
jdbc.password=root
```
在运行时，占位符内的内容会被替换为属性文件中的匹配的value，这个工作就是由PlaceholderConfiguer完成的，不难看出这个类应该实现了BeanFactoryPostProcessor