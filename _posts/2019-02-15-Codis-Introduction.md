---
layout: post
title:  "Codis 简介"
categories: Redis Codis HA
tags:  Codis Redis HA  
author: wenzhilee77
---

* content
{:toc}

## Codis 简介

想必大家的架构中，Redis已经是一个必不可少的部件，丰富的数据结构和超高的性能以及简单的协议，让Redis能够很好的作为数据库的上游缓存层。但是我们会比较担心Redis的单点问题，单点Redis容量大小总受限于内存，在业务对性能要求比较高的情况下，理想情况下我们希望所有的数据都能在内存里面，不要打到数据库上，所以很自然的就会寻求其他方案。 比如，SSD将内存换成了磁盘，以换取更大的容量。

更自然的想法是将Redis变成一个可以水平扩展的分布式缓存服务，在Codis之前，业界只有Twemproxy，但是Twemproxy本身是一个静态的分布式Redis方案，进行扩容/缩容时候对运维要求非常高，而且很难做到平滑的扩缩容。Codis的目标其实就是尽量兼容Twemproxy的基础上，加上数据迁移的功能以实现扩容和缩容，最终替换Twemproxy。从豌豆荚最后上线的结果来看，最后完全替换了Twem，大概2T左右的内存集群。

Codis 是一个分布式 Redis 解决方案, 对于上层的应用来说, 连接到 Codis Proxy 和连接原生的 Redis Server 没有显著区别 (除一些命令不支持), 上层应用可以像使用单机的 Redis 一样使用, Codis 底层会处理请求的转发, 不停机的数据迁移等工作, 所有后边的一切事情, 对于前面的客户端来说是透明的, 可以简单的认为后边连接的是一个内存无限大的 Redis 服务。


## Codis的架构

![](/images/codis/simple0)

简单理解 Codis 就是一个代理中间件，用的是 GO 语言开发的，如下图，Codis 在系统的位置是这样的。
Codis分为四个部分，分别是Codis Proxy (codis-proxy)、Codis Dashboard (codis-config)、Codis Redis (codis-server)和ZooKeeper/Etcd.
Codis就是起着一个中间代理的作用，能够把所有的Redis实例当成一个来使用，在客户端操作着SDK的时候和操作Redis的时候是一样的，没有差别。

![](/images/codis/codis-gov.png)

这幅图是Codis官方的架构图，个人觉得比较难看，但是也反应了Codis的架构，Codis可以对Redis分组，一个RedisGroup里有一个Master，和多个Slave，Redisclient可以直接访问Proxy。还有一个CodisHA的客户端。但是这幅图有些内容还没有体现出来， 于是我就画了一幅更丑的。

![](/images/codis/simple1.png)

Codis的内部，有三个主要的模块， Router，Model，Redis。 Router负责将前端的请求转发给Redis， Model负责和ZK交互，保持数据一致性，这里的数据主要是Group配置，Proxy配置，Slot的配置。Redis模块负责和Redis交互，将前端的命令转发给Redis，并将结果返回。刚才说了Model在负责把Proxy的配置变化发送给ZK， ZK再变化同步到Codisclient。

![](/images/codis/simple2.png)

好了，现在切换到这幅图。刚才的那副是Codis的一个骨架， 这里是细化到肉了，这幅图稍微好看点。首先说Router，这里负责和Client通信，建立session，通过Mapper来过滤所有的请求。Slots中封装了Redis的连接。 下面这块Redis封装了Decode和EncodeRedis协议的过程，这里值得提一下，Redis是有自己的标准协议的，在Redis之间的交互，Codis使用的就是标准协议，这块的代码大家有兴趣可以看一下。

Model模块是和ZK交互的对象，负责发送指令给客户端，并解析返回结果。后面也会给大家主要介绍这块内容。


## dashboard

dashboard在codis中是一个很重要的角色，所有的集群信息变更操作都是通过dashboard发起的（这个设计有点像docker），dashboard对外暴露了一系列RESTfulAPI接口，不管是web管理工具，还是命令行工具都是通过访问这些httpapi来进行操作的，所以请保证dashboard和其他各个组件的网络连通性。

## zookeeper

zk除了存储路由信息，同时还作为一个事件同步的媒介服务，比如变更master或者数据迁移这样的事情，需要所有的proxy通过监听特定zk事件来实现可以说zk被我们当做了一个可靠的rpc的信道来使用。因为只有集群变更的admin时候会往zk上发事件，proxy监听到以后，回复在zk上，admin收到各个proxy的回复后才继续。本身集群变更的事情不会经常发生，所以数据量不大。Redis的主从切换是通过codis-ha在zk上遍历各个server group的master判断存活情况，来决定是否发起提升新master的命令。


## 数据一致性

