---
layout: post
title:  MySQL 当前读/快照度及一致性视图
categories: MySQL ReadView MVCC 
tags:  Spring Transactional Invalid
author: wenzhilee77
---

# MVCC

多版本控制: 指的是一种提高并发的技术。最早的数据库系统，只有读读之间可以并发，读写，写读，写写都要阻塞。引入多版本之后，只有写写之间相互阻塞，其他三种操作都可以并行，这样大幅度提高了 InnoDB 的并发度。在内部实现中，与 Postgres 在数据行上实现多版本不同，InnoDB 是在 undolog 中实现的，通过 undolog 可以找回数据的历史版本。找回的数据历史版本可以提供给用户读(按照隔离级别的定义，有些读请求只能看到比较老的数据版本)，也可以在回滚的时候覆盖数据页上的数据。在 InnoDB 内部中，会记录一个全局的活跃读写事务数组，其主要用来判断事务的可见性。

可以看到 MVCC 主要用来提高并发，还可以用来读取老版本数据。

在学习 MVCC 原理之前，首先我们需要了解 MySQL 记录结构。

![](/images/readView/001.jpg)

如上图所示，account 表一行记录，除了真实数据之外，还会存在三个隐藏字段，用来记录额外信息。
* DB_TRX_ID:事务 id。
* DB_ROLL_PTR: 回滚指针，指向 undolog。
* ROW_ID：行 id,与此次无关。

MySQL InnoDB 里面每个事务都会有一个唯一事务 ID，它在事务开始的时候会跟 InnoDB 的事务系统申请的，并且严格按照顺序递增的。

每次事务更新数据时，将会生成一个新的数据版本，然后会把当前的事务 id 赋值给当前记录的 DB_TRX_ID。并且数据更新记录（1,1000---->1,900）将会记录在 undo log（回滚日志）中，然后使用当前记录的 DB_ROLL_PTR 指向 und olog。

这样 MySQL 就可以通过 DB_ROLL_PTR 找到 undolog 推导出之前版本记录内容。

查找过程如下：

![](/images/readView/002.jpg)

若需要知道 V1 版本记录，首先根据当前版本 V3 的 DB_ROLL_PTR 找到 undolog，然后根据 undolog 内容，计算出上一个版本 V2。以此类推，最终找到 V1 这个版本记录。

V1，V2 并不是物理记录，没有真实存在，仅仅具有逻辑意义。

一行数据记录可能同时存在多个版本，但并不是所有记录都能对当前事务可见。不然上面 t5 就可能查询到最新的数据。所以查找数据版本时候 MySQL 必须判断数据版本是否对当前事务可见。


# ReadView

ReadView所解决的问题是使用READ COMMITTED和REPEATABLE READ隔离级别的事务中，不能读到未提交的记录，这需要判断一下版本链中的哪个版本是当前事务可见的。

ReadView中主要包含4个比较重要的内容：
* m_ids：表示在生成ReadView时当前系统中活跃的读写事务的事务id列表。
* min_trx_id：表示在生成ReadView时当前系统中活跃的读写事务中最小的事务id，也就是m_ids中的最小值。
* max_trx_id：表示生成ReadView时系统中应该分配给下一个事务的id值。
* creator_trx_id：表示生成该ReadView的事务的事务id。

![](/images/readView/003.jpg)

## ReadView是如何工作的？

有了这些信息，这样在访问某条记录时，只需要按照下边的步骤判断记录的某个版本是否可见：
* 如果被访问版本的trx_id属性值与ReadView中的creator_trx_id值相同，意味着当前事务在访问它自己修改过的记录，所以该版本可以被当前事务访问。
* 如果被访问版本的trx_id属性值小于ReadView中的min_trx_id值，表明生成该版本的事务在当前事务生成ReadView前已经提交，所以该版本可以被当前事务访问。
* 如果被访问版本的trx_id属性值大于ReadView中的max_trx_id值，表明生成该版本的事务在当前事务生成ReadView后才开启，所以该版本不可以被当前事务访问。
* 如果被访问版本的trx_id属性值在ReadView的min_trx_id和max_trx_id之间，那就需要判断一下trx_id属性值是不是在m_ids列表中，如果在，说明创建ReadView时生成该版本的事务还是活跃的，该版本不可以被访问；如果不在，说明创建ReadView时生成该版本的事务已经被提交，该版本可以被访问。
* 如果某个版本的数据对当前事务不可见的话，那就顺着版本链找到下一个版本的数据，继续按照上边的步骤判断可见性，依此类推，直到版本链中的最后一个版本。如果最后一个版本也不可见的话，那么就意味着该条记录对该事务完全不可见，查询结果就不包含该记录。

