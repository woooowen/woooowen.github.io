---
layout: post
title: xcode6 设置textField无法呼出键盘,如何关闭键盘
category: [swift]
tags: [xcode,swift]
---

###在xcode6中使用了textField,但是在运行的时候发现即使获取焦点了.也没有自动呼出键盘
解决方法:

* 打开`IOS Simulator` -> `Hardware` -> `Keyboard` -> `Toggle Software Keyboard`
* 再次运行即可

#####顺便补充一个如何在设置点击任意一处的时候自动关闭键盘的操作在ViewController代码中加上这段代码,其中Unamelabel,pwdLabel代表你绑定的空间自定义名称

```ruby

override func touchesEnded(touches: NSSet, withEvent event: UIEvent) {
        uNameLabel.resignFirstResponder()
        pwdLabel.resignFirstResponder()        
        }    

```

>
xcode6 坑好多,慎用!