---
layout: post
title:  MySQL Tree Menu
categories: MySQL Tree Menu
tags:  MySQL Tree Menu
author: wenzhilee77
---

# 树形结构的数据库表Schema设计

## 基本数据

本文列举了一个食品族谱的例子进行讲解，通过类别、颜色和品种组织食品，树形结构图如下：

![](/images/treeMenu/001.gif)

## 继承关系驱动的Schema设计

对树形结构最直观的分析莫过于节点之间的继承关系上，通过显示地描述某一节点的父节点，从而能够建立二维的关系表，则这种方案的Tree表结构通常设计为：{Node_id,Parent_id}，上述数据可以描述为如下图所示：

![](/images/treeMenu/002.gif)

这种方案的优点很明显：设计和实现自然而然，非常直观和方便。缺点当然也是非常的突出：由于直接地记录了节点之间的继承关系，因此对Tree的任何CRUD操作都将是低效的，这主要归根于频繁的“递归”操作，递归过程不断地访问数据库，每次数据库IO都会有时间开销。当然，这种方案并非没有用武之地，在Tree规模相对较小的情况下，我们可以借助于缓存机制来做优化，将Tree的信息载入内存进行处理，避免直接对数据库IO操作的性能开销。

## 基于左右值编码的Schema设计

在基于数据库的一般应用中，查询的需求总要大于删除和修改。为了避免对于树形结构查询时的“递归”过程，基于Tree的前序遍历设计一种全新的无递归查询、无限分组的左右值编码方案，来保存该树的数据。

![](/images/treeMenu/003.gif)

第一次看见这种表结构，相信大部分人都不清楚左值（Lft）和右值（Rgt）是如何计算出来的，而且这种表设计似乎并没有保存父子节点的继承关系。但当你用手指指着表中的数字从1数到18，你应该会发现点什么吧。对，你手指移动的顺序就是对这棵树进行前序遍历的顺序，如下图所示。当我们从根节点Food左侧开始，标记为1，并沿前序遍历的方向，依次在遍历的路径上标注数字，最后我们回到了根节点Food，并在右边写上了18。

![](/images/treeMenu/004.gif)

## 树形结构CRUD算法

### 获取某节点的子孙节点

只需要一条SQL语句，即可返回该节点子孙节点的前序遍历列表，以Fruit为例：

```sql
SELECT* FROM tree WHERE lft BETWEEN 2 AND 11 ORDER BY lft ASC;
```

那么某个节点到底有多少的子孙节点呢？通过该节点的左、右值我们可以将其子孙节点圈进来，则子孙总数 = (右值 – 左值– 1) / 2，以Fruit为例，其子孙总数为：(11 –2 – 1) / 2 = 4。

同时，为了更为直观地展现树形结构，我们需要知道节点在树中所处的层次，通过左、右值的SQL查询即可实现，以Fruit为例：

```sql
SELECT COUNT(*) FROM tree WHERE lft <= 2 AND rgt >=11;
```

### 获取某节点的族谱路径

假定我们要获得某节点的族谱路径，则根据左、右值分析只需要一条SQL语句即可完成，以Fruit为例：

```sql
SELECT* FROM tree WHERE lft < 2 AND rgt > 11 ORDER BY lft ASC;
```

### 为某节点添加子孙节点

假定我们要在节点“Red”下添加一个新的子节点“Apple”，该树将变成如下图所示，其中红色节点为新增节点。

![](/images/treeMenu/005.gif)

```sql
create procedure insert_node
(
     node_id int,
		 node_name varchar(100) CHARACTER SET utf8
)
begin
  if (select count(*) from tree where id = node_id) > 0 then 
    begin
	    declare rgt_param int;
			declare tree_id_param int;
		  select rgt, tree_id into rgt_param, tree_id_param from tree where id = node_id;
		  update tree set rgt = rgt + 2 where rgt >= rgt_param;
		  update tree set lft = lft + 2 where lft >= rgt_param;
		  insert into tree(name, pid, lft, rgt, tree_id) values(node_name, node_id, rgt_param, rgt_param + 1, tree_id_param);
    end;
  else
	  begin
		declare max_tree_id int;
		select max(tree_id) into max_tree_id from tree where pid = 0;
	    insert into tree(name, pid, lft, rgt, tree_id) values(node_name, node_id, 1, 2, max_tree_id + 1);
	  end;
  end if;
end
```

### 删除某节点

如果我们想要删除某个节点，会同时删除该节点的所有子孙节点，而这些被删除的节点的个数为：(被删除节点的右值 – 被删除节点的左值+ 1) / 2，而剩下的节点左、右值在大于被删除节点左、右值的情况下会进行调整。来看看树会发生什么变化，以Beef为例，删除效果如下图所示。

![](/images/treeMenu/006.gif)

```sql
create procedure delete_node
(
     node_id int
)
begin
  if (select count(*) from tree where id = node_id) > 0 then 
    begin
	    declare lft_param int; 
			declare rgt_param int;
		  select lft,rgt into lft_param,rgt_param from tree where id = node_id;
			delete from tree where lft >= lft_param and rgt <= rgt_param;
			update tree set lft = lft-(rgt_param-lft_param+1) where lft>lft_param;
			update tree set rgt = rgt-(rgt_param-lft_param+1) where rgt>rgt_param;
   end;
	end if;
end;
```


# 参考

https://blog.csdn.net/ghostrabbit/article/details/80050777