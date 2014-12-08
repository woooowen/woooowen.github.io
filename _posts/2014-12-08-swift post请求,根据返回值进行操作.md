---
layout: post
title: swift post 根据返回值进行操作
category: [Swift]
tags: [Swift]
---
####需求

点击按钮时,post一些参数到api,并且根据api返回的结果,对按钮进行一些操作,例如修改按钮名称,按钮颜色等

###HttpController 文件,里面封装了2个方法一个post,一个get,跟昨天得那个相比稍微做了改动

```js

import Foundation

protocol HttpProtocol{
    func didRecieveResult(result: NSDictionary)
}
class HttpController: NSObject{
    
    var delegate: HttpProtocol?
    
    //json get方法
    func get(url: String){
        var nsUrl: NSURL = NSURL(string: url)!
        var request: NSURLRequest = NSURLRequest(URL: nsUrl)
        NSURLConnection.sendAsynchronousRequest(request, queue: NSOperationQueue.mainQueue(), completionHandler: {(response: NSURLResponse!,data: NSData!,error: NSError!)->Void in
            var jsonResult: NSDictionary = NSJSONSerialization.JSONObjectWithData(data, options: NSJSONReadingOptions.MutableContainers, error: nil) as NSDictionary
            self.delegate?.didRecieveResult(jsonResult)            
        })
    }
    
    //json post方法
    func post(url: String ,params: NSDictionary,callback: (NSDictionary) -> Void) {
        var nsUrl: NSURL = NSURL(string: url)!
        var request: NSMutableURLRequest = NSMutableURLRequest(URL: nsUrl)      
        request.HTTPMethod = "POST"
        request.addValue("application/x-www-form-urlencoded", forHTTPHeaderField: "Content-Type")
        var objStr = ""
        for(key, value) in params as NSDictionary{
            if(objStr == ""){
                objStr += "\(key)=\(value)"
            }else{
                objStr += "&\(key)=\(value)"
            }
        }
        let data: NSData = objStr.dataUsingEncoding(NSUTF8StringEncoding, allowLossyConversion: false)!
        request.HTTPBody = data
        NSURLConnection.sendAsynchronousRequest(request, queue: NSOperationQueue.mainQueue(), completionHandler: {(response: NSURLResponse!,data: NSData!,error: NSError!)->Void in
            var jsonResult: NSDictionary = NSJSONSerialization.JSONObjectWithData(data, options: NSJSONReadingOptions.MutableContainers, error: nil) as NSDictionary
            self.delegate?.didRecieveResult(jsonResult)
            callback(jsonResult)
        })
    }
}

```

###ViewController

```js

import UIKit

class detailViewController: UIViewController,UIWebViewDelegate,HttpProtocol {
            
    @IBOutlet weak var checkBtn: UIButton!
            
    var timeLineUrl: String = ""
    
    var eHttp: HttpController = HttpController()           
    
    override func viewDidLoad() {
        super.viewDidLoad()        
    }
    
    override func didReceiveMemoryWarning() {
        super.didReceiveMemoryWarning()
        // Dispose of any resources that can be recreated.
    }        
    
    //json 数据处理
    func didRecieveResult(result: NSDictionary){
    //    self.tmpCode = result["status"]?["code"] as Int
    }

    //看过按钮点击事件
    @IBAction func checkBtnClick(sender: AnyObject) {
        var btnName = checkBtn.titleForState(UIControlState.Normal)
        if(btnName == "看过"){
            //post 请求api,并且给api传递参数,根据返回值来判断动作执行
            let url = ""//这边你放任何一个api接口
            //构造参数
            let params = ["twitterId" : "13sik"]
            eHttp.delegate = self
            //使用post方式将params里面得参数post给url
            eHttp.post(url, params: params,callback: {(data: NSDictionary) -> Void in
            	//data是通过接口返回的值
                var tt = false
                if(data["status"]?["code"] as NSNumber == 1001){
                    tt = true
                }
                //如果接口返回的是1001,那么就修改按钮的名称为已看过,并且修改按钮文字的颜色
                if(tt){
                    self.checkBtn.setTitle("已看过", forState: UIControlState.Normal)
                    self.checkBtn.setTitleColor(UIColor.grayColor(), forState: UIControlState.Normal)
                }
            })
        }
    }
}

```
