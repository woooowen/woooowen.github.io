---
layout: post
title: JAVA8 Atomic类型
category: [JAVA]
tags: [JAVA]
---

Atomic 类型主要的特质就是能保持其原子性并且线程安全

```JAVA

	private volatile int value;

  	public final int get() {
        return value;
    }

	public final void set(int newValue) {
        value = newValue;
    }

```

原子性操作

```JAVA
	
	// 先取后设置
	public final int getAndSet(int newValue) {
        return unsafe.getAndSetInt(this, valueOffset, newValue);
    }

    // 先取后自增
    public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }

    // 先取后自减
    public final int getAndDecrement() {
        return unsafe.getAndAddInt(this, valueOffset, -1);
    }

    // 先取后增加
    public final int getAndAdd(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta);
    }

    // 先取后更新
    public final int getAndUpdate(IntUnaryOperator updateFunction) {
        int prev, next;
        do {
            prev = get();
            next = updateFunction.applyAsInt(prev);
        } while (!compareAndSet(prev, next));
        return prev;
    }

```

基本都是根据unsafe类中提供的JNI方法实现原子性操作,通过调用C++的本地方法实现

Unsafe都是直接基于内存去操作比较危险.可以利用反射的机制去声明他的实例

AtomicInteger,AtomicBoolean,AtomicLong几个原子性的类型实现机制都一样


#### 参考
Unsafe详解
<http://www.cnblogs.com/chenpi/p/5389254.html>


