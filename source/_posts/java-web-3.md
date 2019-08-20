---
title: Java Web专题（三）
date: 2018-06-27 09:40:52
tags: Java Web
categories: Java Web
---
1.Servlet
2.Filter
3.Listener
参考《Head First Servlets and JSP》
<!-- more -->
### <font color=green size=5>Servlet</font>
**<font color=#0099ff>简单的MVC</font>**
Servlet充当Controller，JSP充当View，具体的业务逻辑以及数据库的操作充当Model
**<font color=#0099ff>Servlet的生命周期</font>**
1)容器加载Servlet类
2)实例化Servlet Ps容器让其成为一个普通的对象
3)init(servletConfig)（可以覆盖该无参的init方法，执行一些初始化逻辑，比如数据库连接） 
4)service()->doPost()/doGet() etc
5)destroy()
**<font color=#0099ff>HttpServletRequest中的属性</font>**
举几个常见的例子
1)cookies
2)session
3)localPort/Addr tomcat配置的地址端口号
4)serverPort/Addr apache的地址和端口号
5)remotePort/Addr 客户端的地址和端口号
5)parameterValues/parameter
**<font color=#0099ff>HttpServletResponse</font>**
1)setContentType 通知客户端浏览器当前消息类型
2)getWriter/getOutputStream 获取输出流将内容输出到客户端浏览器
3)sendRedirect 重定向
**<font color=#0099ff>ServletConfig VS ServletContext</font>**
每个servlet有一个ServletConfig
每个Web应用有一个ServletContext
<font color=red>ServletConfig：</font>
执行Servlet的init方法时会为该Servlet创建一个唯一的ServletConfig，容器从Servlet的配置项中读取初始化参数并封装到ServletConfig中,并传递给init方法
```java
<Servlet>
 <init-param> key value </init-param>
</Servlet>
```
容器只能读取一次ServletConfig，就是在执行init方法的时候（这意味着如果修改servlet配置，包括initParam，需要重启容器）
直接在doXX()方法中调用getServletConfig().getInitParamterNames()&getInitParamter()即可获取initParam
此时如果想要将initParam传递到某个Servlet或者JSP，可以利用forward指定目标，此时的模式是点对点，并不是整个应用所有部分都能访问到这些参数
这时候引出ServletContext
<font color=red>ServletContext：</font>
针对整个应用，所以下面的配置并未嵌套在某个servlet中，而是位于web-app中
在每一个Servlet中，都可以这样调用getServletContext().getInitParamter("paramName")
```java
<context-param>
key value
</context-param>
```
Ps二者都没有setInitParamter方法（getInitParamter），意味着无法动态改变内容
**<font color=#0099ff>如何保证ServletContext的线程安全？</font>**
1）如果对ServletA的doXX()加synchronized,此时ServletB仍然可以修改context，即使该方法有加synchronized。
synchronized关键字此时的作用范围是对象，如果Servlet非单例，此时如果new ServletA加的锁也是不生效的
2）synchronized(getServletContext())
**<font color=#0099ff>补充</font>**
Servlet是单例的
param更贴近于初始化阶段（get）
attribute更贴近于动态配置（set/get）
### <font color=green size=5>Filter</font>
Filter最主要的几个方法
init(FilterConfig)
doFilter(ServletRequest,ServletResponse)
destroy()
Filter们通过chain连接在一起，在doFilter中有一个重要的方法chain.doFilter(ServletRequest,ServletResponse)
为了防止servlet直接将response（HttpServletResponse）返回给容器而不经过filter执行doFilter后面的代码，一般将response进行包装后传递给下一个filter或者servlet

举个栗子
Filter
```java
public void doFilter(ServletRequest request, ServletResponse response,
 FilterChain chain) {
        System.out.println("hello filter");
        chain.doFilter(request, response);
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("bye filter");    
}
```
Servlet
```java
PrintWriter out = response.getWriter();
        out.println("<!DOCTYPE HTML PUBLIC \"-//W3C HTML 4.01 Transitional//EN\">");
        out.println("<HTML>");
        out.println(" <HEAD><TITLE>A Servlet Written By Milk</TITLE></HEAD>");
        out.println("<BODY>");
        out.println("Filter test");
        out.println("</BODY>");
        out.println("</HTML>");
        out.flush();
        out.close();
```
此时浏览器几乎立刻显示html内容，而控制台过了3s后才打印"bye filter"
可见此时doFilter后面的代码，对返回到客户端的response没任何影响
如何想要先执行chain.doFilter之后的代码，再返回结果给客户端应该怎么办？
思路：这时候需要自己造一个假的response给到servlet,filter还要将加的response中的内容写到真的response中，再返回给客户端
```java
public class MyResponse extends HttpServletResponseWrapper {
    private PrintWriter cachedWriter;
    private CharArrayWriter bufferedWriter;
    
    public MyResponse(HttpServletResponse response) {
        super(response);
        bufferedWriter = new CharArrayWriter();
        cachedWriter = new PrintWriter(bufferedWriter);
    }

    public PrintWriter getWriter() throws IOException {
        return cachedWriter;
    }

    public CharArrayWriter getBufferedWriter() {
        return bufferedWriter;
    }
}
```
```java
public void doFilter(ServletRequest request, ServletResponse response, 
FilterChain chain) {
        System.out.println("hello filter");
        HttpServletResponse httpResponse = (HttpServletResponse) response;
        MyResponse myResponse = new MyResponse(httpResponse);
        chain.doFilter(request,myResponse);
        response.getWriter().write(myResponse.getBufferedWriter().toString());
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("bye filter");
        }
```

### <font color=green size=5>Listener</font>
Java Web的main函数---Listener,在Servlet之前,完成某些全局的初始化工作
Ps我们当然也可以将listener的工作，放在某个servlet中执行，前提是你确保这个servlet能够最先运行，这明显不靠谱

ServletContextListener监听ServletContext的初始化和撤销
初始化时：从ServletContext得到上下文初始化参数
        使用初始化参数（String）建立一个dog或者数据库连接（Object）
        把数据库连接或者dog(Object)存储为一个ServletContext属性，供整个Web应用使用
撤销时：关闭数据库连接

1）创建监听者类
2）在web.xml中放置一个listener元素
```java
<listener>
listener class
</listener>
```
Ps 容器根据listener class实现的接口类型得知监听的目标