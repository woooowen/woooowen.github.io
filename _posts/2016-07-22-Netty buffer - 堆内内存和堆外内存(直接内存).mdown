---
layout: post
title: Netty buffer - 堆内内存和堆外内存(直接内存)
category: [JAVA]
tags: [JAVA,网络]
---

Netty中的内存管理从位置上可以分为堆内和堆外,从功能上可以分为池化和非池化,这边主要说堆内和堆外的区别

##### 池化继承关系图

![](http://pic.woowen.com/netty%20reference%20count.png)


##### 非池化继承关系图

##### 这边为了避免内存池的干扰,因此使用非池化代码分析

![非池化继承关系图](http://pic.woowen.com/netty%20unpool%20referencecount.png)


```JAVA

// 内存分配

protected ByteBuf newHeapBuffer(int initialCapacity, int maxCapacity) {
	
	// hasUnsafe为true,因此调用InstrumentedUnpooledUnsafeHeapByteBuf
    return PlatformDependent.hasUnsafe() ?
            new InstrumentedUnpooledUnsafeHeapByteBuf(this, initialCapacity, maxCapacity) :
            new InstrumentedUnpooledHeapByteBuf(this, initialCapacity, maxCapacity);
}

// 调用这边申请内存
protected UnpooledHeapByteBuf(ByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
    super(maxCapacity);

    checkNotNull(alloc, "alloc");

    if (initialCapacity > maxCapacity) {
        throw new IllegalArgumentException(String.format(
                "initialCapacity(%d) > maxCapacity(%d)", initialCapacity, maxCapacity));
    }

    this.alloc = alloc;
    setArray(allocateArray(initialCapacity));
    setIndex(0, 0);
}

// 其实本质就是一个字节数组
byte[] allocateArray(int initialCapacity) {
    return new byte[initialCapacity];
}

```

#### 因为堆内内存的申请和释放都通过jvm来操作heap实现,因此非常的方便省心


```JAVA

// 直接内存分配
protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
    final ByteBuf buf;
    // 可以调用unsafe,返回true
    if (PlatformDependent.hasUnsafe()) {
    	// 这边获取USE_DIRECT_BUFFER_NO_CLEANER 静态变量的值,默认应该是true
        buf = PlatformDependent.useDirectBufferNoCleaner() ?
                new InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf(this, initialCapacity, maxCapacity) :
                new InstrumentedUnpooledUnsafeDirectByteBuf(this, initialCapacity, maxCapacity);
    } else {
        buf = new InstrumentedUnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
    }
    return disableLeakDetector ? buf : toLeakAwareBuffer(buf);
}

// UnpooledUnsafeDirectByteBuf.java
protected UnpooledUnsafeDirectByteBuf(ByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
    super(maxCapacity);
    if (alloc == null) {
        throw new NullPointerException("alloc");
    }
    if (initialCapacity < 0) {
        throw new IllegalArgumentException("initialCapacity: " + initialCapacity);
    }
    if (maxCapacity < 0) {
        throw new IllegalArgumentException("maxCapacity: " + maxCapacity);
    }
    if (initialCapacity > maxCapacity) {
        throw new IllegalArgumentException(String.format(
                "initialCapacity(%d) > maxCapacity(%d)", initialCapacity, maxCapacity));
    }

    this.alloc = alloc;
    setByteBuffer(allocateDirect(initialCapacity), false);
}

// PlatformDependent0.java
static ByteBuffer allocateDirectNoCleaner(int capacity) {
    return newDirectBuffer(UNSAFE.allocateMemory(capacity), capacity);
}

```

```C

UNSAFE_ENTRY(jlong, Unsafe_AllocateMemory(JNIEnv *env, jobject unsafe, jlong size))
  UnsafeWrapper("Unsafe_AllocateMemory");
  size_t sz = (size_t)size;
  if (sz != (julong)size || size < 0) {
    THROW_0(vmSymbols::java_lang_IllegalArgumentException());
  }
  if (sz == 0) {
    return 0;
  }
  sz = round_to(sz, HeapWordSize);
  void* x = os::malloc(sz, mtInternal);
  if (x == NULL) {
    THROW_0(vmSymbols::java_lang_OutOfMemoryError());
  }
  //Copy::fill_to_words((HeapWord*)x, sz / HeapWordSize);
  return addr_to_java(x);
UNSAFE_END

```

#### 最后调用UNSAFE类获取其中的分配内存native方法交给jvm去分配,然后调用系统方法malloc分配内存

### 我们可以看到堆内内存是通过一个Byte数组去申请堆内内存,而堆外内存是通过系统函数去直接在内存上分配一块空间

### 其中的区别

* 堆内内存交给jvm全权管理,申请,释放本质就是一个字节数组,因此效率高,且释放通过GC直接去操作,而堆外内存(直接内存)位于jvm堆外的native堆,由操作系统进行内存分配和释放,效率较低
* 堆内内存因为会有GC的存在,因此跟socket交互的时候需要先复制一份到堆外内存,而堆外内存跟socket交互可以避免这一步的复制操作,因为进行I/O读写操作的时候,需要给几个参数,其中有内存地址,内存偏移量等等参数,而因为堆内有GC的存在,因此他的内存地址是不准确的,容易出错,而堆外的GC频率要低的多

