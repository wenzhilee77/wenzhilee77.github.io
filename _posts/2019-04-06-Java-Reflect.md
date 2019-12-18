---
layout: post
title:  java 反射机制
categories: Java Reflect
tags:  Java Reflect
author: wenzhilee77
---

# 反射是什么
JAVA反射机制是在运行状态中，对于任意一个类，都能够获取这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意一个方法和属性；这种动态获取类信息以及动态调用对象内容就称为java语言的反射机制。【翻译于 官方文档】

# 反射的作用
* 在运行时判断任意一个对象所属的类；
* 在运行时构造任意一个类的对象；
* 在运行时判断任意一个类所具有的成员变量和方法；
* 在运行时调用任意一个对象的方法；

# new和反射创建的区别
从结果来说没有区别。
* new：静态编译，只能用于编译期就能确定的类型，而且需要import相应的包。
* 反射：动态编译，在运行时可以确定类型并创建其对象，能够实现一些更为动态的效果。

# 反射的实现
我们知道，要使用一个类，就要先把它加载到虚拟机中，生成一个Class对象。这个class对象就保存了这个类的一切信息。 反射机制的实现，就是获取这个Class对象，通过Class对象去访问类、对象的元数据以及运行时的数据。有三种方法获得类的Class对象：
* Class.forName(String className)
* className.class
* 实例对象.getClass()

# 什么是class类
在面向对象的世界里，万物皆对象。类是对象，类是java.lang.Class类的实例对象。另外class类只有java虚拟机才能new出来,任何一个类都是Class 类的实例对象,这实例对象有三种表达方式：

```java
public class Person {
}
```

```java
public class PersonClassTest {
    public static void main(String[] args) {
        try {
            Person p = new Person();
            //方式1
            Class p1 = Person.class;
            //方式2
            Class p2 = p.getClass();
            //方式3
            Class p3 = Class.forName("com.java.xiaoxin.Person");
            //可以通过类的类型创建该类的实例对象
            Person person = (Person) p1.newInstance();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

# 反射涉及的API
反射首先获取Class对象；然后获取Method类和Field类；最后通过Method和Field类进行具体的方法调用或属性访问。

1. 在运行时获取对象所属类的类名等信息

```java
对象名.getClass().getName();
```

```java
public class PersonClassTest {
    public static void main(String[] args) {
        String name = Person.class.getName();
        System.out.println(name);//com.java.xiaoxin.Person
    }
}
```

2. 在运行时，通过创建class对象，获取自己的父类信息

```java
 Class<?> clazz = Class.forName(当前类);
 Class<?> parentClass = clazz.getSuperclass();
 parentClass.getName()；//获得父类名
```

```java
public class PersonClassTest {
    public static void main(String[] args) {
        try {
            Class<?> clazz = Class.forName("com.java.xiaoxin.Person");
            Class<?> parentClass = clazz.getSuperclass();
            parentClass.getName();
            System.out.println(parentClass.getName());//java.lang.Object
        }catch (Exception e){
			e.printStackTrace();
        }
    }
}
```

3. 通过反射机制创建一个类的对象

```java
1：反射创建class对象
2：Classname 对象=classname.newInstance(参数);
```

```java
public class PersonClassTest {
    public static void main(String[] args) {
        try {
           Class clazz = Class.forName("com.java.xiaoxin.Person");
           Person person = (Person) clazz.newInstance();
        }catch (Exception e){
			e.printStackTrace();
        }
    }
}
```

4. 获取类的全部方法，存于一个数组中

```java
//创建class对象
Class<?> clazz = Class.forName(ClassName);
// 返回声明的所有方法，包括公共、保护、默认（包）访问和私有方法，但不包括继承的方法。 
Method[] method1 = clazz.getDeclaredMethods()；
//返回可被访问的公共方法      
Method[] method2 = clazz.getMethods();
```

```java
public class PersonClassTest {

    public static void main(String[] args) {
        try {
            Class clazz = Class.forName("com.java.xiaoxin.Person");
            /**
             * 一个成员方法就是一个method对象
             * getMethod()所有的 public方法，包括父类继承的 public
             * getDeclaredMethods()获取该类所有的方法，包括private ,但不包括继承的方法。
             */
            Method[] methods = clazz.getMethods();//获取方法
            //获取所以的方法，包括private ,c.getDeclaredMethods();
            for (int i = 0; i < methods.length; i++) {
                //得到方法的返回类型
                Class returnType = methods[i].getReturnType();
                System.out.print(returnType.getName());
                //得到方法名：
                System.out.print(methods[i].getName() + "(");

                Class[] parameterTypes = methods[i].getParameterTypes();
                for (Class class1 : parameterTypes) {
                    System.out.print(class1.getName() + ",");
                }
                System.out.println(")");
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

输出

```java
voidwait()
voidwait(long,int,)
voidwait(long,)
booleanequals(java.lang.Object,)
java.lang.StringtoString()
inthashCode()
java.lang.ClassgetClass()
voidnotify()
voidnotifyAll()
```

5. 获取类的全部字段，存于一个数组中

```java
Class<?> clazz = Class.forName(classname);
// 取得本类已声明的所有字段，包括私有的、保护的
Field[] field = clazz.getDeclaredFields();
// 取得本类中可访问的所有公共字段
Field[] filed1 = clazz.getFields();
```

```java
public class Person {
   private String name;
   public int age;
}
```

```java
public class PersonClassTest {

    public static void main(String[] args) {
        try {
            Class clazz = Class.forName("com.java.xiaoxin.Person");
            Field[] field = clazz.getDeclaredFields();
            Field[] filed1 = clazz.getFields();
            for(Field f:field){
                //获取成员变量的类型
                Class filedType=f.getType();
                System.out.println(filedType.getName()+"-->"+f.getName());
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

输出

```java
java.lang.String-->name
int-->age
```

6. 方法反射的操作
获取一个方法：需要获取方法的名称和方法的参数才能决定一个方法。
方法的反射操作：

```java
method.invoke(对象，参数列表);
```

```java
public class Person {
    private String name;
    public int age;

    public void say(String name,int age){
        System.out.print(name+"今年"+age+"岁");
    }

}
```

```java
public class PersonClassTest {

    public static void main(String[] args) {
        try {
            Person person=new Person();
            Class clazz=person.getClass();
            Method method=clazz.getMethod("say",new Class[]{String.class,int.class});
            //也可以 Method method=c.getMethod("add",String.class,int.class);
            //方法的反射操作
            method.invoke(person,"小新",18);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

输出：

```java
小新今年18岁
```


# 参考

https://juejin.im/post/5df5ae586fb9a0161a0c3073?utm_source=gold_browser_extension