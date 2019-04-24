---
layout: post
title:  "二分查找算法的总结"
categories: Algorithms Binary-Search
tags: Algorithms Binary-Search
author: wenzhilee77
---

# 普通的二分查找

最普通的写法:
1. 范围在[L,R]闭区间中，L = 0、R = arr.length - 1；
2. 注意循环条件为 L <= R ，而不是L < R；

```
static int bs1(int[] arr,int key){
        int L = 0,R = arr.length - 1; //在[L,R]范围内寻找key
        int mid;
        while( L <= R){
            mid = L + (R - L) / 2;
            if(arr[mid] == key)
                return mid;
            if(arr[mid] > key)
                R = mid - 1;// key 在 [L,mid-1]内
            else
                L = mid + 1;
        }
        return -1;
    }
```

# 普通二分查找的另一种写法
首先说明，这个和上面的二分查找是完全一样的，只不过我们定义的区间不同而已:

上面的二分查找是在[L,R]的闭区间中查找，而这个二分查找是在[L,R)的左闭右开区间查找；

所以此时的循环条件是L < R ，因为R本来是一个不可到达的地方，我们定义为了开区间，所以R是一个不会考虑的数，所以我们循环条件是L < R；

同理，当arr[mid] > key的时候，不是R = mid - 1，因为我们定义的是开区间，所以R = mid ，因为不会考虑arr[mid]这个数；

```
//和上面的完全一样，只是一开始R不是arr.length-1 而是arr.length
    static int bs2(int[] arr,int key){
        int L = 0, R = arr.length; //注意这里R = arr.length 所以在[L,R)开区间中找
        int mid;
        while( L < R){ //注意这里 不是 L <= R
            mid = L + (R - L)/2;
            if(arr[mid] == key)
                return mid;
            if(arr[mid] > key)
                R = mid; // 在[L,mid)中找
            else
                L = mid + 1;
        }
        return -1;
    }
```

# 第一个=key的，不存在返回-1

这个和之前的不同是:

数组中可能有重复的key，我们要找的是第一个key的位置；

和普通二分查找法不同的是在我们要R = mid - 1前的判断条件不是arr[mid] > key，而是arr[mid] >= key；

为什么是上面那样，其实直观上理解，我们要找的是第一个，那我们去左边找的时候不仅仅arr[mid] > key就去左边找，等于我也要去找，因为我要最左边的等于的；

最后我们要判断L是否越界(L 有可能等于arr.length)，而且最后arr[L]是否等于要找的key；

如果arr[L]不等于key，说明没有这个元素，返回-1；

```
/**查找第一个与key相等的元素的下标，　如果不存在返回-1　*/
    static int firstEqual(int[] arr,int key){
        int L = 0, R = arr.length - 1; //在[L,R]查找第一个>=key的
        int mid;
        while( L <= R){
            mid = L + (R - L)/2;
            if(arr[mid] >= key)
                R = mid - 1;
            else
                L = mid + 1;
        }
        if(L < arr.length && arr[L] == key)
            return L;
        return -1;
    }
```

# 第一个>=key的

这个和上面那个寻找第一个等于key的唯一的区别就是:

最后我们不需要判断( L < arr.length && arr[L] == key)，因为如果不存在key的话，我们返回第一个> key的元素即可；

注意这里没有判断越界(L < arr.length )，因为如果整个数组都比key要小，就会返回arr.length的大小；

```
/**查找第一个大于等于key的元素的下标*/
    static int firstLargeEqual(int[] arr,int key){
        int L = 0, R = arr.length - 1;
        int mid;
        while( L <= R){
            mid = L + (R - L) / 2;
            if(arr[mid] >= key)
                R = mid - 1;
            else
                L = mid + 1;
        }
        return L;
    }
```

# 第一个>key的

这个和上两个的不同在于:

if(arr[mid] >= key) 改成了 if(arr[mid] > key)，因为我们不是要寻找 = key的；

看似和普通二分法很像，但是我们在循环中没有判断if(arr[mid] == key)就返回mid(因为要寻找的不是等于key的)，而是在最后返回了L ；

```
/**查找第一个大于key的元素的下标 */
    static int firstLarge(int[] arr,int key){
        int L = 0,R = arr.length - 1;
        int mid;
        while(L <= R){
            mid = L + (R - L) / 2;
            if(arr[mid] > key)
                R = mid - 1;
            else
                L = mid + 1;
        }
        return L;
    }
```

