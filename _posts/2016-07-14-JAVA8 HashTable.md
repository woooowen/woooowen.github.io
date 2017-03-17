---
layout: post
title: JAVA8 HashTable
category: [JAVA]
tags: [JAVA]
---

* 1.不允许Null值的key和value,直接抛出NPE
* 2.HashTable 同样有2个参数会影响他的性能,inital capacity,load factor
* 3.继承自Dictionary类,实现了Map接口
* 4.线程安全,JDK中大量使用了synchronied同步块来保证线程安全

结构单链表数组
![](http://pic.woowen.com/hashtable1.jpg)


add方法

```JAVA

public synchronized V put(K key, V value) {
        // 校验val不为null
        if (value == null) {
            throw new NullPointerException();
        }

        // 确保key不存在
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        Entry<K,V> entry = (Entry<K,V>)tab[index];
        for(; entry != null ; entry = entry.next) {
            if ((entry.hash == hash) && entry.key.equals(key)) {
            	// 如果键值以及存在,那么直接替换掉老的val,并且返回老的val
                V old = entry.value;
                entry.value = value;
                return old;
            }
        }

        addEntry(hash, key, value, index);
        return null;
    }
```    

remove方法

```JAVA
public synchronized V remove(Object key) {
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        // hash规则
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        Entry<K,V> e = (Entry<K,V>)tab[index];
        for(Entry<K,V> prev = null ; e != null ; prev = e, e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
            	// 存在该key
                modCount++;
                if (prev != null) {
                	// 找到它的前一个节点,并把它的next指针,指向自己的下一个节点
                    prev.next = e.next;
                } else {
                    // 如果前一个节点为空,表示他是first entry,头节点,那么直接将后一个节点设置为头节点
                    tab[index] = e.next;
                }
                count--;
                V oldValue = e.value;
                // 将当前节点的value设置为null,等待GC
                e.value = null;
                return oldValue;
            }
        }
        return null;
    }

```

get方法

```JAVA
public synchronized V get(Object key) {
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                return (V)e.value;
            }
        }
        return null;
}
```

大量使用了同步代码块,因此他的性能比HashMap肯定差了不少,优点就是线程安全
add,remove,get方法本质都是去遍历数组,时间复杂度O(n),这个方法基本可以放弃了.如果只是因为同步需要用到,可以考虑ConcurrentHashMap了.




