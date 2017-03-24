---
layout: post
title: Java性能监控工具-jstat
category: [JAVA]
tags: [JAVA]
---

### Jstat

##### 用于监视jvm的各种运行状态信息命令工具,可以用于显示本地或者远程的虚拟机中的类装载,内存,gc,JIT编译等信息

通过```jps``` 获取正在进行的线程

![](http://pic.woowen.com/jstatjps.png)

#### 参数

class 显示加载class数量以及所占用的空间等信息

compiler 显示vm实时编译的信息

gc 显示gc相关信息

gccapacity 显示gc空间相关信息,各区的大小和使用大小

gccause 显示最近一次gc统计和原因

gcnew 新生代区域统计

gcnewcapacity 新生代区域容量

gcold 老年代统计

gcoldcapacity 老年代区域容量

gcpermcapacity 这个参数应该1.7之前用的永久代吧.1.8没有永久代了

gcmetacapacity 元空间区域容量

gcutil gc统计汇总

printcompilation 虚拟机静态方法

#### class

命令: ```jstat -class 19597``

![](http://pic.woowen.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-03-24%20%E4%B8%8B%E5%8D%886.22.32.png)

* Loaded	装载类梳理
* Bytes	装载类占用字节数
* Unloaded	卸载类数量
* Bytes	卸载类字节数
* Time  装载类和卸载类花费时间

#### compiler

命令: ```jstat -compiler 19597```

![](http://pic.woowen.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-03-24%20%E4%B8%8B%E5%8D%886.26.36.png)

* Compiled	编译执行数量
* Failed  编译失败数量
* Invalid  编译失效数量
* Time  编译耗时
* FailedType  最后一个编译失败类型
* FailedMethod  最后一个编译失败类和方法

#### gc

命令: ```jstat -gc 24212```

![](http://pic.woowen.com/jstatgc.png)

* S0C 新生代第一个survivor区域容量
* S1C 新生代第二个survivor区域容量   
* S0U s0中目前已经使用的容量   
* S1U s1中目前已经使用的容量     
* EC  新生代Eden区域容量     
* EU  新生代Eden区域已使用容量      
* OC  老年代区域       
* OU  老年代已使用区域     
* MC  元空间容量   
* MU  元空间已使用容量  
* CCSC  压缩类空间容量 
* CCSU  压缩类空间已使用容量 
* YGC young GC次数    
* YGCT young GC花费时间
* FGC  Full GC次数  
* FGCT  Full GC花费时间   
* GCT GC总共花费的时间

#### gccapacity

命令: ```jstat -gccapacity 24212```

![](http://pic.woowen.com/jstatgccapacity.png)

* NGCMN  新生代初最小容量
* NGCMX  新生代最大容量   
* NGC    新生代当前容量 
* S0C    S0容量
* S1C 	 S1容量
* EC     Eden区域容量 
* OGCMN	 老年代最小容量	      
* OGCMX  老年代最大容量     
* OGC    老年代当前容量     
* OC     老年代容量  
* MCMN   元空间最小容量  
* MCMX   元空间最大容量   
* MC     元空间当前容量
* CCSMN  压缩类空间最小容量  
* CCSMX  压缩类空间最大容量   
* CCSC   压缩类空间当前容量 
* YGC    新生代GC次数
* FGC    Full GC次数

#### gccause

命令: ```jstat -gccause 24212```

![](http://pic.woowen.com/gccause.png)

#### gcnew

命令: ```jstat -gcnew 24212```

![](http://pic.woowen.com/gcnew.png)

#### gcnewcapacity

命令: ```jstat -gcnewcapacity 24212```

![](http://pic.woowen.com/gcnewcapacity.png)

#### gcold

命令: ```jstat -gcold 24212```

![](http://pic.woowen.com/gcold.png)

#### gcoldcapacity

命令: ```jstat -gcoldcapacity 24212```

![](http://pic.woowen.com/gcoldcapacity.png)

#### gcpermcapacity java1.7以及之前有效,1.8无效

#### gcmetacapacity

命令: ```jstat -gcmetacapacity 24212```

![](http://pic.woowen.com/metacapacity.png)


#### Example

```jstat -gcold 24212 2000 10```

每2秒执行一次命令,执行10次

```jstat -gcnew -h3 21891 250```

每250ms执行一次命令,每输出3个分割一次

```jstat -gcoldcapacity -t 24212 250 3```

加上时间




