---
layout: post
title: Tomcat Service 中的执行流程
category: [源码]
tags: [Tomcat]
---

Tomcat的Service主要包含2个部分,分别是Connector(连接器),Container(容器),Mapper(映射器)

先对Service有个大概的了解

Service是用来控制Connector,Container,Mapper的

```JAVA

// org/apache/catalina/core/StandardService.java
// connectors数组,用于管理service的所有连接器
protected Connector connectors[] = new Connector[0];

// 管理所有线程池,线程池的作用在于处理accepter监听到的请求
protected final ArrayList<Executor> executors = new ArrayList<>();

// Engine是Container最顶层的容器
private Engine engine = null;

// Mapper中记录了所有servlet的映射关系
protected final Mapper mapper = new Mapper();

```

### Connector 连接器

个人觉得连接器主要分成这几个部分
* 1.Acceptor,负责监听http或者ajp请求,最后发送给线程池去执行
* 2.ProtocolHandler,解析http协议,例如http协议的请求行,请求头,并进行check,然后转发
* 3.CoyoteAdapter接收ProtocolHandler的处理结果,并创建request和response对象,在根据映射关系给request设置mappingData属性,然后转发给Container(容器)

#### Acceptor

![](http://pic.woowen.com/connector.png)

#### ProtocolHandler,CoyoteAdapter

![](http://pic.woowen.com/protocolhandler.png)

### Container 容器

容器的层次结构是Engine->Host->Context->Wrapper

Engine代表是servlet引擎,管理所有的容器类,负责其所有子类的生命周期
Host代表引擎中的虚拟主机,一台机器可以部署多个虚拟主机,代表不同的域名指向
Context代表一个web项目
Wrapper代表一个servlet实例

容器之间通过管道和阀门来控制传递,一次请求会依次通过所有的阀门,明显的责任链模式

每个容器都会有一些自带的阀门,一一对应,并且是管道中的最后一个阀门,用来向下级传递请求

Engine 对应 StandardEngineValve
Host 对应 StandardHostValve
Context 对应 StandardContextValve
Wrapper 对应 StandardWrapperValve

![](http://pic.woowen.com/container.png)

* Http11Processor 负责解析Http,根据Http协议的特点去依次解析请求行,请求头,然后调用CoyoteAdapter
* CoyoteAdapter 负责创建Request,Response对象,并且根据Mapper映射关系,给Request赋值mappingData属性
* 此间也会进行一系列的工作,比如http协议的校验,http协议是否需要升级,配置serverName等等
* 最后将配置好的Request对象和Response对象传递给Container的管道的第一个Valve(也就是Engine管道中的第一个Valve)
* Engine中的valve执行invoke方法,在通过request获取它的host,然后再传递request给host容器
* host容器首先会有一些默认的Valve需要执行,例如日志Valve,错误报告Valve,然后在执行自己的StandHostValve,在获取request的context在往下传递
* 一直传递到Wrapper为止,Wrapper是最底层的容器,不能再有子容器了.
* Wrapper容器会去获取servlet实例,如果没有就加载servlet class,然后初始化
* 再获取这个servlet的过滤链(Filter Chain) 因为过滤器是包裹着servlet的
* 如果过滤链中有servlet执行前需要执行的,那么执行,然后执行servlet的service()方法
* 最后得到结果填充response,在将response一层层的往上传递
* response一直传递到Connector,然后返回给请求的客户端,到此整个过程结束


##### HTTP解析

![](http://pic.woowen.com/httpheader.png)

tomcat中Http的解析分为请求行,请求头分开解析
将字符转成字节数组,然后遍历判断是否有空格或者```\t```,然后在按照上图的格式依次去解析```\r\n```以及```:```
http请求头部,最多8MB

说是源码解析,但是是在不想贴代码,因为实在太长了,如果要看源码解析还不如自己去看tomcat的代码,既然写出来,主要是梳理下逻辑.帮助快速理解,那么画图就是最好的方式

我觉得这种方式比较好,看其他的一些博客做源码分析,全是大段代码,那样我还不如自己去看代码分析呢.所以写这篇文章就是大概介绍下tomcat的整体结构和流程,帮助理解

