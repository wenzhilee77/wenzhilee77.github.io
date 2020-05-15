---
layout: post
title: 分布式锁的几种实现和比较
categories: Java Distributed-Lock
tags:  Java Distributed-Lock
author: wenzhilee77
---

对于锁大家肯定不会陌生，在Java中synchronized关键字和ReentrantLock可重入锁在我们的代码中是经常见的，
一般我们用其在多线程环境中控制对资源的并发访问，但是随着分布式的快速发展，本地的加锁往往不能满足我们的需要，
在我们的分布式环境中上面加锁的方法就会失去作用。于是人们为了在分布式环境中也能实现本地锁的效果，也是纷纷各出其招，
今天让我们来聊一聊一般分布式锁实现的套路。

# 分布式锁

## 为何需要分布式锁

Martin Kleppmann是英国剑桥大学的分布式系统的研究员，之前和Redis之父Antirez进行过关于RedLock(红锁，后续有讲到)
是否安全的激烈讨论。Martin认为一般我们使用分布式锁有两个场景:

* 效率:使用分布式锁可以避免不同节点重复相同的工作，这些工作会浪费资源。比如用户付了钱之后有可能不同节点会发出多封短信。
* 正确性:加分布式锁同样可以避免破坏正确性的发生，如果两个节点在同一条数据上面操作，
比如多个节点机器对同一个订单操作不同的流程有可能会导致该笔订单最后状态出现错误，造成损失。

## 分布式锁的一些特点

当我们确定了在不同节点上需要分布式锁，那么我们需要了解分布式锁到底应该有哪些特点:

* 互斥性:和我们本地锁一样互斥性是最基本，但是分布式锁需要保证在不同节点的不同线程的互斥。
* 可重入性:同一个节点上的同一个线程如果获取了锁之后那么也可以再次获取这个锁。
* 锁超时:和本地锁一样支持锁超时，防止死锁。
* 高效，高可用:加锁和解锁需要高效，同时也需要保证高可用防止分布式锁失效，可以增加降级。
* 支持阻塞和非阻塞:和ReentrantLock一样支持lock和trylock以及tryLock(long timeOut)。
* 支持公平锁和非公平锁(可选):公平锁的意思是按照请求加锁的顺序获得锁，非公平锁就相反是无序的。这个一般来说实现的比较少。

## 常见的分布式锁

我们了解了一些特点之后，我们一般实现分布式锁有以下几个方式:

* MySQL
* ZK
* Redis
* 自研分布式锁:如谷歌的Chubby。

# 分布式锁的实现原理

## MySQL分布式锁

### 数据库悲观锁实现

以「悲观的心态」操作资源，无法获得锁成功，就一直阻塞着等待。

1. 有一张资源锁表

```sql
CREATE TABLE `resource_lock` (
`id` int(4) NOT NULL AUTO_INCREMENT COMMENT '主键',
`resource_name` varchar(64) NOT NULL DEFAULT '' COMMENT '锁定的资源名',
`owner` varchar(64) NOT NULL DEFAULT '' COMMENT '锁拥有者',
`desc` varchar(1024) NOT NULL DEFAULT '备注信息',
`update_time` timestamp NOT NULL DEFAULT '' COMMENT '保存数据时间，自动生成',
  PRIMARY KEY (`id`),
UNIQUE KEY `uidx_resource_name` (`resource_name `) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='锁定中的资源';
```

resource_name 锁资源名称必须有唯一索引。

2. 使用姿势

必须添加事务，查询和更新操作保证原子性，在一个事务里完成。

伪代码实现：

```java
@Transaction
public void lock(String name) {
   ResourceLock rlock = exeSql("select * from resource_lock where resource_name = name for update");
if (rlock == null) {
           exeSql("insert into resource_lock(reosurce_name,owner,count) values (name, 'ip',0)");
     }
}
```

使用 for update 锁定的资源。如果执行成功，会立即返回，执行插入数据库，后续再执行一些其他业务逻辑，直到事务提交，执行结束；如果执行失败，就会一直阻塞着。

你也可以在数据库客户端工具上测试出来这个效果，当在一个终端执行了 for update，不提交事务。在另外的终端上执行相同条件的 for update，会一直卡着，转圈圈...

虽然也能实现分布式锁的效果，但是会存在性能瓶颈。

3. 悲观锁优缺点

优点：简单易用，好理解，保障数据强一致性。

