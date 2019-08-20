---
title: Spring-MVC
date: 2018-07-14 17:05:42
tags: Spring
categories: Spring
---
Spring-MVC框架阅读
<!-- more -->
## 关于ServletContext
ServletContext这个名字不太贴切，不如叫AppContext,对整个web应用可用，而不是只针对一个servlet。
``` xml
<web-app>
    <context-param>
        <param-name>foo</param-name>
        <param-value>bar</param-value>
    </context-param>
</web-app>
```
获取方式
<code>getServletContext().getInitParamter("foo");</code>
## Spring MVC启动流程-IOC容器的启动
在bean.xml中
```xml
<bean id="simpleUrlMapping" class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">
    <property name="mappings">
        <props>
            <prop key="/userlist.htm">userController</prop>
        </props>
    </property>
</bean>
<bean id="userController" class="com.springmvc.controller.UserController"/>
```
在web.xml中有如下配置
```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>
        /WEB-INF/applicationContext.xml
    </param-value>
</context-param>
<servlet>
    <servlet-name>springmvc</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>springmvc</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```
这里配置的listener就是IOC初始化的入口点，ContextLoaderListener实现ServletContextListener接口，这个接口是ServletContext的监听者，如果ServletContext（包括被创建、销毁等）发生变化，会调用不同的方法。当web容器启动的时候，发生的是ServletContext的创建，此时会调用ServletContextListener的contextInitialized方法
开车~
``` java
contextInitialized()
    //初始化根上下文
    initWebApplicationContext(event.getServletContext())
        this.context = createWebApplicationContext(servletContext)
            //判断使用什么样的IOC容器
            //如果没有在web.xml中指定，那么使用默认的WebApplicationContext
            contextClass = determineContextClass(ServletContext servletContext);
            //直接实例化容器
            wac = BeanUtils.instantiateCLass(contextClass);
            ...
        configureAndRefreshWebApplicationContext(cwac, servletContext);
            //将IOC容器和web容器挂钩
            wac.setServletContext(servletContext)
            //初始化容器
            wac.refresh()
        //将web容器和IOC容器挂钩
        servletContext.setAttribute(keyXXX, this.context);
```
简单地说，这个listener的作用是初始化当前web的根上下文信息（包括实例化和经典的refresh方法的调用）并将跟上下文和web容器挂钩（set某个属性而已）

