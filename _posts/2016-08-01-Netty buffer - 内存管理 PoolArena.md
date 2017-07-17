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

#### 分配逻辑

```JAVA

private void allocate(PoolThreadCache cache, PooledByteBuf<T> buf, final int reqCapacity) {
    final int normCapacity = normalizeCapacity(reqCapacity);
    // 属于tinySubPagePools或者smallSubPagePools
    if (isTinyOrSmall(normCapacity)) { // capacity < pageSize
        int tableIdx;
        // 用于存放subPage
        PoolSubpage<T>[] table;
        boolean tiny = isTiny(normCapacity);
        if (tiny) { // < 512
        	// 小于512,那么分配到tinySubPagePools中
            if (cache.allocateTiny(this, buf, reqCapacity, normCapacity)) {
                // was able to allocate out of the cache so move on
                return;
            }
            tableIdx = tinyIdx(normCapacity);
            table = tinySubpagePools;
        } else {
        	// 大于512但是小于pageSize(8192)就分配到smallSubPagePools中
            if (cache.allocateSmall(this, buf, reqCapacity, normCapacity)) {
                // was able to allocate out of the cache so move on
                return;
            }
            tableIdx = smallIdx(normCapacity);
            table = smallSubpagePools;
        }

        final PoolSubpage<T> head = table[tableIdx];

        /**
         * Synchronize on the head. This is needed as {@link PoolChunk#allocateSubpage(int)} and
         * {@link PoolChunk#free(long)} may modify the doubly linked list as well.
         */
        synchronized (head) {
            final PoolSubpage<T> s = head.next;
            if (s != head) {
                assert s.doNotDestroy && s.elemSize == normCapacity;
                long handle = s.allocate();
                assert handle >= 0;
                s.chunk.initBufWithSubpage(buf, handle, reqCapacity);
                incTinySmallAllocation(tiny);
                return;
            }
        }
        synchronized (this) {
        	// 初始化subpage的时候会新建一个chunk,然后塞入qinit中
            allocateNormal(buf, reqCapacity, normCapacity);
        }

        incTinySmallAllocation(tiny);
        return;
    }
    if (normCapacity <= chunkSize) {
        if (cache.allocateNormal(this, buf, reqCapacity, normCapacity)) {
            // was able to allocate out of the cache so move on
            return;
        }
        synchronized (this) {
            allocateNormal(buf, reqCapacity, normCapacity);
            ++allocationsNormal;
        }
    } else {
		// 所需分配大于chunkSize的值, > 16777216 这块是不会被加入内存池的,也不会缓存       
        allocateHuge(buf, reqCapacity);
    }
}

```

#### allocateNormal

```JAVA

private void allocateNormal(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
	if (q050.allocate(buf, reqCapacity, normCapacity) || q025.allocate(buf, reqCapacity, normCapacity) ||
	    q000.allocate(buf, reqCapacity, normCapacity) || qInit.allocate(buf, reqCapacity, normCapacity) ||
	    q075.allocate(buf, reqCapacity, normCapacity)) {
	    return;
	}

	// Add a new chunk.
	PoolChunk<T> c = newChunk(pageSize, maxOrder, pageShifts, chunkSize);
	long handle = c.allocate(normCapacity);
	assert handle > 0;
	c.initBuf(buf, handle, reqCapacity);
	qInit.add(c);
}

```

调用allocateNormal会去qInit,q000等中去先调用,初始化的时候因为这些chunkList都为空,因此会去先创建一个chunk,然后塞入qInit中,方便下次来调用

超过chunkSize的申请会直接去调用allocateHuge执行,因为超过这部分的空间不会放入内存池中,因此不用理会cache,直接执行
让jvm去分配一块指定容量的字节数组

```JAVA

 private void allocateHuge(PooledByteBuf<T> buf, int reqCapacity) {
    PoolChunk<T> chunk = newUnpooledChunk(reqCapacity);
    activeBytesHuge.add(chunk.chunkSize());
    buf.initUnpooled(chunk, reqCapacity);
    allocationsHuge.increment();
}

protected PoolChunk<byte[]> newUnpooledChunk(int capacity) {
	return new PoolChunk<byte[]>(this, new byte[capacity], capacity, 0);
}

```

##### 其他

在初始化的过程中也会同时初始化其他的一些参数

例如

```allocationsTiny``` 等用来表明已经分配的tinySubPools次数
```allocationsSmall``` 用来表明已经分配过smallSubPools次数
