---
title: 深入分析ConcurrentHashMap源码
date: 2020-06-16 22:04:15
tags: JUC
---

# 深入分析ConcurrentHashMap源码

## Hash表
通过hash函数来计算数据位置的数据结构（数组）


## hash冲突
多个不同的key通过hash函数运算之后落到同一个数组下标
解决hash冲突的方法：
- 线性探索（开放地址法），i表示位置，通过i+1，i+2往后寻找，直到找到没有值的位置（ThreadLocal）。
- 链地址法，通过单向链表将冲突的值挂在对应位置下面。
- 再hash（通过多个hash函数） --- 布隆过滤器（bitmap）
- 建立公共溢出区，通过hash表和溢出表来存储数据，将和基本表有冲突的数据存储在溢出表里

## ConcurrentHashMap源码设计

#### put方法（分五个阶段）

- 第一个阶段，初始化阶段
    - 第一次循环，初始化hash表
    - 第二次循环，存值
```java
if (key == null || value == null) throw new NullPointerException();
int hash = spread(key.hashCode());
int binCount = 0;
for (Node<K,V>[] tab = table;;) {   // table就是存储数据的 node数组
    Node<K,V> f; int n, i, fh;      // i 就是数组下标
    if (tab == null || (n = tab.length) == 0)
        tab = initTable();
    else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {  //tabAt 从内存中拿值，相当于tab[i], 
        if (casTabAt(tab, i, null,
                        new Node<K,V>(hash, key, value, null)))
            break;                   // no lock when adding to empty bin
    }
```
初始化16个node的数组
```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) { //CAS原子操作，线程安全
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;  // DEFAULT_CAPACITY = 16
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);  //扩容因子 16*0.75
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

- 第二个阶段，存在hash冲突的情况

```java
synchronized (f) {
    if (tabAt(tab, i) == f) {
        if (fh >= 0) {
            binCount = 1;
            for (Node<K,V> e = f;; ++binCount) {
                K ek;
                if (e.hash == hash &&
                    ((ek = e.key) == key ||
                        (ek != null && key.equals(ek)))) {  // 如果是同一个key，就替换值
                    oldVal = e.val;
                    if (!onlyIfAbsent)
                        e.val = value;
                    break;
                }
                Node<K,V> pred = e;
                if ((e = e.next) == null) {    //不同的key，hash冲突了，就新添加一个节点
                    pred.next = new Node<K,V>(hash, key,
                                                value, null);
                    break;
                }
            }
        }
```

- 第三个阶段，元素个数的统计和更新
    - addCount()添加元素个数，初始化阶段
```java
private transient volatile long baseCount;   //在没有线程竞争的情况下，通过CAS操作更新元素个数
private transient volatile CounterCell[] counterCells; //在存在线程竞争的情况下，存储元素个数
```

- addCount()添加元素个数，元素个数更新阶段
    - 直接访问baseCount累加元素个数
    - 找到CounterCell[] 随机的某个数组下标位置， value= v+x --表示记录元素个数
    - 如果前面都失败，则进入到fullAndCount()
        - CountCell[] 为null
        - 已经初始化了，然后存在竞争，CAS进行更新
        - 如果CAS失败，触发扩容

```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {  //CAS操作修改baseCount的值，加1
        CounterCell a; long v; int m;
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
                U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            if (sc < 0) {   // 表示已经有线程在扩容
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0) // if（true） 表示不需要扩容
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) // 记录当前协助扩容线程数
                    transfer(tab, nt);  // 协助扩容
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                            (rs << RESIZE_STAMP_SHIFT) + 2)) //表示当前没有线程扩容
                transfer(tab, null);  // 扩容
            s = sumCount();
        }
    }
}
```
- CounterCell 静态内部类，sumCount（）方法统计元素个数
```java
@sun.misc.Contended static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}

final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

- 第四个阶段，扩容
    - 当元素个数大于阈值的时候，会触发扩容
    - 如果此时正在扩容，在扩容阶段进来的线程会协助扩容

- 第五阶段，数据迁移



