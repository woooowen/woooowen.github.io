---
layout: post
title: Linux下生成alias
category: [Linux]
tags: [Linux,Alias]
---

####Linux下得alias 可以用来在终端中设置快捷命令来执行一个操作

####目录在/etc/bashrc

如果你需要增加一个命令.

	1.vim /etc/bashrc

	2.alias 命令='cd ~/xxx' 将这段代码写入bashrc文件中. "命令"代表的意思就是你想要给你新增的命令什么样的名字.随意,但是不要和系统命令重叠.

	3.最后不要忘记在加一句,source /etc/bashrc .这样你新增的那个命令才回生效.

	4.你可以直接在终端中输入alias 来查看现在已有的自定义命令.

由于这个是写在/etc/目录下得.所以你自定义的命令别名是永久生效的.当然你也可以写在用户目录下. ~/.bashrc 这样的话只有该用户才能使用这些命令别名