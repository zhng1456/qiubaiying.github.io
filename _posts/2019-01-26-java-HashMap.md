---
25layout:     post
title:      java-HashMap
subtitle:   HashMap
date:       2019-01-26
author:     rosewind
header-img: img/animation/2.jpg
catalog: true
tags:
    - java
---

# 概述

要理解HashMap， 就必须要知道了解其底层的实现， 而底层实现里最重要的就是它的数据结构了，HashMap实际上是一个“链表散列”的数据结构，即数组和链表的结合体。

# HashCode

在分析和理解HashMap源码前有必要对hashcode进行说明。

官方文档中的说明:

> hashcode方法返回该对象的哈希码值。支持该方法是为哈希表提供一些优点，例如，java.util.Hashtable 提供的哈希表。
> hashCode 的常规协定是：
> 在 Java 应用程序执行期间，在同一对象上多次调用 hashCode 方法时，必须一致地返回相同的整数，前提是对象上 equals 比较中所用的信息没有被修改。从某一应用程序的一次执行到同一应用程序的另一次执行，该整数无需保持一致。
>
> 如果根据 equals(Object) 方法，两个对象是相等的，那么在两个对象中的每个对象上调用 hashCode 方法都必须生成相同的整数结果。
>
> 以下情况不 是必需的：如果根据 equals(java.lang.Object) 方法，两个对象不相等，那么在两个对象中的任一对象上调用 hashCode 方法必定会生成不同的整数结果。但是，程序员应该知道，为不相等的对象生成不同整数结果可以提高哈希表的性能。
>
> 实际上，由 Object 类定义的 hashCode 方法确实会针对不同的对象返回不同的整数。（这一般是通过将该对象的内部地址转换成一个整数来实现的，但是 JavaTM 编程语言不需要这种实现技巧。）
>
> 当equals方法被重写时，通常有必要重写 hashCode 方法，以维护 hashCode 方法的常规协定，该协定声明相等对象必须具有相等的哈希码。

可以总结出下面几点:

1. 2个对象equals相等，那么hashCode也相同;反过来，hashCode不相同，那这2个对象用equals一定不相等
2. 2个对象hashCode相同，不能推出2个对象equals相同
3. 如果对象的equals方法被重写，那么对象的hashCode也要重写，并且产生hashCode使用的对象，一定要和equals方法中使用的一致

hashCode是用于查找使用的，而equals是用于比较两个对象的是否相等的。**先根据hashCode定位到某个桶里，但一个桶里可能有多个hashCode相同的对象，再通过equals方法，找到所要找的对象**

# 源码分析

为了便于理解，以下源码分析以 JDK 1.7 为主。

## 存储结构

内部包含了一个 Entry 类型的数组 table。

```java
// 默认初始容量为16，必须为2的n次幂
    static final int DEFAULT_INITIAL_CAPACITY = 16;

    // 最大容量为2的30次方
    static final int MAXIMUM_CAPACITY = 1 << 30;

    // 默认加载因子为0.75f
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    // Entry数组，长度必须为2的n次幂
    transient Entry[] table;

    // 已存储元素的数量
    transient int size ;

    // 下次扩容的临界值，size>=threshold就会扩容，threshold等于capacity*load factor
    int threshold;

    // 加载因子
    final float loadFactor ;
```

Entry 存储着键值对。它包含了四个字段，从 next 字段我们可以看出 Entry 是一个链表。即数组中的每个位置被当成一个桶，一个桶存放一个链表。HashMap 使用拉链法来解决冲突，同一个链表中存放哈希值相同的 Entry。

