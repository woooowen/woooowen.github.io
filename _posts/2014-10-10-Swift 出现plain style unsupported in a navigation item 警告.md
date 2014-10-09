---
layout: post
title: Swift 出现 plain style unsupported in a navigation item 警告
category: [Swift]
tags: [Swift,Warning,Xcode]
---

####xcode 在编译的时候出现 plain style unsupported in a navigation item 的警告问题
####是因为你在storyboard中,将一个button 放在了Navigation Bar中
####类似这样
![step1](http://woowen.qiniudn.com/1.png)

####解决方法:
####选中Bar Button Item 然后右侧工具中 选择Attributes inspector
####如图所示
![step2](http://woowen.qiniudn.com/2.png)
####将Style 修改成 Bordered 
####编译完成,warning消除