## 第一个...的总结

上面写了三个第一个.....的程序，可以发现一些共同点 ，也可以总结一下它们微妙的区别:

* 最后返回的都是L；

* 如果是寻找第一个等于key的，是if( arr[mid] >= key) R = mid - 1，且最后要判断L 的合法以及是否存在key；

* 如果是寻找第一个大于等于key的，也是if(arr[mid] >= key) R = mid - 1，但是最后直接返回L；

* 如果是寻找第一个大于key的，则判断条件是if(arr[mid] > key) R = mid - 1，最后返回L ；

# 最后一个=key的，不存在返回-1

和寻找第一个 = key的很类似，不过是方向的不同而已:

数组中有可能有重复的key，我们要查找的是最后一个 = key的位置，不存在返回-1；

为了更加的直观的理解，和寻找第一个...的形成对比，这里是当arr[mid] <= key的时候，我们要去右边查找(L = mid + 1)，同样是直观的理解，因为我们是要去找到最后一个 = key的，所以不仅仅是arr[mid] < key要去左边寻找，等于key的时候也要去左边寻找；

和第一个....不同的是，我们返回的都是R；

同时我们也要判断R的下标的合法性，以及最后的arr[R]是否等于key，如果不等于就返回-1；

```
/**查找最后一个与key相等的元素的下标，　如果没有返回-1*/
    static int lastEqual(int[] arr,int key){
        int L = 0, R = arr.length - 1;
        int mid;
        while( L <= R){
            mid = L + (R - L)/2;
            if(arr[mid] <= key)
                L = mid + 1;
            else
                R = mid - 1;
        }
        if(R >= 0 && arr[R] == key)
            return R;
        return -1;
    }
```

# 最后一个<=key的

这个和上面那个寻找最后一个等于key的唯一的区别就是:

最后我们不需要判断 (R >= 0 && arr[R] == key)，因为如果不存在key的话，我们返回最后一个 < key的元素即可；

注意这里没有判断越界(R >= 0 )，因为如果整个数组都比key要大，数组最左边的更左边一个(也就是-1)；

```
/**查找最后一个小于等于key的元素的下标 */
    static int lastSmallEqual(int[] arr,int key){
        int L = 0, R = arr.length - 1;
        int mid;
        while( L <= R){
            mid = L + (R - L) / 2;
            if(arr[mid] <= key)
                L = mid + 1;
            else
                R = mid - 1;
        }
        return R;
    }
```

# 最后一个<key 的

这个和上面两个不同的是:

和上面的程序唯一不同的就是arr[mid] <= key改成了 arr[mid] < key，因为我们要寻找的不是 = key的；

注意这三个最后一个的都是先对L的操作L = mid + 1，然后在else 中进行对R的操作；

```
/**查找最后一个小于key的元素的下标*/
    static int lastSmall(int[] arr,int key){
        int L = 0, R = arr.length - 1;
        int mid;
        while(L <= R){
            mid = L + (R - L) / 2;
            if(arr[mid] < key)
                L = mid + 1;
            else
                R = mid - 1;
        }
        return R;
    }
```

## 最后一个...的总结

上面三个都是求最后一个.....的，也进行一下总结:

最后返回的都是R；

* 第一个if判断条件(不管是arr[mid] <= key还是arr[mid] < key) ，都是L的操作，也就是去右边寻找；

* 如果是寻找最后一个 等于 key的， if(arr[mid] <= key) L = mid + 1; 不过最后要判断R的合法性以及是否存在key；

* 如果是寻找最后一个 小于等于 key的，也是if(arr[mid] <= key) L = mid + 1；不过最后直接返回R；

* 如果是寻找最后一个 小于 key的，则判断条件是 if(arr[mid] < key) L = mid + 1 ，最后返回R；

# 参考链接

https://github.com/ZXZxin/ZXBlog/blob/master/%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E7%AE%97%E6%B3%95/Algorithm/BinarySearch/%E4%BA%8C%E5%88%86%E6%9F%A5%E6%89%BE%E7%9A%84%E6%80%BB%E7%BB%93(6%E7%A7%8D%E5%8F%98%E5%BD%A2).md