缺点一大堆，罗列一下：

1）在 RR 事务级别，select 的 for update 操作是基于间隙锁（gap lock） 实现的，是一种悲观锁的实现方式，所以存在阻塞问题。

2）高并发情况下，大量请求进来，会导致大部分请求进行排队，影响数据库稳定性，也会耗费服务的CPU等资源。

当获得锁的客户端等待时间过长时，会提示：

[40001][1205] Lock wait timeout exceeded; try restarting transaction

高并发情况下，也会造成占用过多的应用线程，导致业务无法正常响应。

3）如果优先获得锁的线程因为某些原因，一直没有释放掉锁，可能会导致死锁的发生。

4）锁的长时间不释放，会一直占用数据库连接，可能会将数据库连接池撑爆，影响其他服务。

5) MySql数据库会做查询优化，即便使用了索引，优化时发现全表扫效率更高，则可能会将行锁升级为表锁，此时可能就更悲剧了。

6）不支持可重入特性，并且超时等待时间是全局的，不能随便改动。

### 数据库乐观锁实现

乐观锁，以「乐观的心态」来操作共享资源，无法获得锁成功，没关系过一会重试一下看看呗，再不行就直接退出，尝试一定次数还是不行？也可以以后再说，不用一直阻塞等着。

1. 有一张资源表

为表添加一个字段，版本号或者时间戳都可以。通过版本号或者时间戳，来保证多线程同时间操作共享资源的有序性和正确性。

```sql
CREATE TABLE `resource` (
`id` int(4) NOT NULL AUTO_INCREMENT COMMENT '主键',
`resource_name` varchar(64) NOT NULL DEFAULT '' COMMENT '资源名',
`share` varchar(64) NOT NULL DEFAULT '' COMMENT '状态',
`version` int(4) NOT NULL DEFAULT '' COMMENT '版本号',
`desc` varchar(1024) NOT NULL DEFAULT '备注信息',
`update_time` timestamp NOT NULL DEFAULT '' COMMENT '保存数据时间，自动生成',
  PRIMARY KEY (`id`),
UNIQUE KEY `uidx_resource_name` (`resource_name `) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='资源';
```

2. 使用姿势

伪代码实现：

```java
Resrouce resource = exeSql("select * from resource where resource_name = xxx");
boolean succ = exeSql("update resource set version= 'newVersion' ... where resource_name = xxx and version = 
'oldVersion'");

if (!succ) {
    // 发起重试
}
```

实际代码中可以写个while循环不断重试，版本号不一致，更新失败，重新获取新的版本号，直到更新成功。

3. 乐观锁优缺点

优点：简单易用，保障数据一致性。

缺点：

1）加行锁的性能上有一定的开销

2）高并发场景下，线程内的自旋操作 会耗费一定的CPU资源。

另外，比如在更新数据状态的一些场景下，不考虑幂等性的情况下，可以直接利用 行锁 来保证数据一致性，示例：update table set state = 1 where id = xxx and state = 0;

乐观锁就类似 CAS Compare And Swap更新机制

### 锁超时

我们有可能会遇到我们的机器节点挂了，那么这个锁就不会得到释放，我们可以启动一个定时任务，
通过计算一般我们处理任务的一般的时间，比如是5ms，那么我们可以稍微扩大一点，
当这个锁超过20ms没有被释放我们就可以认定是节点挂了然后将其直接释放。

### MySQL小结

* 适用场景: Mysql分布式锁一般适用于资源不存在数据库，如果数据库存在比如订单，那么可以直接对这条数据加行锁，
不需要我们上面多的繁琐的步骤，比如一个订单，那么我们可以用select * from order_table where id = 'xxx' for update
进行加行锁，那么其他的事务就不能对其进行修改。
* 优点:理解起来简单，不需要维护额外的第三方中间件(比如Redis,Zk)。
* 缺点:虽然容易理解但是实现起来较为繁琐，需要自己考虑锁超时，加事务等等。性能局限于数据库，一般对比缓存来说性能较低。对于高并发的场景并不是很适合。

### 乐观锁

