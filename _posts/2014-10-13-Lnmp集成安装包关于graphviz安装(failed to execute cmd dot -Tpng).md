---
layout: post
title: lnmp集成安装包关于graphviz安装(failed to execute cmd dot -Tpng)
category: [开发工具]
tags: [Linux]
---

最近想试试xhprof来调试php,具体就不说什么安装了.这个网上的资料比较多.

关键是说下图形工具graphviz的安装.

###给出的错误```failed to execute cmd " dot -Tpng"```

谷歌的半天,看了半天不给力,资料实在太少了.找不到答案,

于是看代码.最后发现在```XHprof callgraph_utils.php``` 文件中的xhprof_generate_image_by_dot方法调用了proc_open()这个方法.一般是用来执行linux命令的.类似exec(),system()

发现lnmp一键安装里面的```php.ini``` 里面的 disable_functions 里面给禁止了.于是你只要把他给删除了.就能正常运行了.我说呢.为什么别人那么安逸的就能搞定的东西.
