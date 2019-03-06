---
layout:     post
title:      转载-Java Integer用==进行比较
subtitle:   Integer
date:       2019-03-04
author:     转载
header-img: img/animation/6.jpg
catalog: true
tags:
    - Java
---

## 前言

这个几乎是Java 5引入自动装箱和自动拆箱后，很多人都会遇到（而且不止一次），而又完全摸不着头脑的坑。虽然已有很多文章分析了原因，但鉴于我这次还差点坑了同学，还是纪录下来长点记性。

## 问题描述

### 例一

来个简单点的例子

Java

```java
public static void main(String[] args) {
    for (int i = 0; i < 150; i++) {
        Integer a = i;
        Integer b = i;
        System.out.println(i + " " + (a == b));
    }
}
```

`i`取值从0到150，每次循环`a`与`b`的数值均相等，输出`a == b`。运行结果：

```plain
0 true
1 true
2 true
3 true
...
126 true
127 true
128 false
129 false
130 false
...
```

从128开始`a`和`b`就不再相等了。

这个例子还容易看出来涉及到int的自动装箱和自动拆箱，下面来个不太容易看出来的。



### 例二

Java

```java
public static void main(String[] args) {
    Map<Integer, Integer> mapA = new HashMap<>();
    Map<Integer, Integer> mapB = new HashMap<>();
    for (int i = 0; i < 150; i++) {
        mapA.put(i, i);
        mapB.put(i, i);
    }
    for (int i = 0; i < 150; i++) {
        System.out.println(i + " " + (mapA.get(i) == mapB.get(i)));
    }
}
```

`i`取值从0到150，`mapA`和`mapB`均存储`(i, i)`数值对，输出`mapA`的值与`mapB`的值的比较结果。运行结果：

```plain
0 true
1 true
2 true
3 true
...
126 true
127 true
128 false
129 false
130 false
...
```

为什么两个例子都是从0到127均显示两个变量相等，而从128开始不相等？



## 原因分析



### 自动装箱

首先回顾一下自动装箱。对于下面这行代码

Java

```java
Integer a = 1;
```

变量`a`为Integer类型，而`1`为int类型，且Integer和int之间并无继承关系，按照Java的一般处理方法，这行代码应该报错。

但因为自动装箱机制的存在，在为Integer类型的变量赋int类型值时，Java会自动将int类型转换为Integer类型，即

Java

```java
Integer a = Integer.valueOf(1);
```

`valueOf()`方法返回一个Integer类型值，并将其赋值给变量`a`。这就是int的自动装箱。



### 是同一个对象吗？

再看最开始的例子：

Java

```java
public static void main(String[] args) {
    for (int i = 0; i < 150; i++) {
        Integer a = i;
        Integer b = i;
        System.out.println(i + " " + (a == b));
    }
}
```

每次循环时，`Integer a = i`和`Integer b = i`都会触发自动装箱，而自动装箱会将int转换Integer类型值并返回；我们知道Java中两个new出来的对象因为时不同的实例，无论如何`==`都会返回fasle。比如

Java

```java
new Integer(1) == new Integer(1);
```

就会返回false。

那么例子中`Integer a = i`和`Integer b = i`自动装箱产生的变量`a`和`b`就不应该时同一个对象了，那么`==`的结果应该时false。128以上为false容易理解，但为何0到127时返回true了呢？`==`返回true的唯一情况是比较的两个对象为同一个对象，那不妨把例子中`a`和`b`的内存地址都打印出来看看：

Java

```java
for(int i=0;i<150;i++){
    Integer a=i;
    Integer b=i;
    System.out.println(a+" "+b+" "+System.identityHashCode(a)+" "+System.identityHashCode(b));
}
```

`identityHashCode()`方法可以理解为输出对应变量的内存地址，输出为：

```plain
0 0 762119098 762119098
1 1 1278349992 1278349992
2 2 1801910956 1801910956
3 3 1468253089 1468253089
...
126 126 1605164995 1605164995
127 127 1318497351 1318497351
128 128 101224864 479240824
129 129 1373088356 636728630
130 130 587071409 1369296745
...
```

竟然从0到127不同时候自动装箱得到的是同一个对象！从128开始才是正常情况。



### 看看源码

“从0到127不同时候自动装箱得到的是同一个对象”就只能有一种解释：自动装箱并不一定new出新的对象。

既然自动装箱涉及到的方法是`Integer.valueOf()`，不妨看看其源代码：

Java

