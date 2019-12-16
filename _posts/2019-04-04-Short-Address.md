---
layout: post
title:  聊聊短地址及其原理
categories: Algorithms Short-Address
tags:  Algorithms Short-Address
author: wenzhilee77
---

# 简介

短地址的优势在于其短（字符少）、简洁，方便书写又不占位置。方便在社交网络和第三方平台上分享链接，投放广告。
比如有人会在发朋友圈的时候带上短地址，还有一些营销短信里面也会带一些短地址链接。
有现成的短地址生成器，常用的短地址转换有 百度短网址转换、Google短网址转换和新浪短地址转换，可惜的是谷歌关闭了该服务。

# HTTP请求重定向

HTTP 中的 301、302、303、307、308 响应状态码，都表示重定向的响应。
其中，301、308 响应状态码表示永久重定向，302、303、307 表示临时重定向。
那我们来说一下什么是重定向？
重定向(Redirect)就是通过各种方法将各种网络请求重新定个方向转到其它位置（如：网页重定向、域名的重定向、路由选择的变化也是对数据报文经由路径的一种重定向）。
举个例子，你要去 A 部门办理一个证件，等你去了之后A部门的某人告诉你他们不再受理此事了，需要你去 B 部门办理，然后你就自己去了 B 部门。这个过程就类似于重定向。
那么，HTTP 协议中定义的这些30X响应状态码就好比 A部门的某人 它们告诉客户端，你需要访问另外一个地址了。
重定向做了两次 HTTP 请求， 第一次，客户端请求 A 服务器，A 响应告诉浏览器，你应该去 B 服务器访问。此时就去访问服务器 B，这个时候你可以看到浏览器中的网址变了，这就是第二次 HTTP 请求。
重定向过程：
Step-1、浏览器（客户端）发送 HTTP 请求；
Step-2、Web服务器A接收后发送 302 状态码响应，并在响应头中把对应的 Location 给浏览器；
Step-3、浏览器收到服务器返回的 302 响应码，就自动再发送一个新的 HTTP 请求（请求URL是新的 Location 中的地址）；
Step-4、Web服务器（可能是 A 也可能是其他服务器）根据此请求寻找资源并发送给浏览器，最终展示给用户。
关于 Location 可以看下面的截图（这是一个重定向的 HTTP 请求示例），它是被放在响应头中的，其值是需要重定向的网址 http://www.veryitman.com，从这个过程来看可以看出重定向是客户端（浏览器）行为。

![](/images/shortAddr/16efa68cd7bc7252.jpg)

还有个叫转发的技术，这个和重定向是不一样的，转发是服务器行为，还拿上面的 A 部门例子来说，你要去 A 部门办理一个证件，等你去了之后 A 部门的某人告诉你他们不再受理此事了，但是 A 部门可以自己协调资源帮你完成而不需要你去其他部门办理了。这个过程就类似于转发。

# 模拟重定向

现在使用 SpringBoot 模拟一下重定向，我用的 SpringBoot 是 2.2.0.RELEASE 版本。
①、 新建 SpringBoot Web 工程。
②、 修改 pom 文件，增加 fastjson
```java
<dependency>
	<groupId>com.alibaba</groupId>
	<artifactId>fastjson</artifactId>
	<version>1.2.62</version>
</dependency>
```

③、 新建 MSTestRedirectController 文件，源码如下：
```java
@Controller
@RequestMapping(value = "testredirect")
public class MSTestRedirectController {

    @GetMapping(value = "/access/web")
    public String redirect() {
        return "redirect:/testredirect/index/realweb?parameter=coming";
    }

    @ResponseBody
    @GetMapping(value = "/index/realweb")
    public String real(HttpServletRequest request) {
        return "redirect happened：" + JSON.toJSONString(request.getParameterMap());
    }
}
```

这里要注意几个问题：
* 既然是重定向，该 Controller 不能使用 @RestController 注解而要使用 @Controller 注解；
* 这里使用了关键字 redirect 实现重定向；

④、 启动工程，并在浏览器访问下面的网址
```java
http://localhost:8080/testredirect/access/web
```

可以看到浏览器发生了重定向，截图如下：
![](/images/shortAddr/16efa68cd7ec8736.jpg)