![HashMap](https://github.com/zhng1456/CS-Notes/raw/master/pics/8fe838e3-ef77-4f63-bf45-417b6bc5c6bb.png)



Entry的定义如下:

```java
static class Entry<K,V> implements Map.Entry<K,V> {
    final K key;
    V value;
    Entry<K,V> next;
    int hash;

    Entry(int h, K k, V v, Entry<K,V> n) {
        value = v;
        next = n;
        key = k;
        hash = h;
    }

    public final K getKey() {
        return key;
    }

    public final V getValue() {
        return value;
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry e = (Map.Entry)o;
        Object k1 = getKey();
        Object k2 = e.getKey();
        if (k1 == k2 || (k1 != null && k1.equals(k2))) {
            Object v1 = getValue();
            Object v2 = e.getValue();
            if (v1 == v2 || (v1 != null && v1.equals(v2)))
                return true;
        }
        return false;
    }

    public final int hashCode() {
        return Objects.hashCode(getKey()) ^ Objects.hashCode(getValue());
    }

    public final String toString() {
        return getKey() + "=" + getValue();
    }
}
```

## 拉链法的工作原理

```java
HashMap<String, String> map = new HashMap<>();
map.put("K1", "V1");
map.put("K2", "V2");
map.put("K3", "V3");
```

- 新建一个 HashMap，默认大小为 16；
- 插入 <K1,V1> 键值对，先计算 K1 的 hashCode 为 115，使用除留余数法得到所在的桶下标 115%16=3。
- 插入 <K2,V2> 键值对，先计算 K2 的 hashCode 为 118，使用除留余数法得到所在的桶下标 118%16=6。
- 插入 <K3,V3> 键值对，先计算 K3 的 hashCode 为 118，使用除留余数法得到所在的桶下标 118%16=6，插在 <K2,V2> 前面。

**应该注意到链表的插入是以头插法方式进行的，例如上面的 <K3,V3> 不是插在 <K2,V2> 后面，而是插入在链表头部。**

查找需要分成两步进行:

- 计算键值对所在的桶；
- 在链表上顺序查找，时间复杂度显然和链表的长度成正比。

![](https://github.com/zhng1456/CS-Notes/raw/master/pics/49d6de7b-0d0d-425c-9e49-a1559dc23b10.png)

## 构造函数

提供了4个构造函数:

1. HashMap()：构造一个具有默认初始容量 (16) 和默认加载因子 (0.75) 的空 HashMap。
2. HashMap(int initialCapacity)：构造一个带指定初始容量和默认加载因子 (0.75) 的空 HashMap。
3. HashMap(int initialCapacity, float loadFactor)：构造一个带指定初始容量和加载因子的空 HashMap。
4. public HashMap(Map<? extends K, ? extends V> m)：包含“子Map”的构造函数

```java
/**
     * 构造一个指定初始容量和加载因子的HashMap
     */
    public HashMap( int initialCapacity, float loadFactor) {
        // 初始容量和加载因子合法校验
        if (initialCapacity < 0)
            throw new IllegalArgumentException( "Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException( "Illegal load factor: " +
                                               loadFactor);

        // Find a power of 2 >= initialCapacity
        // 确保容量为2的n次幂，是capacity为大于initialCapacity的最小的2的n次幂
        int capacity = 1;
        while (capacity < initialCapacity)
            capacity <<= 1;

        // 赋值加载因子
        this.loadFactor = loadFactor;
        // 赋值扩容临界值
        threshold = (int)(capacity * loadFactor);
        // 初始化hash表
        table = new Entry[capacity];
        init();
    }

    /**
     * 构造一个指定初始容量的HashMap
     */
    public HashMap( int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    /**
     * 构造一个使用默认初始容量(16)和默认加载因子(0.75)的HashMap
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        threshold = (int)(DEFAULT_INITIAL_CAPACITY * DEFAULT_LOAD_FACTOR);
        table = new Entry[DEFAULT_INITIAL_CAPACITY];
        init();
    }

    /**
     * 构造一个指定map的HashMap，所创建HashMap使用默认加载因子(0.75)和足以容纳指定map的初始容量。
     */
    public HashMap(Map<? extends K, ? extends V> m) {
        // 确保最小初始容量为16，并保证可以容纳指定map
        this(Math.max(( int) (m.size() / DEFAULT_LOAD_FACTOR) + 1,
                      DEFAULT_INITIAL_CAPACITY ), DEFAULT_LOAD_FACTOR);
        putAllForCreate(m);
}
```

## put

```java
public V put(K key, V value) {
    if (table == EMPTY_TABLE) {
        inflateTable(threshold);
    }
    // 键为 null 单独处理
    if (key == null)
        return putForNullKey(value);
    int hash = hash(key);
    // 确定桶下标
    int i = indexFor(hash, table.length);
    // 先找出是否已经存在键为 key 的键值对，如果存在的话就更新这个键值对的值为 value
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }

    modCount++;
    // 插入新键值对
    addEntry(hash, key, value, i);
    return null;
}
```

HashMap 允许插入键为 null 的键值对。但是因为无法调用 null 的 hashCode() 方法，也就无法确定该键值对的桶下标，只能通过强制指定一个桶下标来存放。HashMap 使用第 0 个桶存放键为 null 的键值对。

```java
private V putForNullKey(V value) {
    for (Entry<K,V> e = table[0]; e != null; e = e.next) {
        if (e.key == null) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
    modCount++;
    addEntry(0, null, value, 0);
    return null;
}
```

使用链表的头插法，也就是新的键值对插在链表的头部，而不是链表的尾部。

```java
void addEntry(int hash, K key, V value, int bucketIndex) {
    if ((size >= threshold) && (null != table[bucketIndex])) {
        resize(2 * table.length);
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = indexFor(hash, table.length);
    }

    createEntry(hash, key, value, bucketIndex);
}

void createEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    // 头插法，链表头部指向新的键值对
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```

```java
Entry(int h, K k, V v, Entry<K,V> n) {
    value = v;
    next = n;
    key = k;
    hash = h;
}
```

## 确定桶下标

很多操作都需要先确定一个键值对所在的桶下标。

```java
int hash = hash(key.hashCode());//计算hash
int i = indexFor(hash, table.length);//计算在数组中的存储位置
//上面这两行实际上做的是下面这行代码
int i = key.hashCode()%table.length;
```

### 计算hash值

```java
final int hash(Object k) {
    int h = hashSeed;
    if (0 != h && k instanceof String) {
        return sun.misc.Hashing.stringHash32((String) k);
    }

    h ^= k.hashCode();

    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```

```java
public final int hashCode() {
    return Objects.hashCode(key) ^ Objects.hashCode(value);
}
```

### 取模

令 x = 1<<4，即 x 为 2 的 4 次方，它具有以下性质：

```
x   : 00010000
x-1 : 00001111
```

令一个数 y 与 x-1 做与运算，可以去除 y 位级表示的第 4 位以上数：

```
y       : 10110010
x-1     : 00001111
y&(x-1) : 00000010
```

这个性质和 y 对 x 取模效果是一样的：

```
y   : 10110010
x   : 00010000
y%x : 00000010
```

我们知道，位运算的代价比求模运算小的多，因此在进行这种计算时用位运算的话能带来更高的性能。

确定桶下标的最后一步是将 key 的 hash 值对桶个数取模：hash%capacity，如果能保证 capacity 为 2 的 n 次方，那么就可以将这个操作转换为位运算。**由上面构造函数的代码可以看出，可以保证capacity 为 2 的 n 次方**

```java
static int indexFor(int h, int length) {
    return h & (length-1);
}
```

## 扩容基本原理

设 HashMap 的 table 长度为 M，需要存储的键值对数量为 N，如果哈希函数满足均匀性的要求，那么每条链表的长度大约为 N/M，因此平均查找次数的复杂度为 O(N/M)。

为了让查找的成本降低，应该尽可能使得 N/M 尽可能小，因此需要保证 M 尽可能大，也就是说 table 要尽可能大。HashMap 采用动态扩容来根据当前的 N 值来调整 M 值，使得空间效率和时间效率都能得到保证。

和扩容相关的参数主要有：capacity、size、threshold 和 load_factor。

| 参数       | 含义                                                         |
| ---------- | ------------------------------------------------------------ |
| capacity   | table 的容量大小，默认为 16。需要注意的是 capacity 必须保证为 2 的 n 次方。 |
| size       | table 的实际使用量。                                         |
| threshold  | size 的临界值，size 必须小于 threshold，如果大于等于，就必须进行扩容操作。 |
| loadFactor | 装载因子，table 能够使用的比例，threshold = capacity * loadFactor。 |

```java
// 默认初始容量为16，必须为2的n次幂
    static final int DEFAULT_INITIAL_CAPACITY = 16;

    // 最大容量为2的30次方
    static final int MAXIMUM_CAPACITY = 1 << 30;

    // 默认加载因子为0.75f
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    // Entry数组，长度必须为2的n次幂
    transient Entry[] table;

    // 已存储元素的数量
    transient int size ;

    // 下次扩容的临界值，size>=threshold就会扩容，threshold等于capacity*load factor
    int threshold;

    // 加载因子
    final float loadFactor ;
```

从下面的添加元素代码中可以看出**，当需要扩容时，令 capacity 为原来的两倍**。

```java
void addEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    if (size++ >= threshold)
        resize(2 * table.length);
}
```

扩容使用 resize() 实现，需要注意的是，扩容操作同样需要把 oldTable 的所有键值对重新插入 newTable 中，因此这一步是很费时的。

```java
oid resize( int newCapacity) {
        // 当前数组
        Entry[] oldTable = table;
        // 当前数组容量
        int oldCapacity = oldTable.length ;
        // 如果当前数组已经是默认最大容量MAXIMUM_CAPACITY ，则将临界值改为Integer.MAX_VALUE 返回
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }

        // 使用新的容量创建一个新的链表数组
        Entry[] newTable = new Entry[newCapacity];
        // 将当前数组中的元素都移动到新数组中
        transfer(newTable);
        // 将当前数组指向新创建的数组
        table = newTable;
        // 重新计算临界值
        threshold = (int)(newCapacity * loadFactor);
    }

    /**
     * Transfers all entries from current table to newTable.
     */
    void transfer(Entry[] newTable) {
        // 当前数组
        Entry[] src = table;
        // 新数组长度
        int newCapacity = newTable.length ;
        // 遍历当前数组的元素，重新计算每个元素所在数组位置
        for (int j = 0; j < src. length; j++) {
            // 取出数组中的链表第一个节点
            Entry<K,V> e = src[j];
            if (e != null) {
                // 将旧链表位置置空
                src[j] = null;
                // 循环链表，挨个将每个节点插入到新的数组位置中
                do {
                    // 取出链表中的当前节点的下一个节点
                    Entry<K,V> next = e. next;
                    // 重新计算该链表在数组中的索引位置
                    int i = indexFor(e. hash, newCapacity);
                    // 将下一个节点指向newTable[i]
                    e. next = newTable[i];
                    // 将当前节点放置在newTable[i]位置
                    newTable[i] = e;
                    // 下一次循环
                    e = next;
                } while (e != null);
            }
        }
}
```

###  扩容-重新计算桶下标

**这里需要注意下是用每个元素的hash全部重新计算index，而不是简单的把原table对应index位置元素简单的移动到新table对应位置**

ashMap 使用了一个特殊的机制，可以降低重新计算桶下标的操作。

假设原数组长度 capacity 为 16，扩容之后 new capacity 为 32：

```
capacity     : 00010000
new capacity : 00100000
```

对于一个 Key，

- 它的哈希值如果在第 6 位上为 0，那么取模得到的结果和之前一样；
- 如果为 1，那么得到的结果为原来的结果 +16。

## 链表转红黑树

从 JDK 1.8 开始，一个桶存储的链表长度大于 8 时会将链表转换为红黑树。

## 与 HashTable 的比较

1. **两者最主要的区别在于Hashtable是线程安全，而HashMap则非线程安全**。Hashtable的实现方法里面都添加了synchronized关键字来确保线程同步，因此相对而言HashMap性能会高一些。

2. HashMap可以使用null作为key，而Hashtable则不允许null作为key。HashMap以null作为key时，总是存储在table数组的第一个节点上。

3. HashMap的初始容量为16，Hashtable初始容量为11，两者的填充因子默认都是0.75
   HashMap扩容时是当前容量翻倍即:capacity\*2，Hashtable扩容时是容量翻倍+1即:capacity\*2+1

4. HashMap和Hashtable的底层实现都是数组+链表结构实现

5. 两者计算hash的方法不同
   Hashtable计算hash是直接使用key的hashcode对table数组的长度直接进行取模

   ```java
   int hash = key.hashCode();
   int index = (hash & 0x7FFFFFFF) % tab.length;
   ```

   HashMap计算hash对key的hashcode进行了二次hash，以获得更好的散列值，然后对table数组长度取摸

   ```java
   static int hash(int h) {
        // This function ensures that hashCodes that differ only by
        // constant multiples at each bit position have a bounded
        // number of collisions (approximately 8 at default load factor).
        h ^= (h >>> 20) ^ (h >>> 12);
        return h ^ (h >>> 7) ^ (h >>> 4);
    }
   
   static int indexFor(int h, int length) {
        return h & (length-1);
    }
   ```

# 参考资料

[Java 集合干货系列 -（三）HashMap 源码解析](https://juejin.im/entry/58f3052aac502e006c2e8e18)

[java容器](https://github.com/zhng1456/CS-Notes/blob/master/notes/Java%20%E5%AE%B9%E5%99%A8.md)

[[给jdk写注释系列之jdk1.6容器(4)-HashMap源码解析](https://www.cnblogs.com/tstd/p/5055286.html)](http://www.cnblogs.com/tstd/p/5055286.html)