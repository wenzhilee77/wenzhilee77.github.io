---
layout: post
title:  "Java 高并发限流措施概述"
categories: Java Concurrent Limiting
tags:  Java Concurrent Limiting RateLimiter  
author: wenzhilee77
---

* content
{:toc}

## 为什么要限流

在大数据量高并发访问时，经常会出现服务或接口面对暴涨的请求而不可用的情况，甚至引发连锁反映导致整个系统崩溃。此时你需要使用的技术手段之一就是限流，当请求达到一定的并发数或速率，就进行等待、排队、降级、拒绝服务等。
在开发高并发系统时有三把利器用来保护系统：缓存、降级和限流。

* 缓存

缓存比较好理解，在大型高并发系统中，如果没有缓存数据库将分分钟被爆，系统也会瞬间瘫痪。使用缓存不单单能够提升系统访问速度、提高并发访问量，也是保护数据库、保护系统的有效方式。大型网站一般主要是“读”，缓存的使用很容易被想到。在大型“写”系统中，缓存也常常扮演者非常重要的角色。比如累积一些数据批量写入，内存里面的缓存队列（生产消费），以及HBase写数据的机制等等也都是通过缓存提升系统的吞吐量或者实现系统的保护措施。甚至消息中间件，你也可以认为是一种分布式的数据缓存。

* 降级

服务降级是当服务器压力剧增的情况下，根据当前业务情况及流量对一些服务和页面有策略的降级，以此释放服务器资源以保证核心任务的正常运行。降级往往会指定不同的级别，面临不同的异常等级执行不同的处理。根据服务方式：可以拒接服务，可以延迟服务，也有时候可以随机服务。根据服务范围：可以砍掉某个功能，也可以砍掉某些模块。总之服务降级需要根据不同的业务需求采用不同的降级策略。主要的目的就是服务虽然有损但是总比没有好。

* 限流

限流可以认为服务降级的一种，限流就是限制系统的输入和输出流量已达到保护系统的目的。一般来说系统的吞吐量是可以被测算的，为了保证系统的稳定运行，一旦达到的需要限制的阈值，就需要限制流量并采取一些措施以完成限制流量的目的。比如：延迟处理，拒绝处理，或者部分拒绝处理等等。


## 限流算法简介

* 计数器

计数器限流算法也是比较常用的，主要用来限制总并发数，比如数据库连接池大小、线程池大小、程序访问并发数等都是使用计数器算法。也是最简单粗暴的算法。
在一段时间内，进行计数，与阀值进行比较，到了时间临界点，将计数器清0。

![](/images/limit/count.png)

* 滑动窗口

由于计数器存在临界点缺陷，后来出现了滑动窗口算法来解决。
滑动窗口的意思是说把固定时间片，进行划分，并且随着时间的流逝，进行移动，这样就巧妙的避开了计数器的临界点问题。也就是说这些固定数量的可以移动的格子，将会进行计数判断阀值，因此格子的数量影响着滑动窗口算法的精度。

![](/images/limit/sliding.png)

* 令牌桶

令牌桶算法是一个存放固定容量令牌（token）的桶，按照固定速率往桶里添加令牌。令牌桶算法基本可以用下面的几个概念来描述：
1)令牌将按照固定的速率被放入令牌桶中。比如每秒放10个。
2)桶中最多存放b个令牌，当桶满时，新添加的令牌被丢弃或拒绝。
3)当一个n个字节大小的数据包到达，将从桶中删除n个令牌，接着数据包被发送到网络上。
4)如果桶中的令牌不足n个，则不会删除令牌，且该数据包将被限流（要么丢弃，要么缓冲区等待）。

![](/images/limit/token.png)

* 漏桶

漏桶算法即leaky bucket是一种非常常用的限流算法，可以用来实现流量整形（Traffic Shaping）和流量控制（Traffic Policing）。
漏桶算法的主要概念如下：
1) 一个固定容量的漏桶，按照常量固定速率流出水滴；
2) 如果桶是空的，则不需流出水滴；
3) 可以以任意速率流入水滴到漏桶；
4) 如果流入水滴超出了桶的容量，则流入的水滴溢出了（被丢弃），而漏桶容量是不变的。