前面我们介绍的都是悲观锁，这里想额外提一下乐观锁，在我们实际项目中也是经常实现乐观锁，因为我们加行锁的性能消耗比较大，
通常我们会对于一些竞争不是那么激烈，但是其又需要保证我们并发的顺序执行使用乐观锁进行处理，我们可以对我们的表加一个版本号字段，
那么我们查询出来一个版本号之后，update或者delete的时候需要依赖我们查询出来的版本号，判断当前数据库和查询出来的版本号是否相等，
如果相等那么就可以执行，如果不等那么就不能执行。这样的一个策略很像我们的CAS(Compare And Swap),比较并交换是一个原子操作。
这样我们就能避免加select * for update行锁的开销。

## ZooKeeper

Zookeeper 是一种提供「分布式服务协调」的中心化服务，是以 Paxos 算法为基础实现的。
Zookeeper数据节点和文件目录类似，同时具有Watch机制，基于这两个特性，得以实现分布式锁功能。

数据节点：

顺序临时节点：Zookeeper 提供一个多层级的节点命名空间（节点称为 Znode），每个节点都用一个以斜杠（/）分隔的路径来表示，而且每个节点都有父节点（根节点除外），非常类似于文件系统。

节点类型可以分为持久节点（PERSISTENT ）、临时节点（EPHEMERAL），每个节点还能被标记为有序性（SEQUENTIAL），一旦节点被标记为有序性，那么整个节点就具有顺序自增的特点。

一般我们可以组合这几类节点来创建我们所需要的节点，例如，创建一个持久节点作为父节点，在父节点下面创建临时节点，并标记该临时节点为有序性。

Watch 机制：

Zookeeper 还提供了另外一个重要的特性，Watcher（事件监听器）。

ZooKeeper 允许用户在指定节点上注册一些 Watcher，并且在一些特定事件触发的时候，ZooKeeper 服务端会将事件通知给用户。

首先，我们需要建立一个父节点，节点类型为持久节点（PERSISTENT）如图中的 /locks/lock_name1 节点 ，每当需要访问共享资源时，就会在父节点下建立相应的顺序子节点，节点类型为临时节点（EPHEMERAL），且标记为有序性（SEQUENTIAL），并且以临时节点名称 + 父节点名称 + 顺序号组成特定的名字，如图中的 /0000000001 /0000000002 /0000000003 作为临时有序节点。

在建立子节点后，对父节点下面的所有以临时节点名称 name 开头的子节点进行排序，判断刚刚建立的子节点顺序号是否是最小的节点，如果是最小节点，则获得锁。

如果不是最小节点，则阻塞等待锁，并且获得该节点的上一顺序节点，为其注册监听事件，等待节点对应的操作获得锁。当调用完共享资源后，删除该节点，关闭 zk，进而可以触发监听事件，释放该锁。

```java
// 加锁
InterProcessMutex lock = new InterProcessMutex(client, lockPath);
if ( lock.acquire(maxWait, waitUnit) )
{
try 
    {
// do some work inside of the critical section here
    }
finally
    {
lock.release();
    }
}
publicvoidacquire() throws Exception
    {
if ( !internalLock(-1, null) )
            {
throw new IOException("Lost connection while trying to acquire lock: " + basePath);
            }
    }

private boolean internalLock(long time, TimeUnit unit) throws Exception
    {
/*
                 Note on concurrency: a given lockData instance
                 can be only acted on by a single thread so locking isn't necessary
            */

            Thread currentThread = Thread.currentThread();

            LockData lockData = threadData.get(currentThread);
if ( lockData != null )
            {
// re-entering
                    lockData.lockCount.incrementAndGet();
return true;
            }

            String lockPath = internals.attemptLock(time, unit, getLockNodeBytes());
if ( lockPath != null )
            {
                    LockData newLockData = new LockData(currentThread, lockPath);
                    threadData.put(currentThread, newLockData);
return true;
            }

            return false;
    }
// ... 其他代码略
```

InterProcessMutex 是 Curator 实现的可重入锁，可重入锁源码过程分析：

加锁流程：

1）可重入锁记录在 ConcurrentMap threadData 这个 Map 里面。,>

2）如果 threadData.get(currentThread) 是有值的那么就证明是可重入锁，然后记录就会加 1。

3）资源目录下创建一个节点：比如这里创建一个 /0000000002 这个节点，这个节点需要设置为 EPHEMERAL_SEQUENTIAL 也就是临时节点并且有序。

4）获取当前目录下所有子节点，判断自己的节点是否是最小的节点。

5）如果是最小的节点，则获取到锁。如果不是最小的节点，则证明前面已经有人获取到锁了，那么需要获取自己节点的前一个节点。

