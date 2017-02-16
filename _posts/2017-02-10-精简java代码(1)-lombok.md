---
layout: post
title: 精简java代码(1)-lombok
category: [JAVA]
tags: [JAVA]
---

主要介绍如何使用lombok精简自己的java代码,lombok主要是增加了一些功能强大又简单易用的注解,以注解的方式节省了很多工作量

github:  <https://github.com/rzwitserloot/lombok>

`@Getter` , `@Setter`

这个注解可以不用在写实体的get和set方法了.虽然IDE天生可以很方便的生成get和set,但是当一个项目代码中80%以上的都是冗余垃圾代码的时候,你应该开始考虑你的工作效率了

```JAVA

	@Getter
    @Setter
    class test1{
        private String a;
        private String b;
        private Integer c;
    }

```

`@Log` , `@Log4j`, `@Log4j2` , `@Slf4j` , `@XSlf4j`, `@JBossLog`, `@CommonsLog`

这个注解可以不用在类中声明定义日志对象,节省了部分代码,下面是每个注解对应的对象声明

```JAVA

	@Log 
	private static final java.util.logging.Logger log = java.util.logging.Logger.getLogger(LogExample.class.getName());

	@Log4j
	private static final org.apache.log4j.Logger log = org.apache.log4j.Logger.getLogger(LogExample.class);

	@Log4j2
	private static final org.apache.logging.log4j.Logger log = org.apache.logging.log4j.LogManager.getLogger(LogExample.class);

	@Slf4j
	private static final org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(LogExample.class);

	@XSlf4j
	private static final org.slf4j.ext.XLogger log = org.slf4j.ext.XLoggerFactory.getXLogger(LogExample.class);

	@JBossLog
	private static final org.jboss.logging.Logger log = org.jboss.logging.Logger.getLogger(LogExample.class);

	@CommonsLog
	private static final org.apache.commons.logging.Log log = org.apache.commons.logging.LogFactory.getLog(LogExample.class);

```

`@Data` 加上这个注解就自动加上了 `@Getter` , `@Setter`, 以及重写toString,hashCode 等方法 

`@NonNull`

```JAVA

	// 这个等于自动加上了入参检查,如果入参为null会抛出NPE
	public String notNull(@NonNull String a){
        return a + "@";
    }

```

`@Accessors`

fluent = true,chain = true
当fluent为true时,那么所有字段的get跟set方法将会不包含get,set前缀

``` t.getLog() -> t.log() ```
``` t.setLog(String) -> t.log(String) ```

当chain为false时,返回值类型为void

还有个prefix,用处不大,当设置了prefix的字段才会有set/get前缀


`@Build` 自动生成一系列的构造方法,平时很少用到

##### 其他还有诸如`Cleanup`,`SneakyThrows` 等常用的,网上的相关博客也很多,这边主要是简单介绍下,感兴趣的可以自己去深入研究下

关于是否要使用lombok,下面这个链接中的讨论,大家自己见仁见智
http://stackoverflow.com/questions/3852091/is-it-safe-to-use-project-lombok

Lombok优点
1. 避免了各种复杂的重复低效代码,可以略去原来代码的很多get/set方法.以及一些非空判断
2. Lombok通过修改ast来修改字节码,并不会影响到性能问题.
3. 可拓展,不过这个拓展的难度还是蛮大的,因为你同时还需要去修改ide插件
4. 使用方便

Lombok缺点
1. 需要IDE插件支持,否则IDE会自动报错.不过目前eclipse跟idea都有插件支持
2. 代码可读性不好

我觉得还是可以试一试的


