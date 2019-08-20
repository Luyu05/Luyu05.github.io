---
title: Java Web专题（一）
date: 2018-06-26 10:02:20
tags: Java Web
categories: Java Web
---
1.GET请求和POST请求
2.RESTFUL API
3.Session VS Cookie
4.Forward VS Redirect
<!-- more -->
### <font color=green size=5>GET请求和POST请求</font>
可以简单的认为HHTP中的POST GET PUT DELETE对应着CRUD
区别：
1）从语义上讲GET更偏向于查询操作，即GET请求不会对后台数据产生影响；而POST则不同
2）一般来讲GET请求是幂等的
Ps看到一个关于幂等很好的定义 abs(a) = abs(abs(a))
3）GET将数据放在url上，而POST将数据放在HTTP body中；POST相对安全
4）如果使用POST就无法保存书签

### <font color=green size=5>RESTFUL API</font>
参考链接：https://juejin.im/entry/59e460c951882542f578f2f0
1）URL表示资源；两种方式  /employees & /employees/56
2）名词+方法替换动词url /createEmployee -> POST + /employees 
                    /updateEmployee -> PUT /employees/56
3）条件筛选时，使用?添加条件 GET /employees?state=internal&maturity=senior
4）非资源请求使用动词url    GET /calculate?para2=23&para2=432

|POST（创建） | GET（读取）| PUT（更新）| DELETE（删除）
---- | --- | --- | ---
/employees | 创建一个新员工 |列出所有员工 | 批量更新员工信息 | 删除所有员工
/employees/56 | （错误） |获取56号员工的信息 | 更新56号员工的信息 | 删除56号员工

### <font color=green size=5>Session & Cookie</font>
<font color=#0099ff>HTTP是一个无状态协议，如何维持同一个用户的会话？</font>
对于客户的第一个请求，容器会生成一个唯一的sessionId给到客户端，客户再以后的每一个请求中发回这个sessionId，这样容器就知道客户的身份了
<font color=#0099ff>客户端和容器如何交换SessionId？</font>
最常用的方法是通过cookie，在服务端给到客户端的response中，header中有一个叫Set-Cookie的属性，用于存放JSESSIONID;在客户端到服务端的request中，header中有一个叫Cookie的属性，用于存放JSESSIONID
<font color=#0099ff>cookie和session的异同？</font>
二者都生成在服务端，而cookie（保存在客户端）可以视为session（保存在服务端）的一个助手，当然除了当助手之外cookie也可以实现自己的一些功能，比如用户名字的填充（js进行填充）
看一段服务器端生成cookie的代码
```java
public class CookieTest extends HttpServlet {
    public void doPost(HttpServletRequest request,
    HttpServletResponse response) {
        response.setContenType("text/html");
        //得到表单中用户填写的用户名
        String name = request.get("username");
        Cookie cookie = new Cookie("username",name);
        //jsessionId就是-1，浏览器关闭cookie死亡
        cookie.setMaxAge(-1);
        response.setCookie(cookie);
        ...
    }
}
```
而之后每次，服务端收到来自客户端的请求，都可以通过request.getCookies()获得所有的cookie
Ps看起来好像服务端为客户端创建的每个cookie，再之后客户端发送到请求中都会携带~
<font color=#0099ff>如果禁用cookie这套机制怎么玩的转？</font>
用户禁用cookie后，会忽视response的header中的Set-Cookie；此时需要换一种办法来交换sessionId
1）URL重写
origin url + ;jsessionid=1231231
注意这里是服务端返回给客户端的，在响应发回的html中（比如href）
2）表单隐藏
<font color=#0099ff>分布式环境如何实现session共享？</font>
参考链接：https://blog.csdn.net/u010028869/article/details/50773174?ref=myread
1）nginx对特定的sessionId进行映射，即一个用户的请求只会交给一个固定的机器处理
优点：无须对session做任何处理只要存在本机即可
缺点：如果某台机器挂掉，那么这台机器保存session的用户都要重新登录
2）session复制，即所有的服务器共享session，任何一个服务器上session发生改变，都同步到其他服务器
优点：容错性较好，即使机器挂掉也ok
缺点：容易造成网络堵塞
3）session第三方共享，redis（集群） or db（集群）
<font color=#0099ff>补充</font>
session的过时是由容器感知到的，可以在web-app下面配置
```java
<session-timeout/>
```
当然也可以直接在代码中session.setMaxInactiveInterval()方法设置过期时间

cookie设计的初衷是为了支持会话状态，不过也可以用来做一些其他事情
1）服务器和客户端直接交换key/value
2）常用于填写登录用户的用户名
### <font color=green size=5>Forward VS Redirect</font>
二者都可以称为请求转发，前者称为直接请求转发，后者称为间接请求转发
Redirect:客户端发送一个http请求，服务端Servlet1处理，之后执行response.sendRedirect("target url")，之后客户端再次向target url发送http请求
Forward:客户端发送一个http请求，服务端Servlet1处理，之后调用requestDispatcher.forward(request,response)，进行请求转发，直接转发到Servlet2处理，最后处理完成后将response返回到客户端
举个例子：Servlet将模型传递到jsp就是通过forward
```java
RequestDispatcher view = request.getRequestDispatcher("result.jsp");
view.forward(request,response);
```