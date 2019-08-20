---
title: 关于Tomcat
date: 2018-09-05 20:36:24
tags: Java Web
categories: Java Web
---
Tomcat真是不能言说的伤，看了好几遍了总是再门外徘徊
本来想着以后再也不看了，但是前阵子在掘金上又看了一篇讲解的特别清晰的文章
自己也想把Tomcat的内容总结下（搬运下）
<!-- more -->
## 一、Tomcat顶层架构
一图胜千言
<img src='001.jpg'/>
<img src='tomcat1.jpg'/>
（1）Tomcat中只有一个Server（代表整个服务器），一个Server包含多个Service，一个Service可以有多个Connector和一个Container
（2）Connector用于接收请求并将请求封装成Request和Response来具体处理；最底层使用Socket实现，而request和response是按HTTP协议封装的，所以Connector要同时支持TCP/IP和HTTP两种协议
（3）Container用于封装和管理Servlet，以及具体处理request请求
## 二、Connector架构分析
<img src='tomcat2.jpg'/>
Connector就是使用ProtocolHandler来处理请求的，不同的ProtocolHandler代表不同的连接类型，比如：Http11Protocol使用的是普通Socket来连接的，Http11NioProtocol使用的是NioSocket来连接的。

其中ProtocolHandler由包含了三个部件：Endpoint、Processor、Adapter。

（1）Endpoint用来处理底层Socket的网络连接，Processor用于将Endpoint接收到的Socket封装成Request，Adapter用于将Request交给Container进行具体的处理。

（2）Endpoint由于是处理底层的Socket网络连接，因此Endpoint是用来实现TCP/IP协议的，而Processor用来实现HTTP协议的，Adapter将请求适配到Servlet容器进行具体的处理。

（3）Endpoint的抽象实现AbstractEndpoint里面定义的Acceptor和AsyncTimeout两个内部类和一个Handler接口。Acceptor用于监听请求，AsyncTimeout用于检查异步Request的超时，Handler用于处理接收到的Socket，在内部调用Processor进行处理。

Ps Container一般通过HTTP协议和外界交流，AJP仅仅适用于部分apache项目比如apache服务器
## 三、Container架构分析
<img src='tomcat3.jpg'/>
4个子容器的作用分别是：

（1）Engine：引擎，用来管理多个站点，一个Service最多只能有一个Engine； 
（2）Host：代表一个站点，也可以叫虚拟主机，通过配置Host就可以添加站点； 
（3）Context：代表一个应用程序，对应着平时开发的一套程序，或者一个WEB-INF目录以及下面的web.xml文件； 
（4）Wrapper：每一Wrapper封装着一个Servlet；

pipeline模式在Tomcat中的应用
<img src='tomcat4.jpg'/>
每个Pipeline都有特定的Valve，而且是在管道的最后一个执行，这个Valve叫做BaseValve，BaseValve是不可删除的
当执行到StandardWrapperValve的时候，会在StandardWrapperValve中创建FilterChain，并调用其doFilter方法来处理请求，这个FilterChain包含着我们配置的与请求相匹配的Filter和Servlet，其doFilter方法会依次调用所有的Filter的doFilter方法和Servlet的service方法，这样请求就得到了处理！
## 四、Tomcat和类加载器
之前看JVM一直对类加载器不甚理解，最近看了另外一篇文章有了一定的认识
有一点需要认识到，类加载器是有作用范围的，一般某个特定的类加载器只能加载位于特定目录下的类

首先，我们来问个问题：
Tomcat 如果使用默认的类加载机制行不行？
我们思考一下：Tomcat是个web容器， 那么它要解决什么问题： 
1. 一个web容器可能需要部署两个应用程序，不同的应用程序可能会依赖同一个第三方类库的不同版本，不能要求同一个类库在同一个服务器只有一份，因此要保证每个应用程序的类库都是独立的，保证相互隔离。 
2. 部署在同一个web容器中相同的类库相同的版本可以共享。否则，如果服务器有10个应用程序，那么要有10份相同的类库加载进虚拟机，这是扯淡的。 
3. web容器也有自己依赖的类库，不能于应用程序的类库混淆。基于安全考虑，应该让容器的类库和程序的类库隔离开来。 
4. web容器要支持jsp的修改，我们知道，jsp 文件最终也是要编译成class文件才能在虚拟机中运行，但程序运行后修改jsp已经是司空见惯的事情，否则要你何用？ 所以，web容器需要支持 jsp 修改后不用重启。

再看看我们的问题：Tomcat 如果使用默认的类加载机制行不行？ 
答案是不行的。为什么？我们看，第一个问题，如果使用默认的类加载器机制，那么是无法加载两个相同类库的不同版本的，默认的类加载器是不管你是什么版本的，只在乎你的全限定类名，并且只有一份。第二个问题，默认的类加载器是能够实现的，因为他的职责就是保证唯一性。第三个问题和第一个问题一样。我们再看第四个问题，我们想我们要怎么实现jsp文件的热修改（楼主起的名字），jsp 文件其实也就是class文件，那么如果修改了，但类名还是一样，类加载器会直接取方法区中已经存在的，修改后的jsp是不会重新加载的。那么怎么办呢？我们可以直接卸载掉这jsp文件的类加载器，所以你应该想到了，每个jsp文件对应一个唯一的类加载器，当一个jsp文件修改了，就直接卸载这个jsp类加载器。重新创建类加载器，重新加载jsp文件。

Tomcat的类加载器
<img src='tomcat5.jpg'/>
我们看到，前面3个类加载和默认的一致，CommonClassLoader、CatalinaClassLoader、SharedClassLoader和WebappClassLoader则是Tomcat自己定义的类加载器，它们分别加载/common/*、/server/*、/shared/*（在tomcat 6之后已经合并到根目录下的lib目录下）和/WebApp/WEB-INF/*中的Java类库。其中WebApp类加载器和Jsp类加载器通常会存在多个实例，每一个Web应用程序对应一个WebApp类加载器，每一个JSP文件对应一个Jsp类加载器。

WebApp可以视为一个个web应用程序使用的加载器，Jsp类加载器视为一个个Jsp页面使用的加载器
<font color='blue'>Common ClassLoader能加载的类都可以被Catalina ClassLoader和SharedClassLoader使用，从而实现了公有类库的共用，而CatalinaClass Loader和Shared ClassLoader自己能加载的类则与对方相互隔离。
WebApp ClassLoader可以使用Shared ClassLoader加载到的类，但各个WebApp ClassLoader实例之间相互隔离。
而JasperLoader的加载范围仅仅是这个JSP文件所编译出来的那一个.Class文件，它出现的目的就是为了被丢弃：当Web容器检测到JSP文件被修改时，会替换掉目前的JasperLoader的实例，并通过再建立一个新的Jsp类加载器来实现JSP文件的HotSwap功能。</font>

后记：
其实破坏双亲委派模型很简单，只要覆写父类中的loadClass()方法就行了，把委派的逻辑去掉改为自己去加载类。
至于破坏的目的，拿tomcat来讲，主要是<font color='red'>隔离（类的不同版本）、热插拔</font>

参考链接：
https://blog.csdn.net/xlgen157387/article/details/79006434
https://blog.csdn.net/qq_38182963/article/details/78660779