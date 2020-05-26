---
layout: post
title:  SQL查找是否"存在"，别再count了，很耗费时间的
categories: SQL count limit
tags:  SQL count limit
author: wenzhilee77
---

根据某一条件从数据库表中查询 『有』与『没有』，只有两种状态，那为什么在写SQL的时候，还要SELECT count(*) 呢？

无论是刚入道的程序员新星，还是精湛沙场多年的程序员老白，都是一如既往的count

# 目前多数人的写法

多次REVIEW代码时，发现如现现象：

业务代码中，需要根据一个或多个条件，查询是否存在记录，不关心有多少条记录。普遍的SQL及代码写法如下

```sql
##### SQL写法:
SELECT count(*) FROM table WHERE a = 1 AND b = 2

##### Java写法:
int nums = xxDao.countXxxxByXxx(params);
if ( nums > 0 ) {
		//当存在时，执行这里的代码
} else {
		//当不存在时，执行这里的代码
}
```

# 优化方案

推荐写法如下：

```sql
##### SQL写法:
SELECT 1 FROM table WHERE a = 1 AND b = 2 LIMIT 1

##### Java写法:
Integer exist = xxDao.existXxxxByXxx(params);
if ( exist != NULL ) {
		//当存在时，执行这里的代码
} else {
		//当不存在时，执行这里的代码
}
```

SQL不再使用count，而是改用 LIMIT 1，让数据库查询时遇到一条就返回，不要再继续查找还有多少条了

业务代码中直接判断是否非空即可

# 参考

https://mp.weixin.qq.com/s/4vFoie5BHgWqrkBmz8jrhQ