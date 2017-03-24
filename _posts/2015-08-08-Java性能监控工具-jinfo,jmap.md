---
layout: post
title: Java性能监控工具-jinfo,jmap
category: [JAVA]
tags: [JAVA]
---

### Jinfo

一般通过```jps -v```可以看到当前java线程一些显示的参数

![](http://pic.woowen.com/jpsv.png)

而命令```jinfo -flags 24212```可以看到一些隐藏的参数,一般都是通过这个方法获取jvm的一些参数设置

![](http://pic.woowen.com/jinfoflag.png)

### Jmap

```jmap -heap 24212``` 打印heap的一些信息,包括使用的gc收集器,几个线程以及新生代,老年代的容量占用等

![](http://pic.woowen.com/jmapheap.png)

```jmap -histo:live 24212``` 打印heap中的类,对象数目,以及占用容量等,一般客户端显示不了,因此基本都是dump然后使用特定的工具去查看

```jmap -finalizerinfo 26490``` 打印F队列中等待的finalize线程执行finalize方法的对象,只有linux才有这个参数

```jmap -dump:file=Music/2.bin 26490``` 将内存信息打印到文件

![](http://pic.woowen.com/jmapdump.png)

### Jhat

```jhat 2.bin```去查看上面使用```jmap -dump```导出的文件,他会自动去创建一个本地的http server,默认是在7000端口,那么直接访问localhost:7000就可以看到页面了

![](http://pic.woowen.com/jhat1.png)