![](/images/limit/licking.png)


## 限流算法实现

* 计数器

```java
public class CountRateLimiterDemo
{

    private static AtomicInteger count = new AtomicInteger(0);

    public static void exec()
    {
        if (count.get() >= 5)
        {
            System.out.println("请求用户过多，请稍后在试！" + System.currentTimeMillis() / 1000);
        }
        else
        {
            count.incrementAndGet();
            try
            {
                //处理核心逻辑  
                TimeUnit.SECONDS.sleep(1);
                System.out.println("--" + System.currentTimeMillis() / 1000);
            }
            catch (InterruptedException e)
            {
                e.printStackTrace();
            }
            finally
            {
                count.decrementAndGet();
            }
        }
    }
}
```

```java
public class CountRateLimiterDemo
{

    private static Semaphore semphore = new Semaphore(50);

    public static void exec()
    {
        if (semphore.getQueueLength() > 100)
        {
            System.out.println("当前等待排队的任务数大于100，请稍候再试...");
        }
        try
        {
            semphore.acquire();
            // 处理核心逻辑  
            TimeUnit.SECONDS.sleep(1);
            System.out.println("--" + System.currentTimeMillis() / 1000);
        }
        catch (InterruptedException e)
        {
            e.printStackTrace();
        }
        finally
        {
            semphore.release();
        }
    }
}
```

```java
public class CounterRateLimiting
{
    private static long timeStamp = System.currentTimeMillis();
    private static long limitCount = 100;
    private static long interval = 1000;
    private static long reqCount = 0;
    
    public static boolean grant()
    {
        long now = System.currentTimeMillis();
        if(now < timeStamp + interval)
        {
            if (reqCount < limitCount)
            {
                ++reqCount;
                return true;
            }
            else
                return false;
        }
        else
        {
            timeStamp = System.currentTimeMillis();
            reqCount = 0;
            return false;
        }
    }
}
```

* 令牌桶

限流神器：Guava RateLimiter
Guava不仅仅在集合、缓存、异步回调等方面功能强大，而且还给我们封装好了限流的API！
Guava RateLimiter基于令牌桶算法，我们只需要告诉RateLimiter系统限制的QPS是多少，那么RateLimiter将以这个速度往桶里面放入令牌，然后请求的时候，通过tryAcquire()方法向RateLimiter获取许可（令牌）。

* 漏桶

```java
public class LeakyBucket
{
    private static long time = System.currentTimeMillis();
    // 桶里面现在的水
    private static int water = 0;
    private static int size = 10;
    // 出水速度
    private static int rate = 3;
    
    public static boolean grant()
    {
        // 计算出水的数量
        long now = System.currentTimeMillis();
        int out = (int) ((now - time)/700*rate);
        // 漏出水后的剩余
        water = Math.max(0, water - out);
        time = now;
        if((water + 1) < size)
        {
            ++ water;
            return true;
        }
        
        return false;
    }
}
```


## 应用层限流

* 限流总并发/连接/请求数

对于一个应用系统来说一定会有极限并发 / 请求数，即总有一个 TPS/QPS 阀值，如果超了阀值则系统就会不响应用户请求或响应的非常慢，因此我们最好进行过载保护，防止大量请求涌入击垮系统。
如果你使用过 Tomcat ，其 Connector 其中一种配置有如下几个参数：
1) acceptCount ：如果 Tomcat 的线程都忙于响应，新来的连接会进入队列排队，如果超出排队大小，则拒绝连接；
2) maxConnections ： 瞬时最大连接数，超出的会排队等待；
3) maxThreads ： Tomcat 能启动用来处理请求的最大线程数，如果请求处理量一直远远大于最大线程数则可能会僵死。
详细的配置请参考官方文档。另外如 Mysql （如 max_connections ）、 Redis （如tcp-backlog ）都会有类似的限制连接数的配置。

