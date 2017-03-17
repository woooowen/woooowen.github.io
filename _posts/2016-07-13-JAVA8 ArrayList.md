---
layout: post
title: JAVA8 ArrayList
category: [JAVA]
tags: [JAVA]
---

* 1.继承AbstractList类,实现了List接口
* 2.执行,size,isEmpty,get,set,iterator,listIterator时间复杂度都是O(1)
* 3.执行add时间复杂度O(n)
* 4.如果你要存很多的数据进入ArrayList,比如200个,那么可以通过设置ensureCapacity来提高性能,我的理解是他在动态增加数组的时候会进行动态的扩容操作,这样事先扩容好,减少每次动态扩容带来的性能损耗?
* 5.非线程安全,如果要在多线程环境中使用

```JAVA
List l = Collections.synchronizedList(new ArrayList());
```

扩容,每次扩容基本等于原size * 1.5

```JAVA
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

indexOf方法,本质上也是通过遍历来实现的,时间复杂度为O(n)

```JAVA
public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```

#### add 方法,每次增加元素之前都去将自己的容量+1,可以通过下面的扩容代码看到,每次扩容实际上是复制一个新的数组,因此才有前面提到的预先给定义ensurecapacity,因为如果你要存入200个值,那么他会动态扩容200次,预先设置可以避免这个阶段的性能损失

```JAVA
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}

```
扩容

```JAVA
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

remove方法

```JAVA
public E remove(int index) {
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```

##### 通过上面的add以及remove可以了解到,ArrayList的新增删除方法跟HashMap那种链表结构不一样,他每次都需要通过新增一个数组然后将修改(新增或者删除)之后的结构复制到新的数组中去,因此ArrayList的add和remove方法并不高效,特别是针对非常大的数组的情况下

![](http://pic.woowen.com/arraylistimg.png)
针对上面的情况,我突然想到,如果塞入一个非常大的ArrayList,那么他肯定是一直去扩容,就会照成很多临时的array,必然会很频繁的触发GC,然后我就简单测试了一下,结果验证了我的想法,下图可以看到,该进程一直在执行young gc,以及fullgc,光是用于gc的时间就话费了50多秒

#### 因此避免使用ArrayList存放大量的数据,如果有,那么事先设置ensureCapacity参数,提高性能
