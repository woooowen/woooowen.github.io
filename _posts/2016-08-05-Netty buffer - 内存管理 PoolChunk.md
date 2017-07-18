---
layout: post
title: Netty buffer - 内存管理 PoolChunk
category: [JAVA]
tags: [JAVA,网络]
---

#### 了解以下术语对于理解代码非常有帮助

* page : chunk指内存块,而page则是内存块中可以分配的最小单元
* chunk : 一个chunk是多个page的集合
* chunkSize :  chunkSize = 2^{maxOrder} * pageSize,PoolChunk中每个chunk的大小通过前面的计算方式得出


为了简单期间,当我们需要申请的容量>=pageSize的时候,总是根据计算得出>=pageSize且是2的指数级的最接近的数

##### 比如我们要申请的容量是12,那么设定申请容量为16,如果我们要申请的容量是40,那么设定申请的容量就为64,如果是400,那么就是512,以此类推,总是保证在内存中分配的容量大于我们要申请的容量,且该容量为2的指数级,主要用来方便后续的位运算

内部构造一个平衡二叉树,来方便搜索

例如

depth=0        1 node (chunkSize)
depth=1        2 nodes (chunkSize/2)
..
..
depth=d        2^d nodes (chunkSize/2^d)
..
depth=maxOrder 2^maxOrder nodes (chunkSize/2^{maxOrder} = pageSize)

### 初始化PoolChunk

```JAVA

PoolChunk(PoolArena<T> arena, T memory, int pageSize, int maxOrder, int pageShifts, int chunkSize, int offset) {
    unpooled = false;
    // 未分配的一个arena
    this.arena = arena;
    // 未分配的一块内存
    this.memory = memory;
    // 每页的大小,默认8192 2^13
    this.pageSize = pageSize;
    // 2^13 ,默认为13,page指数
    this.pageShifts = pageShifts;
    // 默认11,二叉树中叶子节点的层数
    this.maxOrder = maxOrder;
    // 每个chunk的大小
    this.chunkSize = chunkSize;
    this.offset = offset;
    // 叶子节点maxOrder为11,unusable为maxOrder+1 = 12,也就是当这个节点不可再分配时会将他改成unusable,层级对应叶子节点层数+1
    unusable = (byte) (maxOrder + 1);
    log2ChunkSize = log2(chunkSize);
    subpageOverflowMask = ~(pageSize - 1);
    // 剩余字节,表示一个chunk的可用容量剩余
    freeBytes = chunkSize;

    assert maxOrder < 30 : "maxOrder should be < 30, but is: " + maxOrder;
    // 最大subpage 2048个
    maxSubpageAllocs = 1 << maxOrder;

    // byte[4096]
    memoryMap = new byte[maxSubpageAllocs << 1];
    // byte[4096] 用于记录每个节点对应的树深度,层级
    depthMap = new byte[memoryMap.length];
    int memoryMapIndex = 1;
    // 从树根节点开始遍历
    for (int d = 0; d <= maxOrder; ++ d) {
        int depth = 1 << d;
        for (int p = 0; p < depth; ++ p) {
            // 初始化2个数组都是树节点index对应树深度
            // 例如第一个节点(根节点)对应0
            // 第二个节点对应1
            // 因为是平衡二叉树,所以第三个节点对应也是1
            // 第四个节点就对应2
            memoryMap[memoryMapIndex] = (byte) d;
            depthMap[memoryMapIndex] = (byte) d;
            memoryMapIndex ++;
        }
    }

    subpages = newSubpageArray(maxSubpageAllocs);
}

```

#### 通过poolarena我们知道凡是调用超过8k的内存,会调用```allocateNormal```方法(前提是没有超过chunkSize),因为需要的内存会根据实际申请内存容量去计算,因此哪怕小于8k也会最终的到8k的内存

#### 比如  reqCapacity(实际需要的内存) = 8000,得到计算之后就为8192,因此也会执行allocateRun方法,而reqCapacity = 4000,得到计算之后的容量为4096,才会走allocateSubpage逻辑