在MySQL中，READ COMMITTED和REPEATABLE READ隔离级别的的一个非常大的区别就是它们生成ReadView的时机不同。

## 示例解释

```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `k` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;
insert into t(id, k) values(1,1) ;
```

![](/images/readView/004.jpg)

在这个例子中，我们做如下假设:
* 事务A、B的版本号分别是100、200，且当前系统里只有这3个事务；
* 三个事务开始前，(1,1）这一行数据的row trx_id是90。

![](/images/readView/005.jpg)

READ COMMITTED —— 每次读取数据前都生成一个ReadView

继续上面的例子，假设现在有一个使用READ COMMITTED隔离级别的事务开始执行：

```sql
# 使用READ COMMITTED隔离级别的事务
BEGIN;

# SELECT1：Transaction 100、200未提交
select k from t where id=1 ; # 得到值为1
```

这个SELECT1的执行过程如下：
* 在执行SELECT语句时会先生成一个ReadView，ReadView的m_ids列表的内容就是[100, 200]，min_trx_id为100，max_trx_id为201，creator_trx_id为0。
* 然后从版本链中挑选可见的记录，最新的版本trx_id值为200，在m_ids列表内，所以不符合可见性要求
* 下一个版本的trx_id值也为100，也在m_ids列表内，所以也不符合要求，继续跳到下一个版本。
* 下一个版本的trx_id值为90，小于ReadView中的min_trx_id值100，所以这个版本是符合要求的。
* 之后，我们把事务B的事务提交一下，然后再到刚才使用READ COMMITTED隔离级别的事务中继续查找，如下：

```sql
# 使用READ COMMITTED隔离级别的事务
BEGIN;

# SELECT1：Transaction 100、200均未提交
SELECT * FROM hero WHERE number = 1; # 得到值为1

# SELECT2：Transaction 200提交，Transaction 100未提交
SELECT * FROM hero WHERE number = 1; # 得到值为2
```

这个SELECT2的执行过程如下：
* 在执行SELECT语句时会又会单独生成一个ReadView，该ReadView的m_ids列表的内容就是[100]（事务id为200的那个事务已经提交了，所以再次生成快照时就没有它了），min_trx_id为100，max_trx_id为201，creator_trx_id为0。
* 然后从版本链中挑选可见的记录，从图中可以看出，最新版本trx_id值为100，在m_ids列表内，所以不符合可见性要求
* 下一个版本的trx_id值为200,小于max_trx_id，并且不在m_ids列表中，所以可见，返回的值为2

REPEATABLE READ —— 在第一次读取数据时生成一个ReadView

假设现在有一个使用REPEATABLE READ隔离级别的事务开始执行：

```sql
# 使用REPEATABLE READ隔离级别的事务
BEGIN;

# SELECT1：Transaction 100、200未提交
SELECT * FROM hero WHERE number = 1; # 得到值为1
```

这个SELECT1的执行过程如下：
* 在执行SELECT语句时会先生成一个ReadView，ReadView的m_ids列表的内容就是[100, 200]，min_trx_id为100，max_trx_id为201，creator_trx_id为0。
* 然后从版本链中挑选可见的记录，该版本的trx_id值为100，在m_ids列表内，所以不符合可见性要求
* 下一个版本该版本的trx_id值为200，也在m_ids列表内，所以也不符合要求，继续跳到下一个版本。
* 下一个版本的trx_id值为90，小于ReadView中的min_trx_id值100，所以这个版本是符合要求的。
之后，我们把事务B的事务提交一下
然后再到刚才使用REPEATABLE READ隔离级别的事务中继续查找:

```sql
# 使用REPEATABLE READ隔离级别的事务
BEGIN;

# SELECT1：Transaction 100、200均未提交
SELECT * FROM hero WHERE number = 1; # 得到值为1

# SELECT2：Transaction 200提交，Transaction 100未提交
SELECT * FROM hero WHERE number = 1; # 得到值为1
```

这个SELECT2的执行过程如下：
* 因为当前事务的隔离级别为REPEATABLE READ，而之前在执行SELECT1时已经生成过ReadView了，所以此时直接复用之前的ReadView，之前的ReadView的m_ids列表的内容就是[100, 200]，min_trx_id为100，max_trx_id为201，creator_trx_id为0。
* 然后从版本链中挑选可见的记录，该版本的trx_id值为100，在m_ids列表内，所以不符合可见性要求
* 下一个版本该版本的trx_id值为200，也在m_ids列表内，所以也不符合要求，继续跳到下一个版本。
* 下一个版本的trx_id值为90，小于ReadView中的min_trx_id值100，所以这个版本是符合要求的。

# 当前读与快照读

## 当前读

```sql
select...lock in share mode (共享读锁)
select...for update
update , delete , insert
```

当前读, 读取的是最新版本, 并且对读取的记录加锁, 阻塞其他事务同时改动相同记录，避免出现安全问题。

例如，假设要update一条记录，但是另一个事务已经delete这条数据并且commit了，如果不加锁就会产生冲突。所以update的时候肯定要是当前读，得到最新的信息并且锁定相应的记录。

### 当前读的实现方式：next-key锁(行记录锁+Gap间隙锁)

间隙锁：只有在Read Repeatable、Serializable隔离级别才有，就是锁定范围空间的数据，假设id有3,4,5，锁定id>3的数据，是指的4，5及后面的数字都会被锁定，因为此时如果不锁定没有的数据，例如当加入了新的数据id=6，就会出现幻读，间隙锁避免了幻读。
* 对主键或唯一索引，如果当前读时，where条件全部精确命中(=或者in)，这种场景本身就不会出现幻读，所以只会加行记录锁。
* 没有索引的列，当前读操作时，会加全表gap锁，生产环境要注意。
* 非唯一索引列，如果where条件部分命中(>、<、like等)或者全未命中，则会加附近Gap间隙锁。例如，某表数据如下，非唯一索引2,6,9,9,11,15。如下语句要操作非唯一索引列9的数据，gap锁将会锁定的列是(6,11]，该区间内无法插入数据。

![](/images/readView/006.jpg)

## 快照读

单纯的select操作，不包括上述 select ... lock in share mode, select ... for update。　　　　

Read Committed隔离级别：每次select都生成一个快照读。

Read Repeatable隔离级别：开启事务后第一个select语句才是快照读的地方，而不是一开启事务就快照读。

## 快照读的实现方式：undolog和多版本并发控制MVCC

下图右侧绿色的是数据：一行数据记录，主键ID是10，name='Jack'，age=10,  被update更新set为name= 'Tom'，age=23。

事务会先使用“排他锁”锁定改行，将该行当前的值复制到undo log中，然后再真正地修改当前行的值，最后填写事务的DB_TRX_ID，使用回滚指针DB_ROLL_PTR指向undo log中修改前的行DB_ROW_ID。

![](/images/readView/007.jpg)

DB_TRX_ID: 6字节DB_TRX_ID字段，表示最后更新的事务id(update,delete,insert)。此外，删除在内部被视为更新，其中行中的特殊位被设置为将其标记为已软删除。
DB_ROLL_PTR: 7字节回滚指针，指向前一个版本的undolog记录，组成undo链表。如果更新了行，则撤消日志记录包含在更新行之前重建行内容所需的信息。
DB_ROW_ID: 6字节的DB_ROW_ID字段，包含一个随着新行插入而单调递增的行ID, 当由innodb自动产生聚集索引时，聚集索引会包括这个行ID的值，否则这个行ID不会出现在任何索引中。如果表中没有主键或合适的唯一索引, 也就是无法生成聚簇索引的时候, InnoDB会帮我们自动生成聚集索引, 聚簇索引会使用DB_ROW_ID的值来作为主键; 如果表中有主键或者合适的唯一索引, 那么聚簇索引中也就不会包含 DB_ROW_ID了 。  

其它：insert undo log只在事务回滚时需要, 事务提交就可以删掉了。update undo log包括update 和 delete , 回滚和快照读 都需要。




# 参考

https://mp.weixin.qq.com/s/efWDxdqO4TV2IY7x8LBx3w
https://www.cnblogs.com/luozhiyun/p/11216287.html
https://www.cnblogs.com/wwcom123/p/10727194.html
