---
layout: post
title: ConcurrentHashMap实现线程安全
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

* 1.定义用到volatile的变量会组织jvm在编译的时候进行重排序优化操作
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
这可能也跟synchronized在后面的几个版本的优化有关,现在的synchroinized锁性能以及比ReentrantLock要好了很多.synchronized在更多的时候能保持一个更平衡的性能
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

一般cas的意思就是,当老的值跟我预期的值一样,那么久讲老的值更新为新的值

比如下面的代码
当SIZECTL的值跟我现在的一样,那么就将sc的新值赋值给他,至于后面的-1,应该是JNI参数相关的,毕竟是本地的方法,也只能通过反编译来看到源码
然而..我看到的定义是这个.

```JAVA

	public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
	// 看来大牛也为变量命名而头疼

```

```JAVA
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
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

####参考

java8环境下各种锁对比

<http://blog.takipi.com/java-8-stampedlocks-vs-readwritelocks-and-synchronized/>