```JAVA

long allocate(int normCapacity) {
    if ((normCapacity & subpageOverflowMask) != 0) { // >= pageSize
    	// 超过pageSize(8192)
        return allocateRun(normCapacity);
    } else {
    	// 没超过pageSize
        return allocateSubpage(normCapacity);
    }
}

```

### allocateRun

根据需要申请的容量开始分配内存

```JAVA

private long allocateRun(int normCapacity) {
	// 根据计算得出当前需要容量应该从哪一层开始分配
    int d = maxOrder - (log2(normCapacity) - pageShifts);
    // 根据得到的层,开始分配
    int id = allocateNode(d);
    if (id < 0) {
        return id;
    }
    freeBytes -= runLength(id);
    return id;
}

```    

```JAVA

private int allocateNode(int d) {
    int id = 1;
    int initial = - (1 << d); // has last d bits = 0 and rest all = 1
    // 获取memoryMap中对应的值
    // 初始状态memoryMap[1] = 1
    byte val = value(id);
    // 当memoryMap中的值 > 分配的层级d时表示已经不能在分配了.直接返回
    if (val > d) { // unusable
        return -1;
    }
    while (val < d || (id & initial) == 0) { 
    	// id每次遍历翻倍,用于循环下一层
        id <<= 1;
        // 根据下一层id,去memoryMap中获取所在层级是否为需要申请的空间的层级
        val = value(id);
        // 如果当前节点>d,说明该节点已经被分配了.
        if (val > d) {
        	// 找到它的兄弟节点去分配
            id ^= 1;
            val = value(id);
        }
    }
    byte value = value(id);
    assert value == d && (id & initial) == 1 << d : String.format("val = %d, id & initial = %d, d = %d",
            value, id & initial, d);
    // 得到需要的id,然后将他设置为已使用,后续不可分配
    // unusable = maxOrder + 1
    setValue(id, unusable); // mark as unusable
    updateParentsAlloc(id);
    return id;
}

```

#### 流程如下

