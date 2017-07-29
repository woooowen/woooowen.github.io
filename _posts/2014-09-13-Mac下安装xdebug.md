---
layout: post
title: Mac下安装xdebug
category: [PHP]
tags: [PHP]
---

mac下安装xdebug 总是出错.

整理下避免以后在遇到

* 1.cd ~ && mkdir xdebug

	在当前用户目录下创建一个文件夹用来安放xdebug源文件

* 2.git clone git://github.com/derickr/xdebug.git

	mac下集成了git工具因此可以直接使用该命令,clone一份xdebug到本地.

* 3.cd xdebug

* 4. /Applications/XAMPP/xamppfiles/bin/phpize

* 5.make

* 6.sudo cp modules/* /Applications/XAMPP/xamppfiles/lib/php/extensions/no-debug-non-zts-20121212/

* 7.vim /Applications/XAMPP/etc/php.ini  在最底下加上这么一段代码
{% highlight css %}
[xdebug]
zend_extension=/Applications/XAMPP/xamppfiles/lib/php/extensions/no-debug-non-zts-20121212/xdebug.so
xdebug.remote_enable=on
xdebug.remote_handler=dbgp
xdebug.remote_host=127.0.0.1
xdebug.remote_port=9000
{% endhighlight %}
如果需要xdebug自动运行.加上xdebug.remote_autostart=1

* 8.然后重启xampp的apache服务器

	sudo /Applications/XAMPP/xamppfiles/xampp stop
	sudo /Applications/XAMPP/xamppfiles/xampp start

查看phpinfo.php文件.找到xdebug

###注: 这个只针对64位得处理器.mac下怎么查看处理器呢 打开终端.输入uname -a x86_64 显示这个就是64位得了.