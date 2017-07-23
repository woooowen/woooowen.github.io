---
layout: post
title: Netty Channel
category: [JAVA]
tags: [JAVA,网络]
---

### Channel用来操作I/O,包括,read,write,bind,connect等

#### netty中的所有I/O操作都是异步的,这意味这,你进行的所有I/O都是非阻塞的,所有的请求都能立刻返回,但是不是返回结果,而是返回一个ChannelFuture实例,这个实例会在I/O操作完成时通知你操作是成功还是失败或者被取消

#### 每个channel都有一个属于自己的pipeline,他在创建一个新的channel的时候自动创建

#### Channel类关系

![](http://pic.woowen.com/channelclass.png)

Channel创建的时候主要做了哪些事情

#### 1.申请全局唯一的channelId

```JAVA

// 唯一channelId = 机器id+线程id+自增数+时间+随机数
private DefaultChannelId() {
    this.data = new byte[MACHINE_ID.length + 4 + 4 + 8 + 4];
    byte i = 0;
    System.arraycopy(MACHINE_ID, 0, this.data, i, MACHINE_ID.length);
    int i1 = i + MACHINE_ID.length;
    i1 = this.writeInt(i1, PROCESS_ID);
    i1 = this.writeInt(i1, nextSequence.getAndIncrement());
    i1 = this.writeLong(i1, Long.reverse(System.nanoTime()) ^ System.currentTimeMillis());
    int random = PlatformDependent.threadLocalRandom().nextInt();
    i1 = this.writeInt(i1, random);

    assert i1 == this.data.length;

    this.hashCode = Arrays.hashCode(this.data);
}

```

#### 2.初始化unsafe类

#### 3.新建一个pipeline,所以说创建channel的时候都会直接新建一个他的pipeline

#### 4.设置阻塞类型,这个跟调用的channel类有关,阻塞或者非阻塞

#### 5.实例化一个channelConfig,配置一些默认参数,比如```DEFAULT_CONNECT_TIMEOUT```

#### 6.在初始化过程中,还会新建一个channelHandler,并且放入pipeline中