```java
/**
    * Returns an {@code Integer} instance representing the specified
    * {@code int} value.  If a new {@code Integer} instance is not
    * required, this method should generally be used in preference to
    * the constructor {@link #Integer(int)}, as this method is likely
    * to yield significantly better space and time performance by
    * caching frequently requested values.
    *
    * This method will always cache values in the range -128 to 127,
    * inclusive, and may cache other values outside of this range.
    *
    * @param  i an {@code int} value.
    * @return an {@code Integer} instance representing {@code i}.
    * @since  1.5
    */
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

其注释里就直接说明了-128到127之间的值都是直接从缓存中取出的。看看是怎么实现的：如果int型参数`i`在`IntegerCache.low`和`IntegerCache.high`范围内，则直接由`IntegerCache`返回；否则new一个新的对象返回。似乎`IntegerCache.low`就是-128，`IntegerCache.high`就是127了。

看看IntegerCache的源码：

Java

```java
private static class IntegerCache {
    static final int low = -128;
    static final int high;
    static final Integer cache[];

    static {
        // high value may be configured by property
        int h = 127;
        String integerCacheHighPropValue =
            sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
        if (integerCacheHighPropValue != null) {
            try {
                int i = parseInt(integerCacheHighPropValue);
                i = Math.max(i, 127);
                // Maximum array size is Integer.MAX_VALUE
                h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
            } catch( NumberFormatException nfe) {
                // If the property cannot be parsed into an int, ignore it.
            }
        }
        high = h;

        cache = new Integer[(high - low) + 1];
        int j = low;
        for(int k = 0; k < cache.length; k++)
            cache[k] = new Integer(j++);

        // range [-128, 127] must be interned (JLS7 5.1.7)
        assert IntegerCache.high >= 127;
    }

    private IntegerCache() {}
}
```

果然在其static块中就一次性生成了-128到127直接的Integer类型变量存储在`cache[]`中，对于-128到127之间的int类型，返回的都是同一个Integer类型对象。

这下真相大白了，整个工作过程就是：**Integer.class在装载（Java虚拟机启动）时，其内部类型IntegerCache的static块即开始执行，实例化并暂存数值在-128到127之间的Integer类型对象。当自动装箱int型值在-128到127之间时，即直接返回IntegerCache中暂存的Integer类型对象。**

为什么Java这么设计？我想是出于效率考虑，因为自动装箱经常遇到，尤其是小数值的自动装箱；而如果每次自动装箱都触发new，在堆中分配内存，就显得太慢了；所以不如预先将那些常用的值提前生成好，自动装箱时直接拿出来返回。哪些值是常用的？就是-128到127了。



## 解决方法

既然我们的目的是比较数值是否相等，而非判断是否为同一对象；而自动装箱又不能保证同一数值的Integer一定是同一对象或一定不是同一对象，那么就不要用`==`，直接用`equals()`好了。实际上，Integer重写了`equals()`方法，直接比较对象的数值是否相等。

Java

```java
for (int i = 0; i < 150; i++) {
    Integer a = i;
    Integer b = i;
    System.out.println(i + " " + (a.equals(b)));
}
```

这样返回值就全都是true了。



## 备注

不仅int，Java中的另外7中基本类型都可以自动装箱和自动拆箱，其中也有用到缓存。见下表：

| 基本类型 | 装箱类型  | 取值范围           | 是否缓存 | 缓存范围        |
| -------- | --------- | ------------------ | -------- | --------------- |
| byte     | Byte      | -128 ~ 127         | 是       | -128 ~ 127      |
| short    | Short     | -2^15 ~ (2^15 - 1) | 是       | -128 ~ 127      |
| int      | Integer   | -2^31 ~ (2^31 - 1) | 是       | -128 ~ 127      |
| long     | Long      | -2^63 ~ (2^63 - 1) | 是       | -128～127       |
| float    | Float     | --                 | 否       | --              |
| double   | Double    | --                 | 否       | --              |
| boolean  | Boolean   | true, false        | 是       | true, false     |
| char     | Character | \u0000 ~ \uffff    | 是       | \u0000 ~ \u007f |

# 原链接

[Java: Integer用==比较时127相等128不相等的原因](https://www.polarxiong.com/archives/Java-Integer%E7%94%A8-%E6%AF%94%E8%BE%83%E6%97%B6127%E7%9B%B8%E7%AD%89128%E4%B8%8D%E7%9B%B8%E7%AD%89%E7%9A%84%E5%8E%9F%E5%9B%A0.html)

