---
layout: post
title: JAVA8 LongAdder
category: [JAVA]
tags: [JAVA]
---

看了<http://coolshell.cn/articles/11454.html>的文章,发现里面的实现跟我自己的java版本不一致,而我自己的是最新的java版本,因此看看又做了哪些改动

我自己的JAVA版本

![](http://pic.woowen.com/longadder3.png)

```JAVA
	
	// 新的
	public void add(long x) {
        Cell[] as; long b, v; int m; Cell a;
        if ((as = cells) != null || !casBase(b = base, b + x)) {
            boolean uncontended = true;            
            if (as == null || (m = as.length - 1) < 0 ||            	
                (a = as[getProbe() & m]) == null ||
                !(uncontended = a.cas(v = a.value, v + x)))
                longAccumulate(x, null, uncontended);
        }
    }

```    

* 1.判断的条件变化了.原来的代码是下面这个图片,可以看到用```getProbe()```方法替换了原来的```threadHashCode.get()```

![](http://pic.woowen.com/longadder2.png)


* 2.当发生cas失败的时候执行的方法也做了变化,从原先的```retryUpdate``` 变成了```longAccumulate```

新的方法

```JAVA

 final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
        int h;
        // 如果probe为0,那么初始化一个ThreadLocalRandom线程,因为这个线程在初始化的时候threadLocalRandomProbe字段必然是有值的
        if ((h = getProbe()) == 0) {
        	// 提前初始化h的值,获取当前线程的probe值,实际上他就是一个hashcode
            ThreadLocalRandom.current(); // force initialization
            h = getProbe();
            wasUncontended = true;
        }
        // 是否发生了碰撞
        boolean collide = false;                // True if last slot nonempty
        for (;;) {
        	// 一直重试
            Cell[] as; Cell a; int n; long v;
            if ((as = cells) != null && (n = as.length) > 0) {
                if ((a = as[(n - 1) & h]) == null) {
                	// cellsBusy是用于在新建Cells或者给Cells扩容的时候加上的自旋锁
                    if (cellsBusy == 0) {       // Try to attach new Cell
                    	// 新建一个Cell
                        Cell r = new Cell(x);   // Optimistically create
                        // 先判断cellsBusy锁的值是否为0,如果是,那么将他通过cas改成1                        
                        if (cellsBusy == 0 && casCellsBusy()) {
                            boolean created = false;
                            // 这边是一个double-check,双重检查cellsBusy的值
                            // 开始
                            // 1.新创建cells不为空并且长度>0,表示新创建的cells创建成功
                            // 2.将new Cell(x)的值赋值给新的cell
                            // 3.新的cell创建成功,将cellsBusy锁释放                           
                            try {               // Recheck under lock
                                Cell[] rs; int m, j;
                                if ((rs = cells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
                            continue;           // Slot is now non-empty
                        }
                    }
                    collide = false;
                }
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash
                else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                             fn.applyAsLong(v, x))))
                    break;
                else if (n >= NCPU || cells != as)
                    collide = false;            // At max size or stale
                else if (!collide)
                    collide = true;
                else if (cellsBusy == 0 && casCellsBusy()) {
                    try {
                    	// 将当前的cell的大小增加一倍
                        if (cells == as) {      // Expand table unless stale
                            Cell[] rs = new Cell[n << 1];
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            cells = rs;
                        }
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
                // 更新probe的值
                h = advanceProbe(h);
            }
            else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
                boolean init = false;
                try {                           // Initialize table
                	// 最后如果更新成功,那么直接将cell(x)的值赋值
                    if (cells == as) {
                        Cell[] rs = new Cell[2];
                        rs[h & 1] = new Cell(x);
                        cells = rs;
                        init = true;
                    }
                } finally {
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }
            else if (casBase(v = base, ((fn == null) ? v + x :
                                        fn.applyAsLong(v, x))))
                break;                          // Fall back on using base
        }
    }


```

* 3.probe()方法, 可以认为是线程中增加了一个hashcode,并且当他产生碰撞的时候会使用伪随机(Marsaglia XorShift)去动态的调整自己的code

```JAVA

	static final int getProbe() {
        return UNSAFE.getInt(Thread.currentThread(), PROBE);
    }
    // 初始化的时候,利用反射,去获取threadLocalRandomProbe的值
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> sk = Striped64.class;
            BASE = UNSAFE.objectFieldOffset
                (sk.getDeclaredField("base"));
            CELLSBUSY = UNSAFE.objectFieldOffset
                (sk.getDeclaredField("cellsBusy"));
            Class<?> tk = Thread.class;
            PROBE = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomProbe"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }

```

这有就可以避免rehash的过程,从而优化了效率

longAdder从java1.8以来做了很多改进

* 1.通过多个Cell的方式优化多并发情况下的val更新,提高吞吐
* 2.通过线程的threadLocalRandomProbe字段中存储的probe值替换原来的hashcode方法,避免了每次增加计算hash的过程


### 参考
Xorshift随机算法
<https://en.wikipedia.org/wiki/Xorshift>

longAdder 原理
<http://coolshell.cn/articles/11454.html>

认识CPU Cache
<http://geek.csdn.net/news/detail/114619>