![](http://of83v97ri.bkt.gdipper.com/poolchunktreealloc.png)

分配完成之后,给当前获得的节点标记unusable(就是仔memoryMap中将他的值设置为unusable)

```JAVA

private void updateParentsAlloc(int id) {
	// 分配的节点id > 1
    while (id > 1) {
    	// 获取当前节点的父节点, 右偏移1位,乘以2
        int parentId = id >>> 1;
        // 在memoryMap中获得当前id的值
        byte val1 = value(id);
        // 得到当前节点兄弟节点在memoryMap中的值
        byte val2 = value(id ^ 1);
        // 比较两个节点,如果当前节点大于兄弟节点,那么将他的值赋值给他的父节点
        byte val = val1 < val2 ? val1 : val2;
        setValue(parentId, val);
        id = parentId;
    }
}


```

#### 流程如下

![](http://of83v97ri.bkt.gdipper.com/poolchunknode%20alloc.png)

#### 分配

```JAVA

void initBuf(PooledByteBuf<T> buf, long handle, int reqCapacity) {
    int memoryMapIdx = memoryMapIdx(handle);
    // handel为之前获取到的节点的index
    // bitmap用于记录每个节点的分配情况,0表示未分配,1表示分配
    int bitmapIdx = bitmapIdx(handle);
    // 得知该节点还没有被分配
    if (bitmapIdx == 0) {
        byte val = value(memoryMapIdx);
        // 且在二叉树中记录该节点的值已经为12,表示前面的逻辑已经走完了.
        assert val == unusable : String.valueOf(val);
        // 开始,这边其实只是将之前就已经初始化完成的东西,例如chunk,memory,cache,handle等参数赋值给当前对象
        buf.init(this, handle, runOffset(memoryMapIdx) + offset, reqCapacity, runLength(memoryMapIdx),
                 arena.parent.threadCache());
    } else {
        initBufWithSubpage(buf, handle, bitmapIdx, reqCapacity);
    }
}

```

### 到此,allocateRun 已经分配完毕,下么看下allocateSubpage

```JAVA

private long allocateSubpage(int normCapacity) {
    // Obtain the head of the PoolSubPage pool that is owned by the PoolArena and synchronize on it.
    // This is need as we may add it back and so alter the linked-list structure.
    PoolSubpage<T> head = arena.findSubpagePoolHead(normCapacity);
    synchronized (head) {
        int d = maxOrder; 
        // 跟allocateRun逻辑相似,底层也是通过二叉树维护节点来分配
        int id = allocateNode(d);
        if (id < 0) {
            return id;
        }

        final PoolSubpage<T>[] subpages = this.subpages;
        final int pageSize = this.pageSize;

        freeBytes -= pageSize;

        int subpageIdx = subpageIdx(id);
        // 所有的subPage被存入一个数组中统一维护
        PoolSubpage<T> subpage = subpages[subpageIdx];
        // 如果这个数组为null,那么需要新建一个poolsubpage,并且最终在塞进去,否则直接从poolsubpages中分配
        if (subpage == null) {
            subpage = new PoolSubpage<T>(head, this, id, runOffset(id), pageSize, normCapacity);
            subpages[subpageIdx] = subpage;
        } else {
        	// 从现有的进行分配
            subpage.init(head, normCapacity);
        }
        return subpage.allocate();
    }
}


```

```JAVA

PoolSubpage(PoolSubpage<T> head, PoolChunk<T> chunk, int memoryMapIdx, int runOffset, int pageSize, int elemSize) {
    this.chunk = chunk;
    this.memoryMapIdx = memoryMapIdx;
    this.runOffset = runOffset;
    this.pageSize = pageSize;
    // 初始化的时候bitmap为new long[8]
    bitmap = new long[pageSize >>> 10]; // pageSize / 16 / 64
    init(head, elemSize);
}

void init(PoolSubpage<T> head, int elemSize) {
	// 假设我们需要的为4096大小的内存块
    doNotDestroy = true;
    this.elemSize = elemSize;
    if (elemSize != 0) {
    	// 那么将一个page分成2份分别来存储
        maxNumElems = numAvail = pageSize / elemSize;
        nextAvail = 0;
        bitmapLength = maxNumElems >>> 6;
        if ((maxNumElems & 63) != 0) {
            bitmapLength ++;
        }

        for (int i = 0; i < bitmapLength; i ++) {
            bitmap[i] = 0;
        }
    }
    addToPool(head);
}

long allocate() {
    if (elemSize == 0) {
        return toHandle(0);
    }

    if (numAvail == 0 || !doNotDestroy) {
        return -1;
    }

    final int bitmapIdx = getNextAvail();
    int q = bitmapIdx >>> 6;
    int r = bitmapIdx & 63;
    assert (bitmap[q] >>> r & 1) == 0;
    // 在bitmap中标记这块已经被分配
    bitmap[q] |= 1L << r;

    if (-- numAvail == 0) {
        removeFromPool();
    }

    return toHandle(bitmapIdx);
}

```

* PoolSubpage 在划分内存的时候会根据需要申请的容量,将8192(一个page)划分成多块存储,例如我申请的容量为4000,通过计算后给的容量为4096,这时通过计算,pageSize/申请容量.就得到为2,也就是我申请的那块内存,将一个page划分成了2块
* PoolSubpage中会有一个long[8]数组用来记录当前分配记录,[0,0,0,0,0,0,0,0],第一块被分配那么就变成[1,0,0,0,0,0,0,0,0]

#### 梳理
* 假设每次申请的容量为40,那么经过计算之后系统自动申请的容量就为48
* pageSize / elemSize = 8192 / 48 = 170,也就是一个page被分成了170快
* 每次分配完一块之后numAvail(表示剩余块数)都会减少1个,然后bitmap中的long型左偏移一位+原值记录
* [1,0,0..0] => [3,0,0..0] => [7,0,0..0] => [15,0,0..0],每个long值能记录64位,当第一位记录完毕,就会变成第二个
* [-1,1,0..0] => [-1,3,0..0] => [-1,7,0..0] 以此类推
* runOffset 值增加8192用于表示page的数量,一页分配完毕会继续分配下一页,然后上层是分配完16MB的chunk,再分配下一个chunk

