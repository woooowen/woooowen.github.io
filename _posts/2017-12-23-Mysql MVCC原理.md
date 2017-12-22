---
layout: post
title: Mysql MVCC原理
category: [Mysql]
tags: [Mysql]
---

MVCC 多版本并发控制技术,用于多事务环境下,对数据读写在不加读写锁的情况下实现互不干扰,从而实现数据库的隔离性,在事务隔离级别为Read Commit 和 Repeatable read中使用到

这篇文章主要用来描述mysql mvcc机制的运行原理

### 聚簇索引结构

InnoDB表数据为主键聚簇索引,mysql默认为每个索引行添加了4个隐藏的字段,分别是

* ```DB_ROW_ID``` InnoDB引擎中一个表只能有一个主键,用于聚簇索引,如果表没有定义主键会选择第一个非Null的索引作为主键,如果还没有,生成一个隐藏的DB_ROW_ID作为主键构造聚簇索引
* ```DB_TRX_ID``` 最近更改该行数据的事务ID
* ```DB_ROLL_PTR``` undo log的指针,用于记录之前历史数据在undolog中的位置
* ```DELETED BIT``` 索引删除标志,如果DB删除了一条数据,是优先通知索引将该标志位设置为1,然后通过(purge)清除线程去异步删除真实的数据

![](http://pic.woowen.com/mvcc3.png)

如图所示,undo log中记录之前修改该行数据的事务ID以及被修改的历史数据

整个MVCC的机制都是通过```DB_TRX_ID```,```DB_ROLL_PTR```这2个隐藏字段来实现的

##### 执行过程

* 用排他锁锁定该行
* 记录redo log
* 记录undo log
* 修改当前行的值,修改行的事务ID为当前事务ID
* 回滚指针指向undo log刚刚记录的位置

### 事务链表

当一个事务开始的时候,会将当前数据库中正在活跃的所有事务(执行begin,但是还没有commit的事务)保存到一个叫```trx_sys```的事务链表中,事务链表中保存的都是未提交的事务,当事务提交之后会从其中删除

![](http://pic.woowen.com/trx_sys.png)

### ReadView

在事务开始的时候会根据上面的事务链表构造一个ReadView,初始化方法如下

### read view

```C

// readview 初始化
// m_low_limit_id = trx_sys->max_trx_id; 
// m_up_limit_id = !m_ids.empty() ? m_ids.front() : m_low_limit_id;
ReadView::ReadView()
	:
	m_low_limit_id(),
	m_up_limit_id(),
	m_creator_trx_id(),
	m_ids(),
	m_low_limit_no()
{
	ut_d(::memset(&m_view_list, 0x0, sizeof(m_view_list)));
}


```


* 1.活跃事务链表(```trx_sys```)中事务id最大的值被赋值给```m_low_limit_id```
* 2.活跃事务链表中第一个值(也就是事务id最小)被赋值给```m_up_limit_id```
* 3.```m_ids``` 为事务链表

通过这时ReadView,当前事务可以根据查询到的所有记录中记录的事务ID进行匹配是否能看见该记录,从而实现数据库的事务隔离
我们来梳理下逻辑

* 聚簇索引中有隐藏列记录了当前数据最近被哪个事务ID修改过
* 一个新的事务开始时会根据事务链表构造一个ReadView
* 当前事务根据ReadView中的数据去跟检索到的每一条数据去校验,看看当前事务是不是能看到这条数据

#### 可见性判断

```C

// 判断数据对应的聚簇索引中的事务id在这个readview中是否可见
bool changes_visible(
		trx_id_t		id, // 记录的id
		const table_name_t&	name) const
		MY_ATTRIBUTE((warn_unused_result))
	{
		ut_ad(id > 0);
		// 如果当前记录id < 事务链表的最小值或者等于创建该readview的id就是它自己,那么是可见的
		if (id < m_up_limit_id || id == m_creator_trx_id) {
			return(true);
		}

		check_trx_id_sanity(id, name);
		// 如果该记录的事务id大于事务链表中的最大值,那么不可见
		if (id >= m_low_limit_id) {

			return(false);
		// 如果事务链表是空的,那也是可见的
		} else if (m_ids.empty()) {

			return(true);
		}

		const ids_t::value_type*	p = m_ids.data();

		return(!std::binary_search(p, p + m_ids.size(), id));
	}

```

![](http://pic.woowen.com/trx_sys.png)

还是这张图,这边偷了下懒

* 当检索到的数据的事务ID(数据事务ID < ```m_up_limit_id```) 小于事务链表中的最小值表示这个数据在当前事务开启前就已经被其他事务修改过了,所以是可见的
* 当检索到的数据的事务ID(数据事务ID = ```m_creator_trx_id```) 表示是当前事务自己修改的数据
* 当检索到的数据的事务ID(数据事务ID >= ```m_low_limit_id```) 大于事务链表中的最大值表示这个数据在当前事务开启之前又被其他的事务修改过,那么就是不可见的
* 如果事务链表为空,那么也是可见的,也就是当前事务开始的时候,没有其他任意一个事务在执行

#### RC事务隔离和RR事务隔离下ReadView的区别

* 在RC事务隔离级别下,每次语句执行都关闭ReadView,然后重新创建一份ReadView
* 在RR下,事务开始创建ReadView,一直到事务结束关闭

##### 在上面的事务列表中,当处于RC隔离级别下时,事务1,事务2,事务3的修改,当前事务都是可见的
##### 当处于RR隔离级别下,事务1,事务2,事务3的修改,当前事务都不可见

#### Undo log的删除

当该undo log关联的事务没有出现在其他事务的readview中时(事务已提交,且没有其他事务依赖当前事务),那么InnoDB引擎的后台清除线程(purge线程)会进行遍历删除undo log操作,聚簇索引中凡是DELETED BIT被设置为1的数据库记录也会被清除线程删除,这些操作都是异步执行的


##### 参考

<http://www.ywnds.com/?p=10418>

<http://www.imooc.com/article/17290>

<http://blog.csdn.net/woqutechteam/article/details/68486652>

<http://blog.csdn.net/chen77716/article/details/6742128>

<http://hedengcheng.com/?p=148>







