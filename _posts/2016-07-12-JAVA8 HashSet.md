---
layout: post
title: JAVA8 HashSet
category: [JAVA]
tags: [JAVA]
---

* 1.实现了Set接口
* 2.基于HashMap接口
* 3.能允许Null值
* 4.非线程安全
* 5.如果想要在多线程环境中使用,可以使用Collections.synchronizedSet
* 6.不允许重复的值,插入重复的值返回false

```JAVA
Set s = Collections.synchronizedSet(new HashSet());
```

其实就是一个HashMap,初始化,初始容量16,加载因子0.75

```JAVA
public HashSet() {
    map = new HashMap<>();
}

public HashSet(int initialCapacity) {
    map = new HashMap<>(initialCapacity);
}

public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<>(initialCapacity, loadFactor);
}
```

add方法和remove方法

```JAVA

// HashSet实际上就是一个Hashmap
private transient HashMap<E,Object> map;

private static final Object PRESENT = new Object();
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}

```

#### HashSet中,如果插入相同的key,会更新新的val进去,并且将老的val返回,那么就会导致HashSet中的val == null不成立,返回false,所以HashSet其实并不是不允许重复的值,而是他将老的值已经覆盖了.只是逻辑上告诉你false而已

#### add方法中直接调用HashMap的putval方法,如果key相同,hashcode相同,那么将新的val直接替换掉原来的val,而这个新的val其实就是个虚拟的值,空的object对象

#### 如果成功插入,最后会返回null,因为HashMap中的putval就是返回null,反之都是一个非null的值
