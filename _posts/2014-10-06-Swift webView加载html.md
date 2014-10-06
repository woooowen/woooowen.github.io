---
layout: post
title: Swift之webView用法
category: [Swift]
tags: [Swift]

---

Swift webView的用法
webView其实就是相当于手机App充当了浏览器得功能,用app加载了一个HTTP链接,那么他显示的其实就是一个wap页面.但是我觉得手机新闻资讯类得app将来会被web取代.即app端只用远程加载一个链接地址显示内容即可,至于交互上面交给javascript去做就好了.而oc,或者swift或者android只是充当一个容器的作用.跟pc端得客户端大同小异.


```ruby
	var url = NSURL(string: "http://top.mogujie.com/top/share/note?tid=11ts8")
	var request = NSURLRequest(URL: url)
	webView.loadRequest(request)
```

将上面代码放入页面加载执行的动作中就ok了.稍等一会儿就能将内容显示出来,你也可以在加载之前加上一个交互的图片

```ruby
	func webViewDidStartLoad(webView: UIWebView){
        NSLog("webViewDidStartLoad")
    }
``` 

页面加载完成执行方法

```ruby
	func webViewDidFinishLoad(webView: UIWebView){
        NSLog("webViewDidFinishLoad")        
    }
```       
    