Ps：除了根上下文外，每个servlet都会持有一个自己的上下文，并且把根上下文作为父上下文。
## Spring MVC启动流程-Spring WEB MVC的启动
直接开车
```java
DispatcherServlet#init()
    //模板方法具体由子类实现
    initServletBean()
        //初始化当前servlet的上下文信息，当前上下文信息的父类是根上下文信息
        //其余步骤和初始化根上下文基本一致，当前上下文也要加载servlet指定的bean配置文件
        //这时候加载的bean就是当前上下文独有的，此时也可以获取父类中的bean
        1.this.webApplicationContext = initWebApplicationContext();
            //有个很重要的方法
            onRefresh(wac);
        //空方法
		2.initFrameworkServlet();
```
小结：
从上面的代码，可以看出servlet会持有自己的上下文信息（有自己的Bean定义空间）
在初始化servlet上下文信息的过程中，会调用onRefresh(wac)方法，这个方法会调用DispatcherServlet的initStrategies方法，下面分析这个方法
## Spring MVC的实现
### Spring DispatcherServlet的初始化
```java
protected void initStrategies(ApplicationContext context) {
    initMultipartResolver(context);
    initLocaleResolver(context);
    initThemeResolver(context);
    //重点方法，handlerMapping的作用是为http请求找到匹配的Controller
    initHandlerMappings(context);
    initHandlerAdapters(context);
    initHandlerExceptionResolvers(context);
    initRequestToViewNameTranslator(context);
    initViewResolvers(context);
    initFlashMapManager(context);
}
```
URL请求和控制器（拦截器s）之间的映射关系是通过接口类HandlerMapping来封装的，HandlerMapping中定义了一个getHandler方法，通过该方法可以获得与HTTP请求对应的HandlerExecutionChain，其中封装了具体的Controller对象和拦截器链
```java
initHandlerMappings
    //找到所有的HandlerMapping类型的bean并存放到handlerMappings中
    //DispatcherServlet#handlerMappings(List类型)
    BeanFactoryUtils.beansOfTypeIncludingAncestors(context,HandlerMapping.class,true,false)
```
下面看看HandlerMapping到底是个啥
```java
public interface HandlerMapping {
    ...省略
    //HandlerExecutionChain是核心
    //通过该方法可以获得与HTTP请求对应的HandlerExecutionChain，其中封装了具体的Controller对象和拦截器链
    HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;
}
```
再看看HandlerExecutionChain
```java
public class HandlerExecutionChain {
    ...
    private final Object handler;
	private HandlerInterceptor[] interceptors;
	private List<HandlerInterceptor> interceptorList;
    ...
}
```
这个数据结构基本满足了我们的一切需求，包括和当前url匹配的Controller和Interceptor
我们看看到底是咋根据url匹配到这些内容的。
以SimpleUrlHandlerMapping为例，我们看到具体的<code>HandlerExecutionChain getHandler(HttpServletRequest request)</code>是在AbstractHandlerMapping中实现
```java
HandlerExecutionChain getHandler(HttpServletRequest request)
    //先根据request找到匹配的handler
    //简单地讲是从handlerMap中根据url找到匹配的handler
    Object handler = getHandlerInternal(request);
    //1.根据request找到匹配的拦截器 2.将拦截器和handler封装成handlerExecutionChain
    HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
```
具体的getHandlerInternal逻辑是，先从handlerMap中查找当前url对应的handler，如果没找到的话根据正则表达式找到所有匹配的handler
，再对结果进行排序，找到最符合的那个handler
那么handlerMap是何时初始化的呢？
首先hanlerMap位于AbstractUrlHandlerMapping中，该类是SimpleUrlHandlerMapping的父类
我们看继承关系图，发现SimpleUrlHandlerMapping还实现了ApplicationContextAware接口
这意味着在实例化该bean的时候会默认调用setApplicationContext方法，我们看到这个方法内部调用了
initApplicationContext()方法
```java
@Override
public void initApplicationContext() throws BeansException {
    //在这个方法中找到了所有的interceptor
    super.initApplicationContext();
    //注册url和handler关系
    registerHandlers(this.urlMap);
        //将url和handler放到handlerMap中
        this.handlerMap.put(urlPath, resolvedHandler);
}
```
小结：
1.SimpleUrlHandlerMapping实现了ApplicationContextAware接口，在容器刷新的时候，会调用其setApplicationContext方法
2.在上面方法的内部完成了，对SimpleUrlHandlerMapping的bean的属性中指明的url和handler进行保存，存放到handleMap中（以供handlerMapping调用getHandler()获取匹配的url）
3.在DispatchServlet初始化的时候，会执行initHandlerMappings(context)
4.该方法会找到所有的HandlerMapping类型的bean并存放到handlerMappings中（以供接下来遍历去寻找handler）
### Spring MVC对HTTP请求的分发处理
Servlet的入口doService
```java
doService(request,response)
    //1.准备ModelAndView 2.调用getHandler响应HTTP请求 
    //3.得到对应的ModelAndView 4.将ModelAndView交给视图对象去呈现
    doDispatch(request,response)
        //刚方法视图从request的属性中handler，如果没有再遍历handlerMapping获取handler
        HandlerExecutionChain mappedHandler = getHandler(request,false)
            //遍历所有的handlerMapping直到找到匹配的HandlerChain
            for(HandlerMapping hm : this.handlerMappings)
                handler = hm.getHandler(request)
        HandlerInterceptor[] interceptors = mappedHandler.getInterceptors()
        //遍历interceptors
        for(HandlerInterceptor h : interceptors)
            //执行拦截器的前置方法
            h.preHandle()
        //执行handler中的逻辑
        ModelAndView mv = 
        handler.handleRequest(request,response)
        //遍历interceptors
        for(HandlerInterceptor h : interceptors)
            //执行拦截器的后置方法
            h.postHandle()
        //适用视图对ModelAndView数据进行展示
        render(mv,request,response)
```
### Spring MVC视图的呈现
```java
void render(modelView,request,response)
    //根据modelAndView去得到View对象
    //遍历配置的viewResolvers来找到和modelAndView对应的View
    view = resolveViewName(modelAndView)
    //调用view实现对数据的呈现，通过httpResponse把视图呈现给http客户端
    //View具有很多实现，包括JSP FreeMarker Excel PDF等
    view.render(modedAndView,request,response)
```
## 小结
---
1）ServletContext启动
2）执行ContextLoaderListener的contextInitialized方法
3）该方法会启动并初始化根上下文(ApplicationContext)，并且和当前容器相互绑定，此时实例化的bean是web.xml中指定的applicationContext.xml中的bean
```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>
        classpath:spring/applicationContext.xml
    </param-value>
</context-param>
```
4）接着执行spring mvc的DispatchServlet的初始化工作，也就是调用init方法
5）接着启动当前servlet自己的上下文，并且设置根上下文为当前上下文的父类，此时实例化的bean是web.xml中指定的springmvc-context中配置的bean
```xml
<servlet>
    <servlet-name>springmvc</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <!--这里就是配置属于servlet自己的上下文中的bean的地方-->
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring/springmvc-context.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
```
6）在servlet的上下文启动过程中，同样会执行Spring的一些扩展点，比如通过BeanPostProcessor#postProcessBeforeInitialization会执行ApplicationContextAware的setApplicationContext方法，而SimpleUrlHandlerMapping就是ApplicationContextAware的子类，在该过程将配置的url和controller映射关系填充到了urlMap中
7）执行initHandlerMappings(context)方法，该方法会将所有的HandlerMapping类型的bean都找到并存储在handlerMappings中
8）当有请求到来的时候，会执行DispatchServlet的doService方法，该方法会遍历所有的handlerMappings调用其中handlerMapping的getHandler方法该方法会将找到的和当前request相匹配的handler以及inteceptor，并将其封装为handlerExecutorChain一并返回
9）遍历执行interceptor的preHandle方法；执行handler.handleRequest方法；遍历执行interceptor的postHandle方法
10) 视图的呈现

---