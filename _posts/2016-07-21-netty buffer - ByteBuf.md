---
layout: post
title: Netty buffer - ByteBuf
category: [源码]
tags: [Netty]
---

Netty 中的buffer是非常重要的一个部分,他重新实现了ByteBuffer相关的功能,并且做了很多的优化

ByteBuffer中包含了几个指针,limit position capacity mark用来表示缓冲区中的填充空间,但是因为读写都在一个空间中,所以每次写完读取的时候都要调用flip()方法,非常麻烦,并且每次需要提前分配内存,如果超过之前预订的大小就会报错,因此每次put之前都需要检查一下,如果空间不够,需要手动扩容,然后再复制,正因为nio中的Buffer比较难用,因此netty自己实现了这一块的功能

### 变化

#### 结构变化

##### 0.默认情况下,新申请的一块

```JAVA

+---------------------------------------------------------+
|             writable bytes (got more space)             |
+---------------------------------------------------------+
|                                                         |
0 = readerIndex = writerIndex            <=            capacity

```

这个时候readerIndex和writerIndex都为0,没有写入任何数据,且也无法读取任何数据.

##### 1.当写入一部分数据后

```JAVA

+-------------------+------------------+------------------+
| discardable bytes |  readable bytes  |  writable bytes  |
|                   |     (CONTENT)    |                  |
+-------------------+------------------+------------------+
|                   |                  |                  |
0      <=      readerIndex   <=   writerIndex    <=    capacity

```

* `discardable bytes` 表示废弃的区域,也就是已经读取过的,可以释放的区域,每次调用`readByte`会增加readerIndex
* `readable bytes` 表示已经写入,但是还没有读取过的可读区域
* `writable bytes` 表示可写区域

##### 2.当抛弃废弃区域之后

```JAVA

+------------------+--------------------------------------+
|  readable bytes  |    writable bytes (got more space)   |
+------------------+--------------------------------------+
|                  |                                      |
readerIndex (0) <= writerIndex (decreased)       <=    capacity

```

* 调用方法`discardReadBytes` 回收废弃区域
* 此刻`discardable bytes`区域被回收,那么`readable bytes`保持不变,`writeable bytes`增长

```JAVA

public ByteBuf discardReadBytes() {
	// 检查'io.netty.buffer.bytebuf.checkAccessible' 是否开启,默认为true
	// 主要为了检查是否还具有可抛弃对象,每次释放前都会进行检查,如果没有抛出异常,主要通过引用计数来判断
    ensureAccessible();
    if (readerIndex == 0) {
        return this;
    }

    if (readerIndex != writerIndex) {
    	// 读取部分区域
    	// 底层调用了System.arraycopy,将discardable bytes区域之外的重新拷贝并返回
        setBytes(0, this, readerIndex, writerIndex - readerIndex);
        writerIndex -= readerIndex;
        adjustMarkers(readerIndex);
        readerIndex = 0;
    } else {
    	// readerIndex = writerIndex 全部废弃
        adjustMarkers(readerIndex);
        writerIndex = readerIndex = 0;
    }
    return this;
}

```

##### 3.调用clear之后,返回初始状态,`readerIndex = writerIndex = 0`,调用clear只是重置了两个读写指针为0,没有其他操作,因此比discardReadBytes要快

```JAVA

// 直接重置两个指针
public ByteBuf clear() {
    readerIndex = writerIndex = 0;
    return this;
}


```

#### 扩容变化

在nio.bytebuffer中,每次需要申请内存的时候都要指定容量,如果超出容量范围需要重新申请一个更大的容量,然后再进行复制

```JAVA

ByteBuffer byteBuffer = ByteBuffer.allocate(25);
byteBuffer.put("hello netty,hello world".getBytes());

ByteBuffer byteBuffer1 = ByteBuffer.allocate(50);
byteBuffer1.put(byteBuffer);
byteBuffer1.put("hello netty,hello world".getBytes());

```

ByteBuf中增加了自动扩容机制,避免每次申请容量太大或者太小造成的浪费

```JAVA

private void ensureWritable0(int minWritableBytes) {
	if (minWritableBytes <= writableBytes()) {
	    return;
	}

	if (minWritableBytes > maxCapacity - writerIndex) {
	    throw new IndexOutOfBoundsException(String.format(
	            "writerIndex(%d) + minWritableBytes(%d) exceeds maxCapacity(%d): %s",
	            writerIndex, minWritableBytes, maxCapacity, this));
	}

	// Normalize the current capacity to the power of 2.
	int newCapacity = alloc().calculateNewCapacity(writerIndex + minWritableBytes, maxCapacity);

	// Adjust to the new capacity.
	capacity(newCapacity);
}

```