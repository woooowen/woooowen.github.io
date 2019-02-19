---
layout: post
title: ios8 xcode6 设置Launch Image 启动图片
category: [Swift]
tags: [Swift,Xcode,ios]
---
###如何设置app的启动图,也就是Launch Image?

####Step1
* 1.点击Image.xcassets 进入图片管理,然后右击,弹出"New Launch Image"
* 2.如图,右侧的勾选可以让你选择是否要对ipad,横屏,竖屏,以及低版本的ios系统做支持.这边我选了ios8.0,ios7.0,ios6没有做支持.

![LaunchImage](http://pic.woowen.com/2014-12-12 11-38-19.png)

####Step2

将规定尺寸的图片从你的文件中拖动进到固定位置.

<table>
<tbody>
	<tr>
		<td><em>系统</em></td>
		<td><em>尺寸</em></td>
		<td><em>分辨率</em></td>
	</tr>
	<tr><td>ios8</td><td>Retina HD5.5</td><td>1242x2208</td></tr>
	<tr><td> </td><td>Retina HD4.7</td><td>750x1334</td></tr>
	<tr><td> </td><td>Landscape Retina Hd 5.5</td><td>2208x1242</td></tr>
	<tr><td>ios7</td><td>	</td><td>640x960</td></tr>
	<tr><td></td><td>Retina4</td><td>640x1136</td></tr>
</tbody>
</table>

上传完毕,那么基本就快好了.

####Step3

单击你整个项目名称,然后选择General,就是这个.

![image](http://pic.woowen.com/2014-12-12 11-44-01.png)

###重点来了.

我完成上面的步骤,且设置了Launch Images Srouce 为LaunchImage,但是启动图片还是不变,后来发现Launch SrceenFile,这个里面设置了,进去看下,你的目录下有个文件叫做LaunchScreen.xib
打开右侧框,选择这个文件,然后在如图,把Use as launch Srceen取消掉,这个就是你之前一直设置Launch Image不成功的原因

![image](http://pic.woowen.com/2014-12-12 11-49-31.png)

####Step4
###Run
Launch Image已经更改
顺便发一张我得Launch Image
![iamge](http://pic.woowen.com/2014-12-12 11-54-55.png)
####如果你觉得你开启太快,那么漂亮得LaunchImage还没怎么展示就跳过了.你可以在你的第一个加载页面中添加如下代码来延长LaunchImage的显示时间.

```js
	//Swift code
	//这个是swift得版本的.额,你千万不要自己新增一个方法viewDidLoad哦,你里面有的
    override func viewDidLoad() {
        super.viewDidLoad()
        NSThread.sleepForTimeInterval(3.0)//延长3秒
    }
```

