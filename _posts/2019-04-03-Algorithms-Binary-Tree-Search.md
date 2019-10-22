---
layout: post
title:  数据结构之二叉树——二叉查找树
categories: Algorithms Binary-Search-Tree
tags:  Algorithms Binary-Search-Tree
author: wenzhilee77
---

# 定义
二叉查找树（Binary Search Tree），又称为二叉搜索树，二叉排序树。它可以是一棵空树，如果不是空树，则具有下列的性质：
* 非空左子树的所有键值小于其根结点的键值
* 非空右子树的所有键值大于其根结点的键值
* 左、右子树都是二叉查找树

比如下面两棵树，左边的树，因为5小于10，应该在10的左子树上，因此不是二叉查找树，右边的树则符合二叉查找树的条件。

![](/images/bst/image1.jpg)

由于二叉查找树的特性，中序遍历二叉查找树，得到的就是一个升序的数列。以上图右边的树为例，它的中序遍历则是：15 30 33 41 50

# 查找
二叉查找树的查找过程，可以分为下面的步骤：
* 从根结点开始，如果树为空，则返回NULL
* 如果树不为空，则根结点的值与查找值X进行比较，并进行不同的处理
  ** 如果X等于根结点的值，则查找完成，返回结点
  ** 如果X小于根结点的值，则在左子树中继续查找
  ** 如果X大于根结点的值，则在右子树中继续查找

下图以查找33为例，展示了查找的轨迹

![](/images/bst/image2.jpg)

根据上面的步骤，可以使用递归写出该算法
```java
def find(root: TreeNode, key):
    if not root:
        return None
    if key == root.val:
        return root
    elif key < root.val:
        return find(root.left, key)
    else:
        return find(root.right, key)

```
当然，上面的递归算法中存在尾递归，一般的编译器都会自动优化尾递归，我们也可以将代码改为迭代的方式
```java
def find(root: TreeNode, key):
    if not root:
        return None
    while root:
        if key == root.val:
            return root
        elif key < root.val:
            root = root.left
        else:
            root = root.right

```
可以看的出来，算法的复杂度和树的高度有关，每一次对比之后都会舍弃掉原来数据的一半，因此算法的时间复杂为O(logn)

# 最小元素与最小元素
根据二叉查找树的性质:
* 最小元素一定是在树的最左分枝的端结点上
* 最大元素一定是在树的最右分枝的端结点上

![](/images/bst/image3.jpg)

算法的实现也比较清晰
```java
def findMax(root: TreeNode):
    if root:
        while root.right:
            root = root.right
    return root


def findMin(root: TreeNode):
    if root:
        while root.left:
            root = root.left
    return root

```

# 插入结点
由于二叉查找树具有特定的性质，因此在插入新的结点的时候，也要保证二叉查找树的性质，整个插入新结点的过程与查找的过程类似，

以插入35为例，下图展示了插入35的轨迹

![](/images/bst/image4.jpg)

以插入32为例，下图展示例插入32的轨迹

![](/images/bst/image5.jpg)

```java
def insert(root: TreeNode, key):
    if not root:
        return TreeNode(key) # 如果是空树，则新建根结点
    while root:
        if key == root.val:
            break # 如果插入的key已经存在，则直接退出循环
        elif key < root.val:
            # 小于向左子树查找
            if not root.left:
                root.left = TreeNode(key)
            else:
                root = root.left
        else:
            # 大于向右子树查找
            if not root.right:
                root.right = TreeNode(key)
            else:
                root = root.right
    return root

```

# 删除结点
删除操作比较复杂，要分情况谈论：
1. 要删除的结点为叶子结点
这种情况下，直接删除。如下图要删除35结点

![](/images/bst/image6.jpg)

2. 要删除的结点只有一个孩子结点
这种情况下，直接将要删除结点的孩子结点，向上提，替代要删除的结点，如下图要删除33结点

![](/images/bst/image7.jpg)

3. 要删除的结点有左、右两棵子树
这种情况下，可以有两种方案：
* 取左子树的最大元素替代要删除结点
* 取右子树的最小元素替代要删除结点

如果要删除41结点（使用左子树最大元素代替）
左子树的最大元素，依旧比右子树的所有元素小，但是比其他左子树的元素大，因此它成为新的根结点的时候，依旧能保持左子树下的所有元素比根结点小，右子树下的所有元素比根结点大。

![](/images/bst/image8.jpg)

如果要删除41结点（使用右子树最小元素代替）

右子树的最小元素，依旧比左子树的所有元素大，但是比其他右子树的元素小，成为新的根结点，依旧能保持二叉查找树的性质

![](/images/bst/image9.jpg)

综合上面的情况，代码如下
```java
def delete(root: TreeNode, key):
    if not root:
        return None
    if key < root.val:
        root.left = delete(root.left, key)
    elif key > root.val:
        root.right = delete(root.right, key)
    else:
        if root.left and root.right:
            # 有左右孩子
            leftMax = findMax(root.left)
            root.val = leftMax.val
            root.left = delete(root.left, leftMax.val)
        elif not root.left and not root.right:
            # 叶子结点
            root = None
        elif root.left:
            # 只有左孩子
            root = root.left
        elif root.right:
            # 只有右孩子
            root = root.right
    return root

```


# 参考

https://juejin.im/post/5da13cc551882532d4253c39?utm_source=gold_browser_extension