---
layout:     post
title:      ConCurrentHashMap学习笔记
subtitle:   ConCurrentHashMap学习笔记
date:       2019-01-03
author:     WPF
header-img: img/post-bg-cook.jpg
catalog: true
tags:
- 散列表
- ConCurrentHashMap
- java
---

## 概述
1. JDK1.8底层是散列表+红黑树
2. ConCurrentHashMap支持高并发的访问和更新，它是线程安全的
3. 检索操作不用加锁，get方法是非阻塞的
4. key和value都不允许为null

## JDK1.7底层实现
JDK1.7的底层是：segments+HashEntry数组：

![](https://i.imgur.com/Xhimkj8.png)
Segment继承了ReentrantLock,每个片段都有了一个锁，叫做“锁分段”
## Hashtable和ConCurrentHashMap
1. Hashtable是在每个方法上都加上了Synchronized完成同步，效率低下。
2. ConcurrentHashMap通过在部分加锁和利用CAS算法来实现同步。
## CAS算法
CAS（比较与交换，Compare and swap） 是一种有名的无锁算法
CAS有3个操作数
* 内存值V
* 旧的预期值A
* 要修改的新值B
当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做
当多个线程尝试使用CAS同时更新同一个变量时，只有其中一个线程能更新变量的值(A和内存值V相同时，将内存值V修改为B)，而其它线程都失败，失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次尝试(否则什么都不做)
看了上面的描述应该就很容易理解了，先比较是否相等，如果相等则替换(CAS算法)
## put 实现

```java

    //获取hash
    static final int spread(int h) {
        return (h ^ (h >>> 16)) & HASH_BITS;
    }
    
    //初始化table ,只让一个线程对散列表进行初始化.
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)
                //如果有线程正在初始化，则拒绝其他线程的进入
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) { // 设置值为-1，说明本线程正在初始化
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);// ==0.75*n
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
    
    // 获取节点
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }

    // 替换节点
    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }

    // 设置节点
    static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
    }
    
    // 帮助扩容
    final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            int rs = resizeStamp(tab.length);
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }

    public V put(K key, V value) {
            return putVal(key, value, false);
        }

    /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());// 获取hash值
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();// 初始化 
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                // 如果桶位置为空，则使用CAS更新数据的方式直接插入，这里不需要加锁，如果并发修改了，循环继续即可
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)// 正在扩容扩容
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) {// 对桶首节点加锁
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);// 树化
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
    ```


