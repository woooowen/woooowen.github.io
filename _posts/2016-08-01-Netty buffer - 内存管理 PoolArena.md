---
layout: post
title: Netty buffer - 内存管理 PoolArena
category: [JAVA]
tags: [JAVA,网络]
---

### 总览
![](http://of83v97ri.bkt.gdipper.com/poolarena.png)


先熟悉下面两个变量

* reqCapacity 申请容量的大小,表示我们申请一个多大的heap所需要的容量
* normCapacity 根据我们需要的容量计算得出的应有容量,用来决定我们申请的heap属于哪一个部分

##### 容量计算代码

```JAVA

// 根据需要容量计算常规容量,也就是申请空间的容量所需的容量大于需要容量
int normalizeCapacity(int reqCapacity) {
    // 需要容量小于0,抛出异常
	if (reqCapacity < 0) {
	    throw new IllegalArgumentException("capacity: " + reqCapacity + " (expected: 0+)");
	}

	// 需要容量大于一个chunk的容量,默认直接返回当前需要容量
	if (reqCapacity >= chunkSize) {
	    return directMemoryCacheAlignment == 0 ? reqCapacity : alignCapacity(reqCapacity);
	}

	// 需要的容量不属于tiny也就是 >= 512 字节
	if (!isTiny(reqCapacity)) { // >= 512
	    // Doubled
	    // 当前需要容量 >= 512 ,<= chunkSize(16777216),
	    // 那么计算出当前容量的double值,并且这个值是16的指数,也就是2的指数
	    int normalizedCapacity = reqCapacity;
	    normalizedCapacity --;
	    normalizedCapacity |= normalizedCapacity >>>  1;
	    normalizedCapacity |= normalizedCapacity >>>  2;
	    normalizedCapacity |= normalizedCapacity >>>  4;
	    normalizedCapacity |= normalizedCapacity >>>  8;
	    normalizedCapacity |= normalizedCapacity >>> 16;
	    normalizedCapacity ++;

	    if (normalizedCapacity < 0) {
	        normalizedCapacity >>>= 1;
	    }
	    assert directMemoryCacheAlignment == 0 || (normalizedCapacity & directMemoryCacheAlignmentMask) == 0;

	    return normalizedCapacity;
	}

	if (directMemoryCacheAlignment > 0) {
	    return alignCapacity(reqCapacity);
	}

	// 容量 为16的指数级,16,32,64,那么直接返回,不需要额外计算	
	if ((reqCapacity & 15) == 0) {
	    return reqCapacity;
	}
    // 容量 < 16,那么容量就直接为16
	return (reqCapacity & ~15) + 16;
}

```

上面的方法主要用来计算分配的空间大小

* 小于0 ,肯定有问题,直接抛异常
* 大于chunkSize,这部分不会加入内存池中,当jvm自己去分配
* 大于512,但是小于chunkSize,那么讲他扩容一倍,并且这个值是16的指数级
* 如果需要的容量直接就是16指数级别,那么直接返回
* 如果容量小于16,那么直接返回16

总之就是用来保证分配的最小容量为16,或者16的指数级.netty中最小分配空间是一个subPage,也就是16字节

下么根据需要容量计算后的大小来决定将这一块内存分配到哪个模块下面

#### PoolSubPage

* tinySubPagePools 包含了一个subPage数组中,用于分配小于512容量的内存,内存分配最小为一个subPage(16),其中最多有32个subPage,16*32=512
* smallSubPagePools 用于分配大于512容量的内存,每个subPage容量为2048,默认有4个,2048*4=8192,正好是一个page的容量

```JAVA

private final PoolSubpage<T>[] tinySubpagePools;
private final PoolSubpage<T>[] smallSubpagePools;

```


