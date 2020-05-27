---
layout: post
title:  Insert into select语句把生产服务器炸了
categories: MySQL Insert Select
tags:  MySQL Insert Select
author: wenzhilee77
---

# 事故发生的经过

由于数据数据库中order_today数据量过大，当时好像有700W了并且每天在以30W的速度增加。所以上司命令xxx将order_today内的部分数据迁移到order_record中，并将order_today中的数据删除。这样来降低order_today表中的数据量。

由于考虑到会占用数据库I/O，为了不影响业务，计划是9:00以后开始迁移，但是xxx在8:00的时候，尝试迁移了少部分数据(1000条)，觉得没啥问题，就开始考虑大批量迁移。

* 在迁移的过程中，应急群是先反应有小部分用户出现支付失败，随后反应大批用户出现支付失败的情况，以及初始化订单失败的情况，同时腾讯也开始报警。
* 然后xxx就慌了，立即停止了迁移。

本以为停止迁移就就可以恢复了，但是并没有。后面发生的你们可以脑补一下。

# 事故还原

在本地建立一个精简版的数据库，并生成了100w的数据。模拟线上发生的情况。

## 建立表结构

```sql
CREATE TABLE `order_today` (
`id` varchar(32) NOT NULL COMMENT '主键',
`merchant_id` varchar(32) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '商户编号',
`amount` decimal(15,2) NOT NULL COMMENT '订单金额',
`pay_success_time` datetime NOT NULL COMMENT '支付成功时间',
`order_status` varchar(10) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL COMMENT '支付状态  S：支付成功、F：订单支付失败',
`remark` varchar(100) CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT NULL COMMENT '备注',
`create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
`update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '修改时间 -- 修改时自动更新',
  PRIMARY KEY (`id`) USING BTREE,
KEY `idx_merchant_id` (`merchant_id`) USING BTREE COMMENT '商户编号'
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

```sql
CREATE TABLE order_record like order_today;
```

## 模拟迁移

把8号之前的数据都迁移到order_record表中去。

```sql
INSERT INTO order_record SELECT
    * 
FROM
    order_today 
WHERE
    pay_success_time < '2020-03-08 00:00:00';
```

在navicat中运行迁移的sql,同时开另个一个窗口插入数据，模拟下单。

从上面可以发现一开始能正常插入，但是后面突然就卡住了，并且耗费了23s才成功，然后才能继续插入。这个时候已经迁移成功了，所以能正常插入了。

## 出现的原因

在默认的事务隔离级别下：insert into order_record select * from order_today 加锁规则是：order_record表锁，order_today逐步锁（扫描一个锁一个）。

分析执行过程。

![](/images/insert_into/001.jpg)

通过观察迁移sql的执行情况你会发现order_today是全表扫描，也就意味着在执行insert into select from 语句时，mysql会从上到下扫描order_today内的记录并且加锁，这样一来不就和直接锁表是一样了。

这也就可以解释，为什么一开始只有少量用户出现支付失败，后续大量用户出现支付失败，初始化订单失败等情况，因为一开始只锁定了少部分数据，没有被锁定的数据还是可以正常被修改为正常状态。由于锁定的数据越来越多，就导致出现了大量支付失败。最后全部锁住，导致无法插入订单，而出现初始化订单失败。

## 解决方案

由于查询条件会导致order_today全表扫描，什么能避免全表扫描呢，很简单嘛，给pay_success_time字段添加一个idx_pay_suc_time索引就可以了，由于走索引查询，就不会出现扫描全表的情况而锁表了，只会锁定符合条件的记录。

## 最终的sql

```sql
INSERT INTO order_record SELECT
    * 
FROM
    order_today FORCE INDEX (idx_pay_suc_time)
WHERE
    pay_success_time <= '2020-03-08 00:00:00';
```

执行过程

![](/images/insert_into/002.jpg)

## 总结

使用insert into tablA select * from tableB语句时，一定要确保tableB后面的where，order或者其他条件，都需要有对应的索引，来避免出现tableB全部记录被锁定的情况。



# 参考

https://mp.weixin.qq.com/s/EyNeMpfL6a54t9IZ0HbfMQ