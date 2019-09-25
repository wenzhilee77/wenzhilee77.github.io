---
layout: post
title:  如何优化网站
categories: 架构 限流 缓存
tags: 架构 限流 缓存
author: wenzhilee77
---

# 版本一

![](/images/architecture/image1.jpg)

系统开始是这样子的，一个 Tomcat 拖着一个 MySql 服务，跑在一个 2C 4G 的Linux服务器上，所有的请求都走 Tomcat，所有的查询都走 MySql，看起来像一句废话？

资源是有限的，那么如何有效的利用资源，提升服务性能？Tomcat 号称能抗住数十万并发访问，但是这事也得分场景，还得有足够牛逼的机器。

## Tomcat 优化
Tomcat支持以下三种模式：
1. BIO：一个线程处理一个请求，缺点：并发量高时，线程数较多，浪费资源，Tomcat7或以下在Linux系统中默认使用这种方式。
2. NIO：利用Java的异步IO处理，可以通过少量的线程处理大量的请求。Tomcat8在Linux系统中默认使用这种方式。Tomcat7 必须修改Connector配置来启（conf/server.xml配置文件）：
3. APR(Apache Portable Runtime)：从操作系统层面解决io阻塞问题。Linux如果安装了apr和native，Tomcat直接启动就支持apr。
为了方便易用这里我们选择NIO模式，小伙伴们直接下载使用 Tomcat8 以上版本即可，连接池什么的一般使用默认的即可。

# 版本二

![](/images/architecture/image2.jpg)

可能部分小伙伴知道 Tomcat 容器处理静态请求的性能力并不强，所以这里需要一款能处理静态文件请求又超牛逼的服务，这里推荐 Nginx，当然你可以使用其变种 Tengine、OpenResty 才实现动静分离。

# 版本三

![](/images/architecture/image3.jpg)

后端服务链接资源是宝贵的，在高并发下，会拖慢整个系统的响应时间。这里我们可以把一些热点数据进行缓存，后端读取缓存，如果数据存在则直接返回，否则再去读取数据库。

# 版本四

![](/images/architecture/image4.jpg)

资源是有限的，但用户可能是无限的，还可能有一些恶意用户、爬虫、热点搜索。为了大部门用户可以正常访问，这里我们使用前置限流，通过令牌桶算法或者漏桶算法实现多样的限流方案。

# 版本五

![](/images/architecture/image5.jpg)

在博客系统中，为了提升响应速度，加入了 Redis 缓存，把文章主键 ID 作为 key 值去缓存查询，如果不存在对应的 value，就去数据库中查找 。这个时候，如果请求的并发量很大，就会对后端的数据库服务造成很大的压力。这里我们使用布隆过滤器对空命中进行拦截处理。

# 版本六

![](/images/architecture/image6.jpg)

1. 如果仅仅对于一个博客而已一个Nginx 足够了，后面可以带多个Tomcat 做负载均衡进群
2. Nginx 应用层面做限流，后端单个服务可以做接口限流
3. 后端服务用户 Session 可以集中存储到 Redsi 中
4. 布隆过滤拦截防止缓存穿透
5. 热点数据读取 Redis 缓存
6. 如有必要 Redis 、MySql 可以做主从集群

# 参考

https://mp.weixin.qq.com/s/AfiTSjYw7DKc9283pVRoiw

