---
title: Java Web专题（四）
date: 2018-06-27 09:42:37
tags: Java Web
categories: Java Web
---
1.前后端分离 VS 不分离
2.Web容器
3.Apache VS Tomcat
4.Web应用的部署
<!-- more -->
### <font color=green size=5>前后端分离 VS 不分离</font>
a）不分离
Controller(Servlet)/Model(JavaBean)/View(JSP可以使用fm等模板引擎)
此时JSP=前端写的html+后端写的接口
b）分离
    1）客户端发送一个http请求（GET 最普通的请求只是获得静态资源），首先会先去CDN节点获取静态资源（html js css etc） Ps:CDN节点一般是web服务器
    2）之后在客户端本地执行js调用后端接口（也是http请求）
    3）这时候http请求一般会被nginx/apache/Tengine等web服务器接收到（在这一层可以实现负载均衡）
    4）之后web服务器发送请求到tomcat
    5）tomcat具体执行后端代码逻辑
    6）最后将response（一般是json），返回到js
    7）客户端渲染页面，呈现到用户面前
前后端分离与不分离最大的区别在于，是客户端拼接最终的html还是服务端拼接最终的html
c）node.js
将controller层转移到前端，此时后端只负责mvc中的model层
减少沟通成本，前端可自行封装json

### <font color=green size=5>Web容器</font>
当一个请求到达服务端后，需要有人来实例化Servlet、新建线程处理请求、调用Servlet的doPost()和doGet()方法、管理Servlet的生死，这个人就是Web容器（ Servlet容器）
Servlet没有main方法，它们受控于另一个Java应用（Web容器or Servlet容器）
Web服务器应用（apache）接收HTTP请求，如果该请求需要某个Servlet处理，那么将其交给Servlet容器
web.xml实际上是给容器用的(告诉容器如何使用Servlet和JSP)
生撸一个web容器？
1）main函数
2）创建和服务器的socket连接
3）采用BIO/NIO等
4）管理并运行Servlet
5）线程管理
6）翻译JSP
### <font color=green size=5>Apache VS Tomcat</font>
apache是web服务器，tomcat是servlet容器（也可做web服务器，性能较差）
apache只支持静态页面（html），tomcat支持动态页面（asp、jsp）
apache可以访问tomcat的资源，反之不可（一般通过AJP协议进行通讯）

为什么不单独使用tomcat？
1.tomcat对静态文件支持不好，需要web服务器来提升对静态文件的处理性能
2.利用web服务器进行负载均衡以及容错

### <font color=green size=5>Web应用的部署</font>
WEB-INF目录下的资源都不能通过路径url直接访问

如果url并未指定某一资源，而是定位到某个目录(一般是没有匹配的servlet)，这时候会寻找DD中配置的欢迎页面（从上到下）
```xml
<welcome-file-list>
    <welcome-file>index.html</welcome-file>
    <welcome-file>default.jsp</welcome-file>
</welcome-file-list>
```

错误页面
```xml
<error-page>
    <exception-type>java.lang.Throwable</exception-type>
    <!-- type和code只能存在一个 -->
    <error-code>404</error-code>
    <location>/errorPage.jsp</location>
</error-page>
```

### <font color=green size=5>一个http请求到达后端的全过程</font>
1 用户在浏览器键入url，并敲回车
2 首先去dns服务器解析当前的域名对应的实际ip地址与端口号
3 解析成功后，会与目标ip和端口号，建立tcp连接（三次握手）
Ps 这里可能有两条连接，客户端和nginx服务器，nginx服务器和tomcat服务器
4 将HTTP请求（符合HTTP协议的数据+tcp头+ip头+以太网头）发送到目标服务器
5 目标服务器对HTTP请求进行"剥离"，先是去以太网头，再去ip头
6 此时请求可能到了nginx服务器，nginx服务器对于动态资源的请求会分发到tomcat（通过http协议）
7 这个时候就到了tomcat，再去tcp头，接下来根据HTTP协议对数据进行封装，传递给后端代码

### <font color=green size=5>nginx如何与tomcat通信</font>
tomcat AJP协议只能和部分apache项目配合
非apache程序比如nginx，只能使用http协议

### <font color=green size=5>http协议和tcp协议的关系</font>
HTTP协议：生成针对目标服务器端HTTP请求报文  //人类语言的一封信
TCP协议：将HTTP请求报文分割成报文段（序号），把每个报文可靠地传给对方 //邮差

也可以说，TCP/IP协议是传输层协议，主要解决数据如何在网络中传输，
而HTTP是应用层协议，主要解决如何包装数据（使传输的数据有意义）
实际上socket是对TCP/IP协议的封装，Socket本身并不是协议，而是一个调用接口（API），通过Socket，我们才能使用TCP/IP协议

Ps
http1.1适用于解决包含多张图片的html页面（串行）
http2.0在同一个tcp连接中，可以并行发送多个请求（IO多路复用）

### <font color=green size=5>tomcat如何与客户端建立的tcp连接</font>
客户端发送http请求（包括TCP头，用于建立TCP连接）
通过浏览器与tomcat开放的端口，进行连接
接下来在这条通道进行数据的收发
Ps可以把http请求视为，具有特定格式的数据（符合http协议）通过tcp进行传递


### <font color=green size=5>tomcat/jetty/netty</font>
tomcat和jetty是一个层面上的，可以称为servlet容器
而servlet容器，通信部分要么是bio ，要么是nio
netty就是对java原生的nio都一种封装，大概就是这样吧

就像如果用bio生撸一个servlet容器一样，你也可以机智一些通过nio生撸一个servlet容器
当然如果你聪明绝顶，你会想到使用netty生撸一个servlet容器

简单地讲tomcat/jetty = bio/nio/netty + 管理并运行Servlet