* 限流总资源数

如果有的资源是稀缺资源（如数据库连接、线程），而且可能有多个系统都会去使用它，那么需要限制应用；可以使用池化技术来限制总资源数：连接池、线程池。比如分配给每个应用的数据库连接是 100 ，那么本应用最多可以使用 100 个资源，超出了可以等待或者抛异常。

* 限流某个接口的总并发/请求数

如果接口可能会有突发访问情况，但又担心访问量太大造成崩溃，如抢购业务；这个时候就需要限制这个接口的总并发 / 请求数总请求数了；因为粒度比较细，可以为每个接口都设置相应的阀值。
适合对业务无损的服务或者需要过载保护的服务进行限流，如抢购业务，超出了大小要么让用户排队，要么告诉用户没货了，对用户来说是可以接受的。而一些开放平台也会限制用户调用某个接口的试用请求量，也可以用这种计数器方式实现。这种方式也是简单粗暴的限流，没有平滑处理，需要根据实际情况选择使用；

* 限流某个接口的时间窗请求数

即一个时间窗口内的请求数，如想限制某个接口/服务每秒/每分钟/每天的请求数/调用量。如一些基础服务会被很多其他系统调用，比如商品详情页服务会调用基础商品服务调用，但是怕因为更新量比较大将基础服务打挂，这时我们要对每秒/每分钟的调用量进行限速；

* 平滑限流某个接口的请求数

之前的限流方式都不能很好地应对突发请求，即瞬间请求可能都被允许从而导致一些问题；因此在一些场景中需要对突发请求进行整形，整形为平均速率请求处理（比如 5r/s ，则每隔 200 毫秒处理一个请求，平滑了速率）。这个时候有两种算法满足我们的场景：令牌桶和漏桶算法。 Guava 框架提供了令牌桶算法实现，可直接拿来使用。
Guava RateLimiter 提供了令牌桶算法实现：平滑突发限流 (SmoothBursty) 和平滑预热限流 (SmoothWarmingUp) 实现。


## 接入层限流
接入层通常指请求流量的入口，该层的主要目的有：负载均衡、非法请求过滤、请求聚合、缓存、降级、限流、 A/B 测试、服务质量监控等等。

对于 Nginx 接入层限流可以使用 Nginx 自带了两个模块：连接数限流模块ngx_http_limit_conn_module 和漏桶算法实现的请求限流模块ngx_http_limit_req_module 。还可以使用 OpenResty 提供的 Lua 限流模块 lua-resty-limit-traffic 进行更复杂的限流场景。
limit_conn 用来对某个 KEY 对应的总的网络连接数进行限流，可以按照如 IP 、域名维度进行限流。 limit_req 用来对某个 KEY 对应的请求的平均速率进行限流，并有两种用法：平滑模式（ delay ）和允许突发模式 (nodelay) 。

* ngx_http_limit_conn_module

limit_conn 是对某个 KEY 对应的总的网络连接数进行限流。可以按照 IP 来限制 IP维度的总连接数，或者按照服务域名来限制某个域名的总连接数。但是记住不是每一个请求连接都会被计数器统计，只有那些被 Nginx 处理的且已经读取了整个请求头的请求连接才会被计数器统计。

* ngx_http_limit_req_module

上面我们使用到了ngx_http_limit_conn_module 模块，来限制连接数。那么请求数的限制该怎么做呢？这就需要通过ngx_http_limit_req_module 模块来实现，该模块可以通过定义的键值来限制请求处理的频率。特别的，可以限制来自单个IP地址的请求处理频率。 限制的方法是使用了漏斗算法，每秒固定处理请求数，推迟过多请求。如果请求的频率超过了限制域配置的值，请求处理会被延迟或被丢弃，所以所有的请求都是以定义的频率被处理的。


## 分布式限流

待续。。。。。。







## 参考

https://m635674608.iteye.com/blog/2339587
https://www.idcbest.com/idcnews/11002288.html

