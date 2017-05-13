---
title: java快速入门
date: 2017-05-12
---

[TOC]

## Java基础

***

### 类，抽象类，接口

|      | Class        | Abstract Class               | Interface         |
| ---- | ------------ | ---------------------------- | ----------------- |
| 意义   | 具有相同属性的某一类事物 | 抽象出一些公共特征，但是又不能直接实例化（包含抽象方法） | 定义一个标准（可以看成特殊抽象类） |
| 用法   | 单继承          | 必须存在子类继承，用于实现抽象方法            | 多重实现，多重继承         |

### 常见关键字	

| 关键字    | 含义              | 用法                                       |
| ------ | --------------- | ---------------------------------------- |
| static | 类方法，类属性（属于整个类）  | static方法只能调用static的数据和方法                 |
| final  | 这个数据/方法/类不能被改变了 | 定值 (constant value)，只能赋值一次，不能再被修改；方法不能被覆盖（private的方法默认为final的方法）；类不能被继承。 |

### Class访问权限

| 作用域       | 当前类  | 同一package | 子孙类  | 其他package |
| --------- | ---- | --------- | ---- | --------- |
| public    | √    | √         | √    | √         |
| protected | √    | √         | √    | ×         |
| friendly  | √    | √         | ×    | ×         |
| private   | √    | ×         | ×    | ×         |

**注：不写默认是friendly **

### 垃圾回收（GC）

| 栈                        | 堆                       |
| ------------------------ | ----------------------- |
| object（对象引用）             | new Object()（对象）        |
| 方法调用结束，栈上的数据被清空（引用和基本类型） | 当没有任何引用指向对象时，该对象被清空（GC） |

### 多态

| 概念       | 含义                       |
| -------- | ------------------------ |
| upcast   | 派生类引用转换为基类引用，依然可以调用派生类方法 |
| downcast | 必须先upcast，才能downcast     |

## Java进阶

------

### String类

- String类对象是不可变对象，即程序不能对其进行修改
- String类是唯一个创建对象时不需要new的类 

### 异常

![img](http://images.cnitblog.com/blog/413416/201304/09015838-44b97618e1c84db6be54d72c77ababb9.png)

橙色: unchecked; 蓝色: checked

- Error类通常指内部错误或者资源耗尽，无法在程序层面上解决
- Exception中的RuntimeException(及其衍生类)是Java程序自身造成的，也就是说，由于程序员在编程时犯错造成的，可以通过修正Java程序避免
- checked异常是指程序与外界环境交互时发生错误，程序无法控制，异常处理机制就是用于处理此类异常

### IO基础

- 流的读写来自于四个基类：InputStream，OutputStream，Reader，Writer

- InputStream，OutputStream及其衍生类用于处理字节流（byte）

- Reader，Writer及其衍生类用于处理unicode文本

  它们都位于java.io包中。继承关系如下：
  ![img](http://images.cnitblog.com/blog/413416/201304/11095417-95d48adbe5524f09992c842e1aec1065.gif)
  此外，IOException有如下衍生类：
  ![img](http://images.cnitblog.com/blog/413416/201304/11095513-e986539237a94d4e8ff1e45e77c5a1e1.gif)

###RTTI（运行时类型识别）

- Class类：类的类，每一个Class对象代表一个类，包含方法forName("ClassName")，getName()，getPackage()，getFileds()，getMethonds()

- 普通类：都含有对应的Class类对象，可以通过Object.getClass()或者Object.class的方式获得该类的Class对象

- Class对象的加载：当创建某个类的对象时，会先检查内存中是否存在相应的Class对象，如果没有会在.class文件中寻找并加载，加载之后，对该类对象的创建和其他操作都参考该Class对象

###多线程

- 继承Thread类：Thread(string name), Thread(Runnable r, string name), getName(), run(), start(), join(Thread t), setDeamon()
- 实现Runnable接口：run(), 容易实现多重继承
- synchronize方法修饰符：用于共享资源的互斥访问，同一对象的synchronize方法只能同时被一个线程访问
- 关键代码：synchronize(syncObj){ ... }，用于同步部分代码

### 容器

- 数组：int[] array = new int[3], System.arraycopy()

- Collection：List和Set

- Map：键值对，哈希表，keySet(), values()

  各个类与接口的关系:

  ​

  ![img](http://images.cnitblog.com/blog/413416/201304/15203418-c01e434f91fb482b902463bcacbf3f69.png)


### 嵌套类

- 内部类必须依附于某个外部类对象，内部类对象可以访问它所依附的外部类对象的成员(即使是private）

- 内部类对象附带有创建时的环境信息，也就是闭包

- 内部类可以是static类，称为嵌套static类，跟static方法static成员类似，嵌套static类不依附于外部类的某个对象，依附于整个外部类
###内存管理与垃圾回收

- 栈：每个线程都有一个栈，栈中保存了对象的引用和基本类型的变量。每次方法调用都会在栈中增加一帧	，帧中保存了参数，局部变量和返回地址，方法调用结束，对应的帧将被删除。程序结束，栈也被清空。


- 堆：保存了引用所指向的对象，垃圾回收是针对堆的，早期使用引用计数的方式，但是无法处理相互引用的问题
- JVM的垃圾回收：
  1. "mark and sweep"（标记清除）
  2. "copy and sweep"（复制清除）
  3. "generational collection"（分代收集）
