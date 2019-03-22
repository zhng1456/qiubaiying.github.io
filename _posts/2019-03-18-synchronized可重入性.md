---
layout:     post
title:      synchronized可重入性
subtitle:   synchronized
date:       2019-03-18
author:     rosewind
header-img: img/animation/19.jpg
catalog: true
tags:
    - Java
---

# 前言

最近准备系统地学习一下Java中的几种锁，先用这篇文章开个头，研究一下synchronized的可重入性。

# 1. 线程安全与可重入性

在回答引言的问题前，我们先讲解一下可重入性。在线程这块知识中，可重入性常常和线程安全进行对比。

## 1.1. 线程安全

线程安全函数的概念比较直观，众所周知，同一进程的不同线程会共享同一主内存，线程的私有栈中只包括PC,栈，操作数栈，局部变量数组和动态链接。对共享内存进行读写时，若要保证线程安全，则必须通过加锁的方式。

## 1.2. 可重入

### 1.2.1. 定义

关于可重入这一概念，我们需要参考维基百科。

> 若一个程序或子程序可以“在任意时刻被中断然后操作系统调度执行另外一段代码，这段代码又调用了该子程序不会出错”，则称其为可重入（reentrant或re-entrant）的。即当该子程序正在运行时，执行线程可以再次进入并执行它，仍然获得符合设计时预期的结果。与多线程并发执行的线程安全不同，可重入强调对单个线程执行时重新进入同一个子程序仍然是安全的。

### 1.2.2. 可重入的条件

- 不在函数内使用静态或全局数据。
- 不返回静态或全局数据，所有数据都由函数的调用者提供。
- 使用本地数据（工作内存），或者通过制作全局数据的本地拷贝来保护全局数据。
- 不调用不可重入函数。

## 1.3. 可重入与线程安全

一般而言，可重入的函数一定是线程安全的，反之则不一定成立。在不加锁的前提下，如果一个函数用到了全局或静态变量，那么它不是线程安全的，也不是可重入的。如果我们加以改进，对全局变量的访问加锁，此时它是线程安全的但不是可重入的，因为通常的枷锁方式是针对不同线程的访问（如Java的synchronized），当同一个线程多次访问就会出现问题。只有当函数满足可重入的四条条件时，才是可重入的。

# 2. synchronized的可重入性

## 2.1. synchronized是可重入锁

回到引言里的问题，如果一个获取锁的线程调用其它的synchronized修饰的方法，会发生什么？

从设计上讲，当一个线程请求一个由其他线程持有的对象锁时，该线程会阻塞。当线程请求自己持有的对象锁时，如果该线程是重入锁，请求就会成功，否则阻塞。

我们回来看synchronized，synchronized拥有强制原子性的内部锁机制，是一个可重入锁。因此，在一个线程使用synchronized方法时调用该对象另一个synchronized方法，即一个线程得到一个对象锁后再次请求该对象锁，是**永远可以拿到锁的**。

在Java内部，同一个线程调用自己类中其他synchronized方法/块时不会阻碍该线程的执行，同一个线程对同一个对象锁是可重入的，同一个线程可以获取同一把锁多次，也就是可以多次重入。原因是Java中线程获得对象锁的操作是以线程为单位的，而不是以调用为单位的。

## 2.2. synchronized可重入锁的实现

之前谈到过，每个锁关联一个线程持有者和一个计数器。当计数器为0时表示该锁没有被任何线程持有，那么任何线程都都可能获得该锁而调用相应方法。当一个线程请求成功后，JVM会记下持有锁的线程，并将计数器计为1。此时其他线程请求该锁，则必须等待。而该持有锁的线程如果再次请求这个锁，就可以再次拿到这个锁，同时计数器会递增。当线程退出一个synchronized方法/块时，计数器会递减，如果计数器为0则释放该锁。

## 2.3代码验证

```java
public class AccountingSync implements Runnable{
    static AccountingSync instance=new AccountingSync();
    static int i=0;
    static int j=0;
    @Override
    public void run() {
        for(int j=0;j<1000000;j++){
 
            //this,当前实例对象锁
            synchronized(this){
                i++;
                increase();//synchronized的可重入性
            }
        }
    }
 
    public synchronized void increase(){
        j++;
    }
 
 
    public static void main(String[] args) throws InterruptedException {
        Thread t1=new Thread(instance);
        Thread t2=new Thread(instance);
        t1.start();t2.start();
        t1.join();t2.join();
        System.out.println(i);
    }
}
```

为了更好地理解synchronized，还需要了解monitor。下一篇文章就研究下Java中的monitor。

# 参考资料

[synchronized的可重入性](https://blog.csdn.net/u010002184/article/details/72938691)

[Java多线程：synchronized的可重入性](https://www.jianshu.com/p/5379356c648f)

[死磕Synchronized底层实现--概论](https://juejin.im/post/5bfe6ddee51d45491b0163eb)