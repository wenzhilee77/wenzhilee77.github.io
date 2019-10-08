---
layout: post
title:  字符串拼接，什么时候会走StringBuilder
categories: Java String StringBuilder
tags:  Java String StringBuilder
author: wenzhilee77
---

最近在突然想到了 String 字符串拼接问题，于是做了一个 demo 测试了一下，到底 String 类型的字符串在拼接的时候，哪种情况下会走会走 StringBulider 进行字符串拼接，而哪种情况编译器会对代码进行优化？话不多说，先看 demo

# 一、 问题
1. 案例一
```java
public class TestDemo
{
	public static void main(String[] args) 
	{
		String str = "111";
		String a = "111";
		String str2 = a + "";

		System.out.println(str == str2);
	}
}

false
```

2. 案例二
```java
public class TestDemo
{
	public static void main(String[] args) 
	{
		String str = "111";
		String a = "111";
		String str2 = “111” + "";

		System.out.println(str == str2);
	}
}

true
```

# 二、 探究问题
这时候，疑问就来了，为什么结果会不一致呢？*利用在 cmd 窗口输入 javap -c TestDemo.class 命令，对字节码文件进行反编译，发现了问题所在？*

```java
Compiled from "TestDemo.java"
public class TestDemo {
  public TestDemo();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: ldc           #2                  // String 111
       2: astore_1
       3: ldc           #2                  // String 111
       5: astore_2
       6: new           #3                  // class java/lang/StringBuilder
       9: dup
      10: invokespecial #4                  // Method java/lang/StringBuilder."<init>":()V
      13: aload_2
      14: invokevirtual #5                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      17: ldc           #6                  // String
      19: invokevirtual #5                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      22: invokevirtual #7                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      25: astore_3
      26: getstatic     #8                  // Field java/lang/System.out:Ljava/io/PrintStream;
      29: aload_1
      30: aload_3
      31: if_acmpne     38
      34: iconst_1
      35: goto          39
      38: iconst_0
      39: invokevirtual #9                  // Method java/io/PrintStream.println:(Z)V
      42: return
}
```
可以看到在案例 1 中，java 代码底层走了 StringBuilder，进行字符串拼接，然后调用了 StringBuilder 的 toString 方法。

```java
Compiled from "TestDemo.java"
public class TestDemo {
  public TestDemo();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: ldc           #2                  // String 111
       2: astore_1
       3: ldc           #2                  // String 111
       5: astore_2
       6: ldc           #2                  // String 111
       8: astore_3
       9: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
      12: aload_1
      13: aload_3
      14: if_acmpne     21
      17: iconst_1
      18: goto          22
      21: iconst_0
      22: invokevirtual #4                  // Method java/io/PrintStream.println:(Z)V
      25: return
}
```

而案例 2 中，对 class 文件进行反编译，发现代码出现了一点变化，并没有走 StringBuilder 进行字符串拼接。

# 三、总结
1. 案例 1 中，通过变量和字符串拼接，java 是需要先到内存找变量对应的值，才能进行完成字符串拼接的工作，这种方式 java 编译器没法优化，只能走 StringBuilder 进行拼接字符串，然后调用 toString 方法，当然返回的结果和常量池中的 111 这个字符串的内存地址是不一样的，因此结果为 false。

2. 案例 2 中，直接在表达式里写值，java 不用根据变量去内存里找对应的值，可以在编译的时候直接对这个表达式进行优化，优化后的表达式从 "111" + ""直接变成了"111" ，两个 String 类型的变量都指向了常量池的 111 字符串，因此结果为 true;

# 参考

https://mp.weixin.qq.com/s/bw1xq_9CtBkYb-TCBmRgHQ