---
layout: post
title: JAVA8 LinkedList
category: [JAVA]
tags: [JAVA]
---

LinkedList结构比较简单,一个双向链表

```JAVA

// 定义头节点和尾节点,默认情况下加入和删除都是针对尾节点操作
// 针对头尾节点的操作,时间复杂度为O(1)
transient Node<E> first;
transient Node<E> last;

// remove操作,需要先遍历获得去要remove的节点,因此复杂度O(n)
public boolean remove(Object o) {
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}

// 根据index操作节点,需要先获得节点
// 通过二分查找获得,因此复杂度O(n/2)
// 根据size >> 1判断,index小于容量的一半就从头节点开始遍历,否则从尾节点开始遍历
Node<E> node(int index) {   
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}


```

这么简单的东西也需要记录么? 实际上我觉得blog一部分是方便其他人的,更多的是方便自己,好记性不如烂笔头,勤能补拙


