---
layout: post
title: JAVA Synchronized
category: [JAVA]
tags: [JAVA]
---

JAVA 中的对象锁主要依靠对象头和锁监视器(Monitor)实现

### 对象头

对象头也就是每个对象的头部,其中记录了一些信息

对象头中的信息会根据对象的状态和平台做出相应的改变

![](http://pic.woowen.com/objecthead.png)

其中记录的一些信息

* hash: 对象hashcode
* age: 对象分代年龄
* biased_lock: 是否偏向锁
* lock: 锁标志
* JavaThread: 如果是偏向锁,那么记录线程ID

```C++

//    [JavaThread* | epoch | age | 1 | 01]     给定线程ID的偏向锁
//    [0           | epoch | age | 1 | 01]     匿名偏向锁

//	  锁标志
//    [ptr             | 00]  locked             轻量级锁
//    [header      | 0 | 01]  unlocked           无锁
//    [ptr             | 10]  monitor            基于monitor实现的重量级锁
//    [ptr             | 11]  marked             GC标记

```

### 偏向锁

偏向锁的存在是为了在单线程情况下没有竞争出现的条件下使用锁,而尽量减少CAS的使用,避免本地延迟

线程持有过偏向锁,那么这个锁就是有偏向的,当线程再次获取锁的时候可以不用执行CAS,通过对象头中的线程ID就能判断是当前线程,则直接获取到锁

Synchronized字节码编译之后分别是两个命令```monitorenter```,```monitorexit```

```C++

// InterpreterRuntime.cpp : 561
IRT_ENTRY_NO_ASYNC(void, InterpreterRuntime::monitorenter(JavaThread* thread, BasicObjectLock* elem))
#ifdef ASSERT
  thread->last_frame().interpreter_frame_verify_monitor(elem);
#endif
  if (PrintBiasedLockingStatistics) {
    Atomic::inc(BiasedLocking::slow_path_entry_count_addr());
  }
  Handle h_obj(thread, elem->obj());
  assert(Universe::heap()->is_in_reserved_or_null(h_obj()),
         "must be NULL or an object");
  if (UseBiasedLocking) {
    // 通过系统命令可以选择开启或者关闭偏向锁,默认开启
    // 进入fast_enter流程
    ObjectSynchronizer::fast_enter(h_obj, elem->lock(), true, CHECK);
  } else {
    ObjectSynchronizer::slow_enter(h_obj, elem->lock(), CHECK);
  }
  assert(Universe::heap()->is_in_reserved_or_null(elem->obj()),
         "must be NULL or an object");
#ifdef ASSERT
  thread->last_frame().interpreter_frame_verify_monitor(elem);
#endif
IRT_END

```

##### 获取偏向锁

* 1.判断对象是否为偏向状态,markword中,偏向标志(biased_lock)为1,锁标志(lock)为01
* 2.判断是否有线程持有该对象,markword中JavaThread是否有值,如果为空,则进入下一步,如果指向当前线程,则执行同步代码块,如果指向其他线程则进入步骤(4)
* 3.markword中JavaThread为空则通过cas设置为当前线程的ID,如果成功则获得偏向锁并执行代码块,如果失败进入步骤(4)
* 4.cas失败或者JavaThread中指向了其他进程,则表示有其他进程在竞争,当达到全局安全点时,获得偏向锁的进程被挂起,撤销偏向锁,升级轻量锁,继续之前的线程

##### 释放偏向锁
线程是不会主动释放偏向锁的,只要有其他线程来竞争,那么就会释放偏向锁,且偏向锁释放需要等到全局安全点时才会进行

* 1.等到savepoint
* 2.暂停拥有偏向锁的线程,判断对象是否处于偏向锁状态
* 3.撤销偏向锁,恢复到无锁或者轻量级锁状态(01,00)

### 轻量级锁

轻量级锁是用CAS尽量代替mutex的作用,适用2个线程互相获取锁的情况下,不能用于多线程竞争,也不是用来代替互斥锁的,只是为了减少互斥锁的目的,提高性能

```C++

// 获取轻量级锁
void ObjectSynchronizer::slow_enter(Handle obj, BasicLock* lock, TRAPS) {
  markOop mark = obj->mark();
  assert(!mark->has_bias_pattern(), "should not see bias pattern here");

  // 判断对象是否处于无锁状态,偏向锁标志(biased_lock)为0,锁标志(lock)标志位为01
  if (mark->is_neutral()) {       
  	// 把mark复制到_displaced_header字段,字段保存在线程的栈帧上,线程私有
  	// 将栈帧中的字段_displaced_header设置为markword
    lock->set_displaced_header(mark);
    // 通过cas将markword指向栈帧
    if (mark == (markOop) Atomic::cmpxchg_ptr(lock, obj()->mark_addr(), mark)) {    	
      TEVENT (slow_enter: release stacklock) ;
      return ;
    }    
  } else
  if (mark->has_locker() && THREAD->is_lock_owned((address)mark->locker())) {  	
    assert(lock != mark->locker(), "must not re-lock the same lock");
    assert(lock != (BasicLock*)obj->mark(), "don't relock with same BasicLock");
    lock->set_displaced_header(NULL);
    return;
  }
  // 重置_displaced_header字段
  lock->set_displaced_header(markOopDesc::unused_mark());
  // 锁膨胀为重量级锁
  ObjectSynchronizer::inflate(THREAD, obj())->enter(THREAD);
}

// 释放轻量级锁
void ObjectSynchronizer::fast_exit(oop object, BasicLock* lock, TRAPS) {
  assert(!object->mark()->has_bias_pattern(), "should not see bias pattern here");  
  markOop dhw = lock->displaced_header();
  markOop mark ;
  if (dhw == NULL) {         
     mark = object->mark() ;
     assert (!mark->is_neutral(), "invariant") ;
     if (mark->has_locker() && mark != markOopDesc::INFLATING()) {
        assert(THREAD->is_lock_owned((address)mark->locker()), "invariant") ;
     }
     if (mark->has_monitor()) {
        ObjectMonitor * m = mark->monitor() ;
        assert(((oop)(m->object()))->mark() == mark, "invariant") ;
        assert(m->is_entered(THREAD), "invariant") ;
     }
     return ;
  }

  mark = object->mark() ;

  if (mark == (markOop) lock) {
     assert (dhw->is_neutral(), "invariant") ;
     if ((markOop) Atomic::cmpxchg_ptr (dhw, object->mark_addr(), mark) == mark) {
        TEVENT (fast_exit: release stacklock) ;
        return;
     }
  }

  ObjectSynchronizer::inflate(THREAD, object)->exit (true, THREAD) ;
}


```

##### 获取轻量级锁

* 1.```mark->is_neutral()```判断是否无锁
* 2.把mark保存到_displaced_header字段
* 3.通过cas尝试将mark指向栈帧中的lock(BasicLock)类型的对象,如果执行成功,表示获得了锁,执行同步代码块
* 4.否则升级为重量级锁

##### 释放轻量级锁

* 1.将栈帧中的_displaced_header替换到mark中,如果成功,释放锁,否则升级锁


### 重量级锁

重量级锁就是指monitor,每个对象和类都有属于自己的monitor

结构

```C++

protected:                              // Monitor-Mutex metadata
  SplitWord _LockWord ;                  // Contention queue (cxq) colocated with Lock-byte
  enum LockWordBits { _LBIT=1 } ;
  Thread * volatile _owner;              // The owner of the lock
                                         // Consider sequestering _owner on its own $line
                                         // to aid future synchronization mechanisms.
  ParkEvent * volatile _EntryList ;      // List of threads waiting for entry
  ParkEvent * volatile _OnDeck ;         // heir-presumptive
  volatile intptr_t _WaitLock [1] ;      // Protects _WaitSet
  ParkEvent * volatile  _WaitSet ;       // LL of ParkEvents
  volatile bool     _snuck;              // Used for sneaky locking (evil).
  int NotifyCount ;                      // diagnostic assist
  char _name[MONITOR_NAME_LEN];  

```

![](http://pic.woowen.com/monitor.jpeg)

* Contention List 竞争队列,所有请求锁的线程会放入这个队列中
* Entry List 竞争队列中,有资格称为候选人的线程被放入这个队列中
* Wait Set 当前锁拥有线程调用wait()等待方法,就会放入等待队列
* OnDeck 任意时刻只有一个线程可以竞争锁
* Owner 监视器拥有者(锁的拥有者)

流程

![](http://pic.woowen.com/monitor.png)

Contention queue --> EntryList --> OnDeck --> Owner --> !Owner

竞争线程push到竞争队列的时候,会先进行自旋获取锁,因为如果当前锁得到锁之后立马释放了,那么竞争队列中的任意线程都有机会通过一定的自旋来获取到锁,而不用进入阻塞状态,提高了性能

调用wait()方法会让当前线程进入wait set,调用notify(),notifyAll(),将线程从wait set移动到竞争队列或者Entry List中

owner每次释放的时候会deck中的线程会去获取锁,如果获取成功,那么从EntryList的头部wakeone,并传入deck

每次释放也会从Contention queue中获取一部分传递到EntryList


##### 参考
<http://www.jianshu.com/p/c5058b6fe8e5><br>
<http://www.jianshu.com/p/759329da16e2>










