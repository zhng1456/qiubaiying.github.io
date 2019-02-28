---
layout:     post
title:      Spring-AOP详解
subtitle:   leetcode
date:       2019-02-27
author:     rosewind
header-img: img/animation/10.jpg
catalog: true
tags:
    - Spring
---

# 前言

开始整理Spring的相关知识，这一篇整理Spring的AOP模块。涉及到一些代理的知识，可以参见[Java代理详解](http://langos.top/2019/01/24/java%E4%BB%A3%E7%90%86%E8%AF%A6%E8%A7%A3/)

# 为什么叫面向切面

我们学Java面向对象的时候，如果代码重复了怎么办啊？？可以分成下面几个步骤：

- 1：抽取成方法
- 2：抽取类

抽取成类的方式我们称之为：**纵向抽取**

- 通过继承的方式实现纵向抽取

但是，我们现在的办法不行：即使抽取成类还是会出现重复的代码，因为这些逻辑(开始、结束、提交事务)**依附在我们业务类的方法逻辑中**！



![img](https://user-gold-cdn.xitu.io/2018/5/24/1639259ea399586b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



现在纵向抽取的方式不行了，AOP的理念：就是将**分散在各个业务逻辑代码中相同的代码通过横向切割的方式**抽取到一个独立的模块中！



![img](https://user-gold-cdn.xitu.io/2018/5/24/1639259ea3e1fbcf?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



上面的图也很清晰了，将重复性的逻辑代码横切出来其实很容易(我们简单可认为就是封装成一个类就好了)，但我们要将这些**被我们横切出来的逻辑代码融合到业务逻辑中**，来完成和之前(没抽取前)一样的功能！这就是AOP首要解决的问题了！

# AOP的原理

没有学Spring AOP之前，我们就可以使用代理来完成。

- 代理能干嘛？代理可以帮我们**增强对象的行为**！使用动态代理实质上就是**调用时拦截对象方法，对方法进行改造、增强**！

其实Spring AOP的底层原理就是**动态代理**！

> Spring AOP构建在动态代理基础之上，因此，**Spring对AOP的支持局限于方法拦截**。

# 相关术语

- `Aspect`（切面）： Aspect 声明类似于 Java 中的类声明，在 Aspect 中会包含着一些 Pointcut 以及相应的 Advice。

- `Joint point`（连接点）：表示在程序中明确定义的点，典型的包括方法调用，对类成员的访问以及异常处理程序块的执行等等，它自身还可以嵌套其它 joint point。

- `Pointcut`（切点）：表示一组 joint point，这些 joint point 或是通过逻辑关系组合起来，或是通过通配、正则表达式等方式集中起来，它定义了相应的 Advice 将要发生的地方。

- `Advice`（增强）：Advice 定义了在 `Pointcut` 里面定义的程序点具体要做的操作，它通过 before、after 和 around 来区别是在每个 joint point 之前、之后还是代替执行的代码。

- `Target`（目标对象）：织入 `Advice` 的目标对象.。

- `Weaving`（织入）：将 `Aspect` 和其他对象连接起来, 并创建 `Advice`d object 的过程

# 参考资料

[Spring AOP就是这么简单啦](https://juejin.im/post/5b06bf2df265da0de2574ee1)

[Spring【AOP模块】就这么简单](https://mp.weixin.qq.com/s?__biz=MzI4Njg5MDA5NA==&mid=2247483954&idx=1&sn=b34e385ed716edf6f58998ec329f9867&chksm=ebd74333dca0ca257a77c02ab458300ef982adff3cf37eb6d8d2f985f11df5cc07ef17f659d4#rd)

[知乎:细说Spring——AOP详解（AOP概览）](https://zhuanlan.zhihu.com/p/37497663)