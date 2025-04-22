---
title: Java中的CAS机制
slug: java-cas
date: 2021-02-08T19:11:20-08:00
categories:
    - Java
tags:
    - CAS
---

## 概述

CAS，Compare And Swap，直译为比较并替换，是JDK提供的非阻塞原子性操作。

位于java.util.concurrent.atomic包下，例如：AtomicInteger、AtomicBoolean、AtomicLong等等。

## 原理

给变量赋值之前，先从内存中取出最新的值，与之前旧值进行比较，如果一致则更新；如果不一致则重新计算结果，并重复刚才的比较操作，这个操作叫CAS的”自旋“操作。

## 源码

```java
public class AtomicInteger extends Number implements java.io.Serializable {
  	
  	private static final sun.misc.Unsafe U = sun.misc.Unsafe.getUnsafe();
  
		public final int incrementAndGet() {
        return U.getAndAddInt(this, VALUE, 1) + 1;
    }
  	
}
```

内部是通过JDK的sun.misc.Unsafe类来实现CAS操作，sun.misc.Unsafe是JDK内部工具类，从名字上就可以看出，该类对于Java层来讲是”不安全“的，只能通过反射的方式调用。

## ABA问题

CAS操作会有ABA问题，就是把一个变量的值从A更新到B再更新回A时，无法确定变量是否有被更新过的问题。

如何解决？

通过版本号控制的方式来区分，就是每次更新都会改变版本号，这样即使变量的值被更新成相同的值但因为版本号不同也可以区分出来。

