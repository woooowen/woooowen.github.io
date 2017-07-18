---
layout: post
title: Netty buffer - 内存管理 PoolChunkList
category: [JAVA]
tags: [JAVA,网络]
---

PoolChunkList是一堆PoolChunk的集合,主要用来管理PoolChunk

```JAVA

// PoolChunkList是属于arena的.每个arena管理多个PoolChunkList
private final PoolArena<T> arena;
// 指向下一个chunkList
private final PoolChunkList<T> nextList;
// 最小使用率
private final int minUsage;
// 最大使用率
private final int maxUsage;
// 最大容量
private final int maxCapacity;
// PoolChunkList中的头,也就是第一个PoolChunk
private PoolChunk<T> head;

```

#### 构造函数

```JAVA

PoolChunkList(PoolArena<T> arena, PoolChunkList<T> nextList, int minUsage, int maxUsage, int chunkSize) {
	// 断言判断
    assert minUsage <= maxUsage;
    this.arena = arena;
    this.nextList = nextList;
    this.minUsage = minUsage;
    this.maxUsage = maxUsage;
    // 计算最大容量
    maxCapacity = calculateMaxCapacity(minUsage, chunkSize);
}

```

```JAVA

q100 = new PoolChunkList<T>(this, null, 100, Integer.MAX_VALUE, chunkSize);
q075 = new PoolChunkList<T>(this, q100, 75, 100, chunkSize);
q050 = new PoolChunkList<T>(this, q075, 50, 100, chunkSize);
q025 = new PoolChunkList<T>(this, q050, 25, 75, chunkSize);
q000 = new PoolChunkList<T>(this, q025, 1, 50, chunkSize);
qInit = new PoolChunkList<T>(this, q000, Integer.MIN_VALUE, 25, chunkSize);

q100.prevList(q075);
q075.prevList(q050);
q050.prevList(q025);
q025.prevList(q000);
q000.prevList(null);
qInit.prevList(qInit);

List<PoolChunkListMetric> metrics = new ArrayList<PoolChunkListMetric>(6);
metrics.add(qInit);
metrics.add(q000);
metrics.add(q025);
metrics.add(q050);
metrics.add(q075);
metrics.add(q100);
chunkListMetrics = Collections.unmodifiableList(metrics);

```

![](http://of83v97ri.bkt.gdipper.com/poolchunklistinarena.png)

![](http://of83v97ri.bkt.gdipper.com/poolchunksize.png)


#### PoolChunkList的分配

```JAVA

boolean allocate(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
	// 如果不是初始化过程,那么从poolchunklist中直接获取存在的chunk去分配内存
    if (head == null || normCapacity > maxCapacity) {                    	
        return false;
    }

    // 否则遍历列表
    for (PoolChunk<T> cur = head;;) {
        long handle = cur.allocate(normCapacity);
        if (handle < 0) {
        	// 分配失败,移到下一个poolchunk
            cur = cur.next;
            if (cur == null) {
                return false;
            }
        } else {
        	// 分配成功
            cur.initBuf(buf, handle, reqCapacity);
            // 检查当前chunk的利用率释放大于他本身的最大利用率
            if (cur.usage() >= maxUsage) {
            	// 如果大于,那么就移除
                remove(cur);
                // 将他加到下一个poolchunk中去
                nextList.add(cur);
            }
            return true;
        }
    }
}

```

#### 释放过程

```JAVA

boolean free(PoolChunk<T> chunk, long handle) {
    chunk.free(handle);
    // 依然检查利用率
    if (chunk.usage() < minUsage) {
        remove(chunk);        
        return move0(chunk);
    }
    return true;
}

```

>在分配和释放的过程中,因为本身内存分配是连续的一大块,而需要的内存往往比实际申请的容量要小,因此很容易产生很多碎片,从而造成内存的浪费,不断的将高利用率的chunk移动到list尾部,而将低利用率的移动到前面,随着内存的释放,再将低利用率的移动到list前面从而提高分配的成功率



