---
title: 谈谈Web开发中的跨域问题
date: 2019-08-14 19:45:49
tags: Java-Web
categories: Java-Web
---
谈谈Web开发中的跨域问题
<!-- more -->
## <font color=green size=5>什么是跨域</font>
『跨域』对于Web开发者来说是永远绕不过去的一个问题，日常开发中在和前端联调阶段经常就会因为跨域而联调失败。那么啥是跨域问题呢？
简单地说，跨域问题的根源是浏览器的同源策略引起的，这是出于安全考虑的。

<font color=blue size=3>那么什么是同源呢？</font>
同源：同协议&同域名&同端口
同源策略：针对跨域请求，请求发送了，服务器响应了，但是无法被浏览器接收
看几个例子
```java
举例来说：http://www.example.com/dir/page.html
协议是http://，域名是www.example.com，端口是80（默认端口可以省略）。
它的同源情况如下：
http://www.example.com/dir2/other.html：同源
https://example.com/dir/other.html ：非同源（协议不同）
http://v2.www.example.com/dir/other.html：非同源（域名不同）
http://www.example.com:81/dir/other.html：非同源（端口不同）
```

<font color=blue size=3>假如没有同源策略又会怎么样？</font>
我们看个例子：
假设你找到了一个盗版网站(www.meiju.com) 正在观看热播的美剧《绝命毒师》，看完一集之后突然想起来昨天加了购物车的宝贝还没有支付，于是打开了淘宝(www.taobao.com) ，登录之后就去支付，接着就愉快地去追剧了。but，事情并没有这么简单，晚上睡前你又看了下今天的订单，发现多了很多不是自己下的订单并且已经支付。
我们看看到底发生了什么，由于浏览器没有同源限制，在你登录了淘宝网之后携带了相应的cookie，而这时候盗版网站内置的js发送了很多购物的请求（淘宝网的服务器发现携带了你的登录cookie），这不是坑爹呢么。

所以同源策略是相当必要的，但是在我们日常开发中经常有跨域的需求。
一般公司都会有多个域名，假设公司A，他有这些域名，www.A.com,m.A.com,g.A.com
如果本次开发任务的页面是www.A.com 这个域名，而后端开发的ajax接口域名为m.A.com ，这时候就会因为浏览器的同源策略导致调用接口失败。

对于这种合理的跨域请求，有啥解决办法吗？我们继续往下看

## <font color=green size=5>解决跨域的方法</font>
一般常见的有两种方法，JSONP和CORS
我们要先明确几个点
1.一般跨域都是针对ajax接口来讲的
2.同源策略限制的只是浏览器能否正常展示后端返回值，即使存在跨域问题，前端的请求也是可以打到后端服务器的
3.所以我们需要对信任的域名开白处理
### <font color=blue size=3>JSONP</font>
原理：浏览器同源策略并不限制请求不同源的JS脚本，这就为开发者留下了后门，我们可以把原本要返回的json格式返回值，封装为一段JS代码，前端获得代码后会触发前后端约定好的函数从而完成整套逻辑。
举个例子：
```javascript
<script src='http://localhost:8080/testJsonp?callback=luyu05' type='text/javascript'/>
</script>
<script>
   function luyu05(data){//data需要的数据
      alert(data.message);
   }
</script>
```
当页面载入时候，会去拉取src中指定的JS脚本，即发送一个GET请求（<font color=red>这也是JSONP的短板，只能支持GET方法</font>）http://localhost:8080/testJsonp?callback=luyu05 到后端服务器。
我们再看看后端代码
```java
@Controller
public class TestJsonP {
    @RequestMapping("/testJsonp")
    @ResponseBody
    public Object test() {
        JsonObject res = new JsonObject();
        res.put("key0", "value0");
        res.put("key1", "value1");
        return res;
    }
}
```
Spring提供了统一地解决方案，只要配置了如下的bean就ok了，我们看到这里借助了Spring的aop机制（其实是针对每个@ResponseBody的返回值进行织入）
```java
@ControllerAdvice
public class JsonpAdvice extends AbstractJsonpResponseBodyAdvice {
    public JsonpAdvice() {
        super("callback");
    }
}
```
看下返回值
```java
luyu05({
  "key0": "value0",
  "key1": "value1"
});
```
当前端接收到这串代码时候，就会执行前端的luyu05方法（这个就是所谓的回调），而我们的json就是方法的入参data，从而完成整个流程。

