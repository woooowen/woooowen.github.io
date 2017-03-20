---
layout: post
title: JAVA8 ArrayBlockingQueue
category: [JAVA]
tags: [JAVA]
---

```JAVA
// 数组队列
final Object[] items;
```

```JAVA
// 入队
private void enqueue(E x) {
    final Object[] items = this.items;
    // 插入队列
    items[putIndex] = x;
    // 如果队列中只有一个值那么下标重置为0
    if (++putIndex == items.length)
        putIndex = 0;
    // 队列元素值+1
    count++;
    // 信号量,take操作会去取队列,如果队列为空那么回阻塞线程,这个就是给take一个信号,表示队列中已经有新的元素了,你可以取消阻塞了
    notEmpty.signal();
}

// 出队
private E dequeue() {       
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    // 获取需要取的对象
    E x = (E) items[takeIndex];
    // 将要取的下标队列置为空
    items[takeIndex] = null;
    // 如果元素还有一个值那么下标重置为0
    if (++takeIndex == items.length)
        takeIndex = 0;
    // 队列中的对象数-1
    count--;
    if (itrs != null)
        itrs.elementDequeued();
    // 给put一个信号,表示队列已经剔除了一个对象,可以put
    notFull.signal();
    return x;
}

```

### add,offer,put 添加操作

```JAVA
public boolean add(E e) {
    return super.add(e);
}

public boolean offer(E e) {
    checkNotNull(e);
    // 阻塞队列本质也是用的可重入锁实现阻塞
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
    	// 阻塞队列一开始定义一个固定长度的数组,如果队列中的元素数量跟数组长度一致,表示队列已经满了.无法添加
        if (count == items.length)
            return false;
        else {
            enqueue(e);
            return true;
        }
    } finally {
        lock.unlock();
    }
}

public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        // 如果队列已经满了,那么线程阻塞,这也是put跟offer最大的不同
        while (count == items.length)
            notFull.await();
        enqueue(e);
    } finally {
        lock.unlock();
    }
}

```

### poll,take,remove

```JAVA

public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return (count == 0) ? null : dequeue();
    } finally {
        lock.unlock();
    }
}

public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        // 当队列中对象数为0,那么线程阻塞
        while (count == 0)
            notEmpty.await();
        return dequeue();
    } finally {
        lock.unlock();
    }
}

// 移除指定位置
public boolean remove(Object o) {
    if (o == null) return false;
    final Object[] items = this.items;
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        if (count > 0) {
            final int putIndex = this.putIndex;
            int i = takeIndex;
            do {
                if (o.equals(items[i])) {
                    removeAt(i);
                    return true;
                }
                if (++i == items.length)
                    i = 0;
            } while (i != putIndex);
        }
        return false;
    } finally {
        lock.unlock();
    }
}

public void remove() {
    final ReentrantLock lock = ArrayBlockingQueue.this.lock;
    lock.lock();
    try {
        if (!isDetached())
            incorporateDequeues(); // might update lastRet or detach
        final int lastRet = this.lastRet;
        this.lastRet = NONE;
        if (lastRet >= 0) {
            if (!isDetached())
                removeAt(lastRet);
            else {
                final E lastItem = this.lastItem;                
                this.lastItem = null;
                if (itemAt(lastRet) == lastItem)
                    removeAt(lastRet);
            }
        } else if (lastRet == NONE)
            throw new IllegalStateException();
        
        if (cursor < 0 && nextIndex < 0)
            detach();
    } finally {
        lock.unlock();              
    }
}

```

* 1.ArrayBlockingQueue,put/take方法,当塞入队列或者从队列中取出,而队列中没有元素的时候,线程会阻塞
* 2.线程的阻塞锁就是ReentrantLock,独占可重入锁
* 3.初始定义一个数组用来做队列,这个队列的长度是不可变的
* 4.LinkedBlockingQueue跟ArrayBlockingQueue的区别就是,每次插入队列的时候都会动态变化队列的size,而ArrayBlockingQueue是不会变化的数组
* 5.LinkedBlockingQueue初始可以设置队列的大小,如果没有设置,那么默认是Integer.MAX_VALUE,如果生产者的速度大于消费者,那么可能队列一直增长,内存消耗的厉害

