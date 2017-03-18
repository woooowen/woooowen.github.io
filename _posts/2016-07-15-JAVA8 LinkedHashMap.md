---
layout: post
title: JAVA8 LinkedHashMap
category: [JAVA]
tags: [JAVA]
---

* 1.继承自HashMap
* 2.按照插入顺序排序
* 3.双向链表

LinkedHashMap大部分的特性都跟HashMap一样,但是重写了下么的3个方法从而导致其有序性

```JAVA

void afterNodeRemoval(Node<K,V> e) { // unlink
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.before = p.after = null;
        // 如果节点没有前一个节点,那么他就是头节点
        if (b == null)
            head = a;
        else
            b.after = a;
        // 如果节点的after为null,表示他没有后一个节点,那么尾节点就是他
        if (a == null)
            tail = b;
        else
            a.before = b;
    }

    // put操作会调用这个方法
    void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMap.Entry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
    }

    // 当这个节点被访问之后,replace,get,等遍历都会访问到这个方法
    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }

```

双向链表

```JAVA
 transient LinkedHashMap.Entry<K,V> head;

 transient LinkedHashMap.Entry<K,V> tail;

```  

####所以LinkedHashMap一个双向链表,定义了一个头和一个尾,每次操作之后调用重写的3个方法来维护另一个有有序的链表,而另一个链表就是原来的HashMap,那个只用来记录数据,单独维护有序链表用来保证整个LinkedHashMap的有序性
