---
layout: post
title: JAVA8 HashMap
category: [JAVA]
tags: [JAVA]
---

* 1.HashMap实现了Map的接口,运行Null的key和value
* 2.跟HashTable差不多,但是非线程安全,以及允许Null值
* 3.不保证Map的顺序
* 4.如果这个HashMap需要多次循环遍历的话,那么就不要设置太高的初始容量,或者加载因子太高,太低
* 5.有两个关键元素影响HashMap的性能,一个是initial capacity(初始容量),一个是load factor(加载因子或者叫负载系数)
初始容量是创建的时候hash表中的buckets数目
加载因子是指当容量超过多少的时候进行rehashed扩容
0.75作为加载因子,是在空间跟时间上一个比较让人满意的数值,值太高增加了空间消耗,以及查找的成本
* 6.HashMap是非线程安全的,如果一定要在多线程的环境下使用HashMap,那么可以使用SynchronizedMap或者ConcurrentHashmap
* 7.如果HashMap的迭代器创建之后,HashMap的结构发生改变,讲会立刻触发fail-fast,抛出异常

```JAVA
HashMap<String, Integer> h1 = new HashMap<String, Integer>();
h1.put("a",1);
h1.put("b",2);

for(Map.Entry<String, Integer> e: h1.entrySet()){
    h1.put("c",3);
    System.out.println(e.getKey());
}
```

如上图所示,将会直接抛出异常,而不是等到执行的时候再抛出
好了,上面是JDK中对于HashMap的一些Notes
加载因子为什么选择0.75,因为一个”泊松分布”,主要用于在某个时间段内多时间的发生概率,感兴趣的可以拓展了解(https://en.wikipedia.org/wiki/Poisson_distribution)

加载因子如果太大,hash碰撞的概率变低,但是每个hashmap的容量变大,占用的空间也随之增多.加载因子太小,占用空间变小,但是碰撞的概率也会变大

```JAVA
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
static final int MAXIMUM_CAPACITY = 1 << 30;
static final float DEFAULT_LOAD_FACTOR = 0.75f;
```

初始容量为16,最大容量为2的30次方,默认加载因子,0.75
HashMap由一个个Node组成,Node的结构

```JAVA
final int hash;
final K key;
V value;
Node<K,V> next;
```

从而形成一个单链表

put 方法

```JAVA
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}

```

* 1.先检查,该key,该value是否已经包含在map中,并且检查原map中是否有值,如果没有,那么直接生成一个node给他.因为他是第一个元素嘛.
* 2.如果这个该key以及存在一个node,那么直接将新的node赋值给他,替换掉原来的node
* 3.再检查是否树状Node,如果是,那么需要按照红黑树的原则去增加,因为红黑树为了平衡,需要变色,以及左旋,右旋来保持高度,并满足红黑树的特性,使得时间复杂度始终为O(logn)
* 4.第一次增加的时候,给binCount赋值,并且自增,增加一个节点,如果binCount大于之前定义的TREEIFY_THRESHOLD(默认为8) 那么需要转换成红黑树来提高效率

hashMap内存结构图

![结构图](http://pic.woowen.com/hashMap%E5%86%85%E5%AD%98%E7%BB%93%E6%9E%84%E5%9B%BE.png)

HashMap是通过数组+链表的形式,当每个链表长度超过8的时候会转化成红黑树的形式,保证性能

#### fail-fast

说下什么是fail-fast,fail-fast是在多线程环境下对非线程安全的集合进行操作时触发的一种机制,直接抛出异常提醒
结构中定义了一个int 类型的字段名叫modCount,每次对集合类的结构进行更改时(增删),该值都会自增,在集合调用Iterator准备对集合进行遍历的时候,会先记录一份modCount的值,如果在遍历的过程中modCount的值发生了变化(被其他线程修改了结构)那么就会抛出异常提示不能修改,需要注意的是并不是一定会抛出异常,这只是一种检测机制


#### 哈希算法

```JAVA

// 通过hashCode右偏移16位,让32位的int值分成高16位和低16位,然后进行异或,生成的hash值能够均匀分布
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

```

且针对hash值使用了计算h & (length - 1) 当length是2的n次方值的时候,等价于h % n,正好可以获取到hashCode对应的散列值


#### 哈希冲突解决

* 开放寻址(线性探查): 取key的线性函数值作为新的hashCode,比如当h发生了冲突,那么就使用h+1,h+2,h+3一直找到不冲突为止

* 拉链: 冲突的值用链表的形式添加在已存在的值后面,这有所有冲突的key元素就通过一个链表连接了起来,然后遍历链表去获取

* 随机数: 取key的随机数作为新的散列地址,通常用于长度不同的key值

* 再哈希: 通过多个不同的哈希算法来完成,当哈希算法1发生冲突,再通过哈希算法2来尝试,哈希算法3尝试,以此类推

#### 关于HashMap每次扩容都是2^n

HashMap默认容量为16,加载因子为0.75,因此每次元素到达16 * 0.75 = 12的时候都需要扩容
扩容的基本操作是先将容量扩容一倍,然后将原先的元素再复制进去
java中通过对容量的左偏移1位,oldCapacity << 1 得到newCapacity

所以新得到的容量也是2^n,这样做的好处是

* 1.通过hashCode % length(等价于 hashCode & (length - 1))操作的时候性能提升,因为 & 性能在CPU级别是好于 % 的
* 2.容器通过扩容之后需要重新散列,保证元素分布平均,容量是2^n的话,那么每次扩容之后二进制就多了一位用于计算

例如16的二进制为 10000,32的二进制为100000,那么length - 1之后二进制分别为1111和11111

那么如果一部分的元素的散列值冲突的话,扩容之后最高位不是0就是1,如果为0表示原先的散列值不变,如果为1,那么新的散列值就是原散列值+新增扩容容量,java8通过这样扩容的方法将元素均匀的分布到新的容器中


参考

<http://tech.meituan.com/java-hashmap.html><br>
