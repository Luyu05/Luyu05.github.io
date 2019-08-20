---
title: Spring容器优雅关闭
date: 2019-03-27 14:29:02
tags: Spring
categories: Spring
---
由一个线上报错引发的关于Spring容器优雅关闭的探究
<!-- more -->
# 背景
内部一个用于任务调度的项目，依赖该项目的工程在发布部署时候经常会有这个报错
```java
java.lang.IllegalStateException: BeanFactory not initialized or already closed 
call 'refresh' before accessing beans via the ApplicationContext
```
经排查问题的原因是工程启动后会开启一个daemon线程不断轮询db中的任务并执行，执行过程中会调用ApplicationContext的getBean()方法，而当项目重新部署时会先停掉tomcat，这时候触发了Spring容器的关闭，而后台线程还在调用getBean方法，此时就会提示这个错误。

# 解决方法
## Listener
基于Spring FrameWork的web项目，在web.xml中都会配置一个Spring相关的Listener
```xml
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```
这个Listener提供了很多回调的方法，eg：contextInitialized/contextDestroyed
前者执行了Spring容器的启动过程（refresh()）；后者执行了容器的销毁工作，主要包括销毁所有bean（单例）以及关闭容器(BeanFactory)。在销毁Bean的过程中执行了，每个bean注册的destroy()方法(如果注册了的话)。
解决方法比较简单，配置一个自定义的Listener，放到上面这个Listener的后面，值得一提的是Listener的contextInitialized顺序和contextDestroyed的执行顺序刚好相反，前者和配置的上下顺序一致，后者反之。
但是，本工程是以jar包的形式供业务使用，采用本方案需要业务方自行在web.xml中配置Listener，这对业务的侵入性较大，于是否定了该方案。

## ShutDownHook
在项目工程启动时候注册一个钩子方法，方法中执行调度的关闭
```java
 Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
            @Override
            public void run() {
                stopScheduler();
            }
        }));
```
最开始用的该方法，后来发现可能有问题。
我们发现Tomcat启动时候也会注册一个hook方法，代码如下
```java
public void start() {
//....
if (useShutdownHook) {
if (shutdownHook == null) {
    shutdownHook = new CatalinaShutdownHook();
}
Runtime.getRuntime().addShutdownHook(shutdownHook);
//....
}
```
```java
protected class CatalinaShutdownHook extends Thread {
    public void run() {
        try {
            if (getServer() != null) {
                Catalina.this.stop();
            }
        } catch (Throwable ex) {
            log.error(sm.getString("catalina.shutdownHookFail"), ex);
        } finally {
            // If JULI is used, shut JULI down *after* the server shuts down
            // so log messages aren't lost
            LogManager logManager = LogManager.getLogManager();
            if (logManager instanceof ClassLoaderLogManager) {
                ((ClassLoaderLogManager) logManager).shutdown();
            }
        }
    }
}
```
上面代码中的<code>Catalina.this.stop()</code>会触发tomcat一系列组件的关闭，包括StandardWrapper，而该组件会触发Servlet的关闭，进而被特定的Listener监听到执行Spring的销毁工作。

关于hook  java doc中这样描述的
```java
When the virtual machine begins its shutdown sequence it will start all 
registered shutdown hooks in some unspecified order and let them run concurrently.
```
这意味着hook之间是并行执行的，所以通过该方法并不能完全保证，先关闭任务调度再关闭Spring容器，所以排除了该方法。

## Bean#destroy()
该方案利用了Spring在销毁BeanFactory之前会销毁所有单例的bean这一特性。编写自定义的bean，在destroy方法中实现关闭调度任务的逻辑。
该方法可行，但需要配置一个和业务无关的bean，有些不够优雅。

## ApplicationListener
我们先看看ApplicationContext的关闭方法
```java
protected void doClose() {
        // Publish shutdown event.
        publishEvent(new ContextClosedEvent(this));
        // Destroy all cached singletons in the context's BeanFactory.
        destroyBeans();
        // Close the state of this context itself.
        closeBeanFactory();
        // Let subclasses do some final clean-up if they wish...
        onClose();
        synchronized (this.activeMonitor) {
            this.active = false;
        }
}
```
我们看到在销毁Beans以及关闭BeanFactory之前会发布容器关闭的事件。我们可以在这里去自定义监听器，去实现停止调度的逻辑。
```java
@Service
public class MyListener implements ApplicationListener {
    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        if(event instanceof ContextClosedEvent) {
            stopScheduler();
        }
    }
}
```
最终采用该方案，利用Application内置的监听机制，在容器关闭前执行一些业务的清理工作。