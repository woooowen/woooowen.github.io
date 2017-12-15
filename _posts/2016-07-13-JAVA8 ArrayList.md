---
layout: post
title: JAVA8 ArrayList
category: [JAVA]
tags: [JAVA]
---

* 1.继承AbstractList类,实现了List接口
* 2.执行,size,isEmpty,get,set,iterator,listIterator时间复杂度都是O(1)
* 3.执行add时间复杂度O(1)
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

#### add 方法,每次增加元素之前都去判断当前容器能否容纳新元素的加入,可以通过下面的扩容代码看到,每次扩容实际上是复制一个新的数组,因此才有前面提到的预先给定义ensurecapacity,因为如果你要存入200个值,预先设置可以避免这个阶段的性能损失

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
针对上面的情况,我突然想到,如果塞入一个非常大的ArrayList,那么他肯定是一直去扩容,就会照成很多临时的array,必然会很频繁的触发GC,然后我就简单测试了一下,结果验证了我的想法,下图可以看到,该进程一直在执行young gc,以及fullgc,光是用于gc的时间就花费了50多秒

#### 因此避免使用ArrayList存放大量的数据,如果有,那么事先设置ensureCapacity参数,提高性能

测试代码

```JAVA
    ArrayList<Integer> l1 = new ArrayList<Integer>();
    long startTime = System.currentTimeMillis();
    for (int i = 0; i <= 999999; i++){
        l1.add(i);
    }
    long endTime = System.currentTimeMillis();
    System.out.println(endTime - startTime);


    ArrayList<Integer> l2 = new ArrayList<Integer>();
    l2.ensureCapacity(1000000);
    long startTime1 = System.currentTimeMillis();
    for (int i = 999999; i >= 0; i--){
        l2.add(i);
    }
    long endTime1 = System.currentTimeMillis();
    System.out.println(endTime1 - startTime1);
    // 输出 42   23

    // 当i的值再*10
    // 输出 2115   812

```    

#### 可以看到加了扩容之后性能提高了一倍左右

#### 之前的理解有问题,以为每次加元素都会扩容,也太蠢了.后来看了实现,并不是每次都会扩容

每次add新的元素会先取判断当前容量是否能容纳新的元素,如果能,那么modCount++,直接加入就行了.只有当容量不够,才会用grow方法去扩容

调用add会先去判断容器的容量,而remove方法会判断移除的是不是数组最后一个,如果是,那么把最后一个置为空值,如果是中间的,每次都会构造一个新的容量-1的新容器,然后将数组复制到新容器

排除扩容,add时间复杂度为O(1),remove为O(n)