除了上面的方法可以实现重定向外还可使用 HttpServletResponse 的 sendRedirect 方法，示例如下：
```java
@GetMapping(value = "/access/web2")
public String redirect2(HttpServletResponse response) {
    try {
        // 方法1：自定义状态码方式
        // String url = "http://localhost:8080/testredirect/index/realweb?parameter=coming";
        //response.setHeader("Location", url);
        //response.sendError(301);
        
        // 方法2：sendRedirect，默认返回的状态码是 302
        response.sendRedirect("/testredirect/index/realweb?parameter=coming");
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        return "";
    }
}
```

# 请求短地址过程
我拿网址 http://www.veryitman.com/ 来举实例，使用百度短地址服务。

在 百度短地址页面 去生成 http://www.veryitman.com/ 对应的短地址。

得到短地址是 https://dwz.cn/hnmau4Zs 复制该地址拷贝到浏览器（我用的是 Chrome 浏览器）的地址栏中，并打开 Chrome 的审查视图。切换到 Network 选项，此时回车打开短网址。

![](/images/shortAddr/16efa68cdaba8586.jpg)

HTTP 发送了 GET 请求（红色1），请求地址是  https://dwz.cn/hnmau4Zs ，服务器（百度的短地址服务）返回给 Chrome 浏览器 302 状态码，浏览器发现是该重定向码就再次用 Location 里面包含的地址发送了第二次请求即重定向请求。

你也可以使用微博的短地址服务，提醒一点，微博的短地址请求返回码是 301 而百度返回的是 302 响应码。302 状态码允许各种各样的重定向，一般情况下都会实现为到 GET 的重定向，但是不能确保 POST 会重定向为 POST，302 表示旧地址A的资源还在（仍然可以访问），这个重定向只是临时地从旧地址A跳转到地址B；而 301 状态码表明目标资源被永久的移动到了一个新的 URI，任何未来对这个资源的引用都应该使用新的 URI。

# 短地址原理

刚开始我很好奇，为什么我把长地址 A 转换为短地址 B，然后用 B 去访问居然还是 A 地址的内容，浏览器是怎么做到的？
通过上面的请求过程示例，相信大家应该大概理解了请求短地址的原理了。
在百度短地址服务中，我们将 http://www.veryitman.com/ 转换为 https://dwz.cn/hnmau4Zs  ，此时百度短地址服务维持了 短-长 地址的映射关系了而且是唯一的，当我们去访问 https://dwz.cn/hnmau4Zs ，其实请求的是百度短地址服务，该服务将短地址对应的长地址（放在响应头的 Location 中）返回给我们的浏览器，并返回 302 状态码，此时浏览器就重定向到了 http://www.veryitman.com/ 这个网址上了。
简单总结一下其步骤如下：
Step-1、用户在浏览器里输入 https://dwz.cn/hnmau4Zs  这个网址去访问;
Step-2、浏览器解析 DNS，获取该域名对应的 IP 地址;
Step-3、获取到 IP 后，浏览器发送 HTTP GET 请求查询 hnmau4Zs 并获取到 https://dwz.cn/hnmau4Zs  对应的长地址;
Step-4、HTTP 通过 302 状态码转到去请求对应的长地址  http://www.veryitman.com/ 上面了。
我把 http://www.veryitman.com/ 放到百度和微博的短地址生成分别是：

```java
// 百度短地址
https://dwz.cn/hnmau4Zs
```

```java
// 微博段地址
http://1t.click/aMtD
```

可以看出百度生成较复杂，首先协议变成了 HTTPS，其次生成代码是 8 位（hnmau4Zs），而微博生成的是 4 位（aMtD）代码。

短地址码一般都是由26个大写字母  A-Z 、26个小写字母 a-z 和10个数字  0-9 共62个字符随机组合而成，那么可以这样来生成短地址码，我们定义一个62进制，把这62个字符按照10进制数转成62进制数，那么就可以得到每个字符对应的62进制数了。同理，将短地址还原的时候把62进制转换为对应的10进制就可以了。

根据上面算法，可以看出百度可以支持 62^8 个短地址，微博可以支持 62^4 个短地址。

关于短地址生成的算法，大家可以用 SpringBoot 自己撸一个或者去网上找找别人已经实现的。


# 参考

https://juejin.im/post/5df247d8e51d455824270d7b?utm_source=gold_browser_extension