6）节点 /0000000002 的前一个节点是 /0000000001，我们获取到这个节点之后，再上面注册 Watcher，Watcher 调用的是 object.notifyAll()，用来解除阻塞。

7）object.wait(timeout) 或 object.wait() 进行阻塞等待

解锁流程：

1）如果可重入锁次数减1后，加锁次数不为 0 直接返回，减1后加锁次数为0，继续。

2）删除当前节点。

3）删除 threadDataMap 里面的可重入锁的数据。

### ZK小结

* 优点:ZK可以不需要关心锁超时时间，实现起来有现成的第三方包，比较方便，并且支持读写锁，ZK获取锁会按照加锁的顺序，所以其是公平锁。对于高可用利用ZK集群进行保证。
* 缺点:ZK需要额外维护，增加维护成本，性能和Mysql相差不大，依然比较差。并且需要开发人员了解ZK是什么。

## Redis

大家在网上搜索分布式锁，恐怕最多的实现就是Redis了，Redis因为其性能好，实现起来简单所以让很多人都对其十分青睐。

### Redis分布式锁简单实现

Redis 2.6.12 之前的版本中采用 setnx + expire 方式实现分布式锁，示例代码如下所示：

```java
publicstaticbooleanlock(Jedis jedis, String lockKey, String requestId, int expireTime) {
        Long result = jedis.setnx(lockKey, requestId);
//设置锁
if (result == 1) {
//获取锁成功
//若在这里程序突然崩溃，则无法设置过期时间，将发生死锁
//通过过期时间删除锁
            jedis.expire(lockKey, expireTime);
return true;
        }
return false;
    }
```

如果 lockKey 存在，则返回失败，否则返回成功。设置成功之后，为了能在完成同步代码之后成功释放锁，方法中使用 expire() 方法给 lockKey 设置一个过期时间，确认 key 值删除，避免出现锁无法释放，导致下一个线程无法获取到锁，即死锁问题。

但是 setnx + expire 两个命令放在程序里执行，不是原子操作，容易出事。

如果程序设置锁之后，此时，在设置过期时间之前，程序崩溃了，如果 lockKey 没有设置上过期时间，将会出现死锁问题。

解决以上问题 ，有两个办法：

1）方式一：lua脚本

我们也可以通过 Lua 脚本来实现锁的设置和过期时间的原子性，再通过 jedis.eval() 方法运行该脚本：

```
// 加锁脚本，KEYS[1] 要加锁的key，ARGV[1]是UUID随机值，ARGV[2]是过期时间
private static final String SCRIPT_LOCK = "if redis.call('setnx', KEYS[1], ARGV[1]) == 1 then redis.call('pexpire', KEYS[1], ARGV[2]) return 1 else return 0 end";
// 解锁脚本，KEYS[1]要解锁的key，ARGV[1]是UUID随机值
private static final String SCRIPT_UNLOCK = "

if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
```

2）方式二：set原生命令

在 Redis 2.6.12 版本后 SETNX 增加了过期时间参数：

SET lockKey anystring NX PX max-lock-time

程序实现代码如下：

```java
public static boolean lock(Jedis jedis, String lockKey, String requestId, int expireTime) {
        String result = jedis.set(lockKey, requestId, "NX", "PX", expireTime);
if ("OK".equals(result)) {
return true;
        }
return false;
    }
```

虽然 SETNX 方式能够保证设置锁和过期时间的原子性，但是如果我们设置的过期时间比较短，而执行业务时间比较长，
就会存在锁代码块失效的问题，失效后其他客户端也能获取到同样的锁，执行同样的业务，此时可能就会出现一些问题。

我们需要将过期时间设置得足够长，来保证以上问题不会出现，但是设置多长时间合理，也需要依具体业务来权衡。
如果其他客户端必须要阻塞拿到锁，需要设计循环超时等待机制等问题，感觉还挺麻烦的是吧。

### Redis小结

* 优点:对于Redis实现简单，性能对比ZK和Mysql较好。如果不需要特别复杂的要求，那么自己就可以利用setNx进行实现，如果自己需要复杂的需求的话那么可以利用或者借鉴Redission。对于一些要求比较严格的场景来说的话可以使用RedLock。
* 缺点:需要维护Redis集群，如果要实现RedLock那么需要维护更多的集群。


# 参考

