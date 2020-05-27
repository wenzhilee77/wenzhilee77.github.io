---
layout: post
title:  数据库：Mysql中“select ... for update”排他锁分析
categories: MySQL InnoDB Lock
tags:  MySQL InnoDB Lock
author: wenzhilee77
---

# Mysql InnoDB 排他锁

用法： select … for update;

例如：select * from goods where id = 1 for update;

排他锁的申请前提：没有线程对该结果集中的任何行数据使用排他锁或共享锁，否则申请会阻塞。

for update仅适用于InnoDB，且必须在事务块(BEGIN/COMMIT)中才能生效。在进行事务操作时，通过“for update”语句，MySQL会对查询结果集中每行数据都添加排他锁，其他线程对该记录的更新与删除操作都会阻塞。排他锁包含行锁、表锁。

# 场景分析

假设有一张商品表 goods，它包含 id，商品名称，库存量三个字段，表结构如下：

```sql
CREATE TABLE `goods` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(100) DEFAULT NULL,
  `stock` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_name` (`name`) USING HASH
) ENGINE=InnoDB
```

插入如下数据：

```sql
INSERT INTO `goods` VALUES ('1', 'prod11', '1000');
INSERT INTO `goods` VALUES ('2', 'prod12', '1000');
INSERT INTO `goods` VALUES ('3', 'prod13', '1000');
INSERT INTO `goods` VALUES ('4', 'prod14', '1000');
INSERT INTO `goods` VALUES ('5', 'prod15', '1000');
INSERT INTO `goods` VALUES ('6', 'prod16', '1000');
INSERT INTO `goods` VALUES ('7', 'prod17', '1000');
INSERT INTO `goods` VALUES ('8', 'prod18', '1000');
INSERT INTO `goods` VALUES ('9', 'prod19', '1000');
```

# 数据一致性

假设有A、B两个用户同时各购买一件 id=1 的商品，用户A获取到的库存量为 1000，用户B获取到的库存量也为 1000，用户A完成购买后修改该商品的库存量为 999，用户B完成购买后修改该商品的库存量为 999，此时库存量数据产生了不一致。

有两种解决方案：

悲观锁方案：每次获取商品时，对该商品加排他锁。也就是在用户A获取获取 id=1 的商品信息时对该行记录加锁，期间其他用户阻塞等待访问该记录。悲观锁适合写入频繁的场景。

```sql
begin;
select * from goods where id = 1 for update;
update goods set stock = stock - 1 where id = 1;
commit;
```

乐观锁方案：每次获取商品时，不对该商品加锁。在更新数据的时候需要比较程序中的库存量与数据库中的库存量是否相等，如果相等则进行更新，反之程序重新获取库存量，再次进行比较，直到两个库存量的数值相等才进行数据更新。乐观锁适合读取频繁的场景。

```sql
#不加锁获取 id=1 的商品对象
select * from goods where id = 1

begin;
#更新 stock 值，这里需要注意 where 条件 “stock = cur_stock”，只有程序中获取到的库存量与数据库中的库存量相等才执行更新
update goods set stock = stock - 1 where id = 1 and stock = cur_stock;
commit;
```

如果我们需要设计一个商城系统，该选择以上的哪种方案呢？

查询商品的频率比下单支付的频次高，基于以上我可能会优先考虑第二种方案（当然还有其他的方案，这里只考虑以上两种方案）。

# 行锁与表锁

* InnoDB行锁是通过给索引上的索引项加锁来实现的，只有通过索引条件检索数据，InnoDB才使用行级锁，否则，InnoDB将使用表锁。
* 由于MySQL的行锁是针对索引加的锁，不是针对记录加的锁，所以虽然是访问不同行的记录，但是如果是使用相同的索引键，是会出现锁冲突的。应用设计的时候要注意这一点。
* 当表有多个索引的时候，不同的事务可以使用不同的索引锁定不同的行，另外，不论是使用主键索引、唯一索引或普通索引，InnoDB都会使用行锁来对数据加锁。
* 即便在条件中使用了索引字段，但是否使用索引来检索数据是由MySQL通过判断不同执行计划的代价来决定的，如果MySQL认为全表扫描效率更高，比如对一些很小的表，它就不会使用索引，这种情况下InnoDB将使用表锁，而不是行锁。因此，在分析锁冲突时，别忘了检查SQL的执行计划，以确认是否真正使用了索引。
* 检索值的数据类型与索引字段不同，虽然MySQL能够进行数据类型转换，但却不会使用索引，从而导致InnoDB使用表锁。通过用explain检查两条SQL的执行计划，我们可以清楚地看到了这一点。


# 参考

https://blog.csdn.net/claram/article/details/54023216