其实Codis的HA，并不能保证数据完全不丢失，因为是异步复制，所以master挂掉后，如果有没有同步到slave上的数据，此时将slave提升成master后，刚刚写入的还没来得及同步的数据就会丢失。不过在RebornDB中我们会尝试对持久化存储引擎（qdb）可能会支持同步复制(syncreplication)，让一些对数据一致性和安全性有更强要求的服务可以使用。

说到一致性，这也是Codis支持的MGET/MSET无法保证原本单点时的原子语义的原因。 因为MSET所参与的key可能分不在不同的机器上，如果需要保证原来的语义，也就是要么一起成功，要么一起失败，这样就是一个分布式事务的问题，对于Redis来说，并没有WAL或者回滚这么一说，所以即使是一个最简单的二阶段提交的策略都很难实现，而且即使实现了，性能也没有保证。所以在Codis中使用MSET/MGET其实和你本地开个多线程SET/GET效果一样，只不过是由服务端打包返回罢了，我们加上这个命令的支持只是为了更好的支持以前用Twemproxy的业务。


## 关于 HA

这里的HA分成两部分，一个是proxy层的HA，还有底层Redis的HA。先说proxy层的HA。之前提到过proxy本身是无状态的，所以proxy本身的HA是比较好做的，因为连接到任何一个活着的proxy上都是一样的，在生产环境中，我们使用的是jodis，这个是我们开发的一个jedis连接池，很简单，就是监听zk上面的存活proxy列表，挨个返回jedis对象，达到负载均衡和HA的效果。也有朋友在生产环境中使用LVS和HA Proxy来做负载均衡，这也是可以的。 Redis本身的HA，这里的Redis指的是codis底层的各个server group的master，在一开始的时候codis本来就没有将这部分的HA设计进去，因为Redis在挂掉后，如果直接将slave提升上来的话，可能会造成数据不一致的情况，因为有新的修改可能在master中还没有同步到slave上，这种情况下需要管理员手动的操作修复数据。后来我们发现这个需求确实比较多的朋友反映，于是我们开发了一个简单的ha工具：codis-ha，用于监控各个server group的master的存活情况，如果某个master挂掉了，会直接提升该group的一个slave成为新的master。

## Codis 分片原理

在Codis中，Codis会把所有的key分成1024个槽，这1024个槽对应着的就是Redis的集群，这个在Codis中是会在内存中维护着这1024个槽与Redis实例的映射关系。这个槽是可以配置，可以设置成 2048 或者是4096个。看你的Redis的节点数量有多少，偏多的话，可以设置槽多一些。

Codis中key的分配算法，先是把key进行CRC32 后，得到一个32位的数字，然后再hash%1024后得到一个余数，这个值就是这个key对应着的槽，这槽后面对应着的就是redis的实例。(可以思考一下，为什么Codis很多命令行不支持，例如KEYS操作)

## Codis之间的槽位同步

思考一个问题：如果这个Codis节点只在自己的内存里面维护着槽位与实例的关系,那么它的槽位信息怎么在多个实例间同步呢？

Codis把这个工作交给了ZooKeeper来管理，当Codis的Codis Dashbord 改变槽位的信息的时候，其他的Codis节点会监听到ZooKeeper的槽位变化，会及时同步过来。如图：

![](/images/codis/zookeeper)

## Codis中的扩容

思考一个问题：在Codis中增加了Redis节点后,槽位的信息怎么变化，原来的key怎么迁移和分配？如果在扩容的时候，这个时候有新的key进来，Codis的处理策略是怎么样的？

因为Codis是一个代理中间件，所以这个当需要扩容Redis实例的时候，可以直接增加redis节点。在槽位分配的时候，可以手动指定Codis Dashbord来为新增的节点来分配特定的槽位。
在Codis中实现了自定义的扫描指令SLOTSSCAN，可以扫描指定的slot下的所有的key，将这些key迁移到新的Redis的节点中(话外语：这个是Codis定制化的其中一个好处)。
首先，在迁移的时候，会在原来的Redis节点和新的Redis里都保存着迁移的槽位信息，在迁移的过程中，如果有key打进将要迁移或者正在迁移的旧槽位的时候，这个时候Codis的处理机制是，先是将这个key强制迁移到新的Redis节点中，然后再告诉Codis,下次如果有新的key的打在这个槽位中的话，那么转发到新的节点。

## 自动均衡策略

面对着上面讲的迁移策略，如果有成千上万个节点新增进来，都需要我们手动去迁移吗？那岂不是得累死啊。当然，Codis也是考虑到了这一点，所以提供了自动均衡策略。自动均衡策略是这样的，Codis 会在机器空闲的时候，观察Redis中的实例对应着的slot数，如果不平衡的话就会自动进行迁移。


## 引用

https://juejin.im/post/5c132b076fb9a04a08218eef

https://blog.csdn.net/shmiluwei/article/details/51958359


