---
layout: post
title:  一个Java对象到底占用多大内存
categories: Object Size
tags:  Object Size
author: wenzhilee77
---

# 简介

在进行 JVM 调优时，我们经常关注 JVM 各个区域大小以及相关参数，从而进行特定的优化，在一次排查内存溢出问题时我不禁想到一个问题，一个 Java 对象到底占用多大内存？下面我们就来分析验证下。

# Java 对象内存结构

在 JVM 中，Java 对象都是在堆内存上分配的，想要分析出 Java 对象内存占用，首先要了解 Java 对象内存结构，一个 Java 对象内存占用由三部分组成：对象头（Header）,实例数据（Instance Data）和对齐填充（Padding）。

## 对象头（Header）
### 对象头的组成
虚拟机的对象头包括两部分信息，第一部分用于存储对象自身的运行时数据，如 hashCode 、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等。这部分数据的长度在 32 位和 64 位的虚拟机（未开启指针压缩）中分别为 4B 和 8B ，官方称之为 ”Mark Word”。
对象的另一部分是类型指针（kclass），即对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是那个类的实例。另外如果对象是一个 Java 数组，那在对象头中还必须有一块用于记录数组长度的数据，因为虚拟机可以通过普通 Java 对象的元数据信息确定 Java 对象的大小，但是从数组的元数据中却无法确定数组的大小。同样，这部分数据的长度在 32 位和 64 的虚拟机（未开启指针压缩）中分别为 4B 和 8B。
### 指针压缩
从 JDK 1.6 update14 开始，64 bit JVM 正式支持了 -XX:+UseCompressedOops 这个可以压缩指针，起到节约内存占用的新参数。

如果 UseCompressedOops 是打开的，则以下对象的指针会被压缩：
```java
所有对象的 klass 属性
所有对象指针实例的属性
所有对象指针数组的元素（objArray）
```

由此我们可以计算出对象头大小：
```java
32位虚拟机对象头大小= Mark Word（4B）+ kclass(4B) = 8B   
64位虚拟机对象头大小= Mark Word（8B）+ kclass(4B) = 12B
```

## 实例数据
一个 Java 对象中的实例数据可能包括两种，一是 8 种基本类型，二是实例数据也是一个对象，说到这里很多人可能有个误区：
```java
基本类型？基本类型不是在栈上分配内存的吗？怎么要计算到分配在堆内存上对象的大小里面去？
```

基本类型在栈上分配内存？其实并不是，所谓“栈内存保存基本类型以及对象的引用(reference)，堆内存保存对象” 只是一句不严谨的话，实际仔细研究起来，栈内存（更专业的术语叫做堆栈）作为虚拟机作为方法调用和方法执行的数据结构，可能保存五种信息：
```java
局部变量表
操作数栈
动态链接
方法返回地址
附加信息
```

其中局部变量表中存储了方法中的局部变量，可能为 8 种基本类型或者 reference

也就是说，栈内存中保存的基本类型，都是方法中的局部变量，而如果基本类型作为对象的实例变量，是在堆上分配空间的，此外，如果实例变量被final修饰，则既不在栈也不在堆上分配空间，而是分配到常量池里面。

## 对齐填充（Padding）
由于虚拟机内存管理体系要求 Java 对象内存起始地址必须为 8 的整数倍，换句话说，Java 对象大小必须为 8 的整数倍，当对象头+实例数据大小不为 8 的整数倍时，将会使用Padding机制进行填充，譬如， 64 位虚拟机上 new Object() 实际大小为:

Mark Word（8B）+ kclass(4B)[开启指针压缩] = 12B

但由于Padding机制，实际占用空间为： Mark Word（8B）+ kclass(4B)[开启指针压缩]+Padding(4B) = 16B

## 数组的大小
Java 中数组也是一种对象，数组的大小与普通 Java 对象相比多了数组长度的信息（4B），即一个数组对象大小为 Mark Word（8B）+ kclass(4B)[开启指针压缩] + 数组长度(4B) = 16B

# 测试
```java
package sizeof;

public class ObjectSizeTest {

    private String s; // 4
    private int i1; // 4
    private byte b1; // 1
    private byte b2; // 1
    private int i2;// 4
    private Object obj; //4
    private byte b3;  // 1


    public static void main(String[] args) {
        System.out.println(ObjectShallowSize.sizeOf(new ObjectSizeTest()));
    }
}
```

理论值：Mark Word（8B）+ kclass(4B) + s(4B) + i1(4B) + b1(1B) + b2(1B) + 2(padding) + i2(4B) + obj(4B)+ b3(1B) + Paddding(7B) = 40B

实际值:32B

纳尼？这里为什么理论值和实际值不一致？

事实上，HotSpot创建的对象的字段会先按照给定顺序排列一下,默认的顺序如下，从长到短排列，引用排最后:  long/double --> int/float -->  short/char --> byte/boolean --> Reference
这个顺序可以使用JVM参数:  -XX:FieldsAllocationSylte = 0 (默认是1)来改变。
按照这种方法我们来重新计算下对象大小
Mark Word（8B）+ kclass(4B) + i1(4B) + i2(4B) + b1(1B) + b2(1B) + b3(1B) + Paddding(1B) + s(4B) + obj(4B) = 32B
与预期值一致。



# 参考

https://juejin.im/post/5df829b5518825122322cca3?utm_source=gold_browser_extension