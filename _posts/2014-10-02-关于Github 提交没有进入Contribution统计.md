---
layout: post
title: 关于Github提交没有进入Contribution统计的问题
category: [Github]
tags: [Github]
---
本地一直使用github的mac客户端去提交代码,发现提交了好多次的代码,但是依然没有进入github的Contribution统计,于是参考了相关资料找到原因.

####会被记录进入Contribution的情形

	1.操作在1年之内
	2.操作针对的是一个独立的仓库,在Fork仓库中提交代码不会计入
	3.进行commit的时候关联了你的Github账户
	
####除了以上情况之外也会被记录进入Contribution的情形

	1.你是这个项目组织的一员
	2.你对这个项目有过pull,issue
	3.你标记过了这个项目
	4.你fork过这个仓库
	
我得问题是我本地的客户端没有关联我得Github账户,mac客户端

* `Preferences`->`Advanced`->`Git` 
* Config中的用户名跟邮箱地址要跟你的账号一样

这样才能关联.好了等于之前的提交都白干了.哈哈.