### <font color=blue size=3>CORS</font>
相比JSONP的取巧方式，个人觉得CORS更加合规一些。

<font color=red>原理</font>：跨域资源共享标准新增了一组HTTP首部字段，通过这些字段让浏览器与服务器进行沟通，从而决定请求应该成功还是失败。

<font color=red>分类</font>：浏览器将跨域请求分为两类请求

只要同时满足以下两大条件，就属于<code>简单请求</code>。
```java
请求方法是以下三种方法之一：HEAD、GET、POST
HTTP的头信息不超出以下几种字段：
Accept
Accept-Language
Content-Language
Last-Event-ID
Content-Type：只限于三个值application/x-www-form-urlencoded、multipart/form-data、text/plain
```
凡是不同时满足上面两个条件，就属于非简单请求。

浏览器对这简单请求和非简单请求的处理是不一样的。
<font color=blue size=3>针对简单请求</font>
需要在Response的header里的Access-Allow-Origin的值包含请求头中Origin的域名（支持通配符和完整匹配两种）
后端代码如下
```java
public static void crossDomain(HttpServletRequest request,HttpServletResponse response){
    String origin = request.getHeader("Origin");
    if(!whiteList.contains(origin)) return;
    response.setHeader("Access-Control-Allow-Origin", origin);
}
```
<font color=blue size=3>针对非简单请求</font>
会执行一次预检查请求，这次的请求方法类型是OPTIONS，代表这次是用来询问的（如果servlet针对OPTIONS请求也进行处理可能会出现接口被调用两次的情况），除了Access-Allow-Origin外，此时还需要Access-Control-Request-Method和Access-Control-Request-Headers，这个其实很好理解，这次预检请求就是问问服务器是否支持当前的非简单请求（Method or Header特殊），如果支持的话会发送下次的真正请求，反之直接抛出跨域错误
```java
public static void crossDomain(HttpServletRequest request,HttpServletResponse response){
    String origin = request.getHeader("Origin");
    //过滤非法域名的请求
    if(!whiteList.contains(origin)) return;
    response.setHeader("Access-Control-Allow-Origin", origin);
    //如果需要http请求中带上cookie，需要前后端都设置credentials，且后端设置指定的origin
    response.setHeadr("Access-Control-Allow-Credentials", true);
    //需要针对非简单请求的特异项进行配置，比如本次请求方法是PUT方法，那么需要在Access-Control-Allow-Methods中写入PUT
    //相应地如果本次请求加入了自定义的header项，那么需要在Access-Control-Allow-Headers中写入对应的header项
    response.setHeader("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE");
    response.setHeader("Access-Control-Allow-Headers", "customer-header");
}
```
## <font color=green size=5>两种方案优缺点对比</font>
1.JSONP因为采用获取资源的方式，所以只能支持GET方法；而CORS基本支持所有类型的方法
2.JSONP方案前端需要进行特殊的编码；而CORS方案对前端几乎完全透明
3.可能某些老版本的浏览器不支持CORS方式，此时只能使用JSONP
4.使用CORS时可能出现预检请求触发了业务逻辑的问题，这点要注意
5.两种方案如果不做特殊处理都会执行对应ajax的逻辑，只是不返回到浏览器而已（因此某些场景要考虑过滤无效请求or保证接口幂等）

参考资料：
http://www.ruanyifeng.com/blog/2016/04/cors.html
https://segmentfault.com/a/1190000015597029