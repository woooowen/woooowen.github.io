---
layout: post
title: JAVA8 ConcurrentHashMap
category: [JAVA]
tags: [JAVA]
---

#### 类似HashTable,跟HashMap不一样,ConcurrentHashMap不允许Null的key和value

#### 计算table的容量,一般是初始化ConcurrentHashMap容量的时候使用,
只有当容量超过2的指数的时候,该容量就会跟着翻倍
比如当c = 8的时候容量等于7
当c = 9的时候,容量等于15
当c = 16的时候,容量还是等于15
但是当c = 17的时候,容量就会翻倍为31

```JAVA
private static final int tableSizeFor(int c) {
    int n = c - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

#### 主要看下为什么ConcurrentHashMap是线程安全的,如何实现的

类中大量使用了volatile关键字,定义volatile变量,他可以保证该变量对每个线程的可见性,就是保证每个线程看到volatile的值都是一样的.

* 1.定义用到volatile的变量会组织jvm在编译的时候禁止重排序优化操作
* 2.定义volatile变量,其他线程每次获取的时候会放弃本地内存中的值,直接去主内存中捞取
* 3.他每次更新的时候会调用JNI,也就是本地方法,其实就是CPU级别的CAS方法去更新这个值来保证他的原子性,具体的后面再说,先了解这么多就行了

```JAVA

final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);

        else {
            V oldVal = null;
            synchronized (f) {
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
                    treeifyBin(tab, i);
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

使用了synchronized来保证线程的安全,需要注意的是,他的基本实现其实跟HashMap是差不多的.当binCount >= 8的时候,也会转换成红黑树,增加性能

```JAVA

static class Segment<K,V> extends ReentrantLock implements Serializable {
    private static final long serialVersionUID = 2249069246763182397L;
    final float loadFactor;
    Segment(float lf) { this.loadFactor = lf; }
}

```

在1.8之前的版本中,ConcurrentHashMap使用分段锁Segment来保证并发情况下的线程安全,但是在1.8的时候已经剔除了.只保留了一些初始化工作,为了保证向前兼容,网上的很多资料还停留在之前的版本,这个需要注意.
这可能也跟synchronized在后面的几个版本的优化有关,现在的synchroinized锁性能已经比ReentrantLock要好了很多.synchronized在更多的时候能保持一个更平衡的性能
#### 各种锁性能对比
#####5个读进程,5个写进程场景下的性能(数字越小,性能越好)
![](http://pic.woowen.com/lockvs1.png)

#####10个读写进程场景下的性能对比
![](http://pic.woowen.com/lockvs2.png)

#####16个读进程,4个写进程场景下的性能对比
![](http://pic.woowen.com/lockvs3.png)

#####19个读进程,1个写进程场景下的性能对比
![](http://pic.woowen.com/lockvs4.png)

可见,synchronized在各个场景下的性能都比较平衡,且他是在jvm中实现的.未来版本肯定也是优先去优化这个关键字的,因此还是推荐使用他更多一些
所以ConcurrentHashMap保证线程安全的几个要素

* 1.volatile变量
* 2.synchronized同步块
* 3.Unsafe方法中的cas JNI方法,这个方法在初始化以及原子自增的时候会用到.

一般cas的意思就是,当老的值跟我预期的值一样,那么就将老的值更新为新的值

比如下面的代码

当SIZECTL的值(老的值)跟新给的sc一样,那么就将-1赋值给SIZECTL

```JAVA

	public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
	// 看来大牛也为变量命名而头疼

```

```JAVA

private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            // 让出线程资源,保证只有一个线程能初始化table
            Thread.yield(); 
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
        // 用unsafe的cas方法来原子性的给SIZECTL赋值
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}

private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
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
            if (sc < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}

```

#### 如何保证线程安全

在initTable的时候初始化table,通过cas修改sizeCtl的值,有且只有一个线程能获得锁,其他的线程通过Thread.yield让出CPU执行资源

ConcurrentHashmap通过Synchronized和自旋cas来保证容器修改的线程安全

扩容

通过单线程构造两倍容量的nexttable,然后多线程协助复制数据的方法来安全扩容

```JAVA

static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        super(MOVED, null, null, null);
        this.nextTable = tab;
    }

    Node<K,V> find(int h, Object k) {
        // loop to avoid arbitrarily deep recursion on forwarding nodes
        outer: for (Node<K,V>[] tab = nextTable;;) {
            Node<K,V> e; int n;
            if (k == null || tab == null || (n = tab.length) == 0 ||
                (e = tabAt(tab, (n - 1) & h)) == null)
                return null;
            for (;;) {
                int eh; K ek;
                if ((eh = e.hash) == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
                if (eh < 0) {
                    if (e instanceof ForwardingNode) {
                        tab = ((ForwardingNode<K,V>)e).nextTable;
                        continue outer;
                    }
                    else
                        return e.find(h, k);
                }
                if ((e = e.next) == null)
                    return null;
            }
        }
    }
}

```

大概意思就是,扩容操作一般分为两步,先构造一个两倍于本身的数组,然后将老的node复制到新数组中

单线程构建新容器,然后遍历老容器往新容器中复制元素,当遍历复制的时候,线程对当前节点加Synchronized锁保证同步,并且将该节点换成forwardingNode,当多线程环境下,其他线程参与扩容的时候看到该节点为forwardingNode,说明有其他线程正在进行复制,那么就跳过该节点,处理下一个节点的复制操作,如此完成多线程环境下的扩容复制操作


####参考

java8环境下各种锁对比

<http://blog.takipi.com/java-8-stampedlocks-vs-readwritelocks-and-synchronized/>

ConcurrentHashmap

<http://blog.csdn.net/u010723709/article/details/48007881>
