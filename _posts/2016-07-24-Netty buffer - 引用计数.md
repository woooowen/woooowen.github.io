---
layout: post
title: Netty buffer - 引用计数
category: [源码]
tags: [Netty]
---

#### 类继承关系图

![](http://pic.woowen.com/netty%20class.png)

几个实现类中都继承了这个`AbstractReferenceCountedByteBuf`类,它的主要功能是用来维护引用计数

```JAVA

// 通过一个volatile变量来指定引用计数
private volatile int refCnt = 1;

// 增加计数, refCnt + 1 ,循环去cas改变计数
private ByteBuf retain0(int increment) {
    for (;;) {
        int refCnt = this.refCnt;
        final int nextCnt = refCnt + increment;

        // Ensure we not resurrect (which means the refCnt was 0) and also that we encountered an overflow.
        if (nextCnt <= increment) {
            throw new IllegalReferenceCountException(refCnt, increment);
        }
        if (refCntUpdater.compareAndSet(this, refCnt, nextCnt)) {
            break;
        }
    }
    return this;
}

// 释放计数 refCnt - 1
private boolean release0(int decrement) {
    for (;;) {
        int refCnt = this.refCnt;
        if (refCnt < decrement) {
            throw new IllegalReferenceCountException(refCnt, -decrement);
        }

        if (refCntUpdater.compareAndSet(this, refCnt, refCnt - decrement)) {
            if (refCnt == decrement) {
                deallocate();
                return true;
            }
            return false;
        }
    }
}


```

