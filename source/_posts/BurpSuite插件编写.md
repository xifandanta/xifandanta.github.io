---
title: BurpSuite插件编写
top: false
cover: true
toc: true
mathjax: true
tags:
  - 博客
categories:
  - BurpSuite插件编写
abbrlink: 44963
date: 2021-10-14 12:51:04
password:
summary:
  - Shiro,Fastjson,Jackson,Thinkphp,越权
---

> 这是我去年写的插件和记录，今天翻文件夹的时候找出来了，里面的内容可能有些过时，有些图是新配的，如果写的有些不到位，大佬轻喷。

### 背景

1.像*Fastjson*，*Jackson*，这种依赖于爬虫，甚至*Cookie*的反序列化漏洞，简单的插件式扫描器是无法扫出的。但是如果把它集成到*BurpSuite*中，那么这些问题就都可以解决。

2.比较强大的扫描器基本都是国外的，它们对国产框架漏洞的支持不是很好。

3.很多出现的反序列化漏洞没有回显，我们需要用到*BurpSuite*自身的*dnslog*，这样就省去了调用第三方*dnslog*的步骤。

### 思路

我们先来总结一下我们需要扫描的漏洞原理和插入点。

- #### *Shiro*漏洞

  - ##### 原理

     	Apache Shiro 1.2.4及以前版本中，加密的用户信息序列化后存储在名为remember-me的Cookie中。攻击者可以使用Shiro的默认密钥伪造用户Cookie，触发Java反序列化漏洞，进而在目标机器上执行任意命令。

  - ##### 插入点

     	cookie

- #### *Fastjson*漏洞
  - ##### 原理

     	fastjson在解析json的过程中，支持使用autoType来实例化某一个具体的类，并调用该类的set/get方法来访问属性。通过查找代码中相关的方法，即可构造出一些恶意利用链。

    ​	 在1.2.48以前的版本中，攻击者可以利用特殊构造的json字符串绕过白名单检测，成功执行任意命令。

  - ##### 插入点

     	body（参数名可以随意定义）

- #### *Jackson*漏洞

  - ##### 原理

     	Jackson-databind 支持 [Polymorphic Deserialization](https://github.com/FasterXML/jackson-docs/wiki/JacksonPolymorphicDeserialization) 特性（默认情况下不开启），当 json 字符串转换的 Target class 中有 polymorph fields，即字段类型为接口、抽象类或 Object 类型时，攻击者可以通过在 json 字符串中指定变量的具体类型 (子类或接口实现类)，来实现实例化指定的类，借助某些特殊的 class，如 TemplatesImpl，可以实现任意代码执行。

  - ##### *插入点*

     	参数值（参数值必须存在）

- #### *Thinkphp*  框架漏洞

  - ##### 原理

     	ThinkPHP 2.x版本中，使用preg_replace的/e模式匹配路由

  - ##### 插入点

     	url中

- #### *dnslog*

  - 以上漏洞三个漏洞没有回显，所以要调用*dnslog*

    点击`Run health check`来查看一下`Burp Collaborator Server`的情况![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141533996.jpeg)

    正常即可，如果出现下面这种情况![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141534671.jpeg)

    就要打开`Poll over unencrypted HTTP` 之后为了方便可以保存项目选项，在下一次启动时导入即可

    ![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141535684.png)

- #### 越权插件（被动扫描）

  - 方便查找越权漏洞

- #### *poc*范式化

  ​	poc规则：

  ​			poc.py (poc+json)   其中  poc直接调用，json定义漏洞规则。

  ​	插件：

  ​			 vulscan.py  主要代码编写。

### 环境准备

- #### 安装*docker*

  ​	使用清华镜像

  ```shell
  curl -fsSL https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/debian/gpg | sudo apt-key add -
  ```

  ​	 配置docker apt

  ```shell
  echo 'deb https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/debian/ buster stable' | sudo tee /etc/apt/sources.list.d/docker.list
  ```

  ​	  更新apt

  ```shell
  sudo apt-get update
  ```

  ​	  进行docker安装

  ```shell
  sudo apt-get install docker-ce
  ```

    	以下是常见命令

  ```shell
  #看docker版本
  	docker -v
  #查看doucker运行状态
  	sudo systemctl status docker
  #开启和停止docker服务
  	sudo systemctl start docker #开启
  	sudo systemctl stop docker  #停止
  #设置自启动
  	sudo systemctl enable docker
  ```

- #### 安装*Vulhub*

  ​	   下载

  ```shell
  git clone https://github.com/vulhub/vulhub.git
  ```

  ​	    安装pip

  ```shell
  sudo apt-get install python-pip
  ```

  ​		 使用pip安装docker-compose

  ```shell
  sudo pip install docker-compose
  ```

- #### 安装靶场（举例）

  ​		 进入shiro目录

  ​		 ![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141631018.jpeg)

  ​		输入启动命令

  ```shell
  #编译靶场环境
  	docker-compose build
  #启动靶场
  	sudo docker-compose up -d
  ```

  ​		 下载完成后，查看服务启动情况

  ```shell
  sudo docker ps
  ```

  ​		![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141633422.jpeg)

  ​			关闭服务

  ```shell
  sudo docker-compose down
  ```

### 主要代码编写

- #### *dsnlog*编写

  ​    首先引入了`IBurpCollaboratorInteraction`库和`IBurpCollaboratorClientContext`库然后引入了时间`time`，让它在完成后进入休眠。![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141806430.png)

  ​	 打印对象![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141808886.png)

  ​	建立循环侦听，有两种侦听方法

  ​		一是获取所有payload的请求结果`fetchAllCollaboratorInteractions`。

  ​		另一种是获取单个payload的请求结果`fetchCollaboratorInteractionsFor`。

  ​		最后再来手动触发`ping  fp8tkoehkvv60uh0j4mgd4fa51brzg.burpcollaborator.net -c 1`。

  ​		具体如下![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141831029.png)

  ​		测试结果成功![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141814195.jpeg)

  

- #### *shiro*模块

  ##### ***vulscan中：***

  ​	由于需要参数进行攻击，我们将它放置主动扫描中

  ​	首先我们定义域名列表，注册调用POC类（这一步所有模块都相同）

  ![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141829087.png)

  ​	![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141829603.png)

  ​		因为这块是主动扫描，所以被动扫描返回空	

  ![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141829375.png)

  ​		一个路径下出现如果多个一样的问题，实现只报一次

  ![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141830849.png)

  ![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141830623.png)

  ​		获取http服务信息![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141834164.png)

  ​		判断是否存在，若不存在则加入列表中![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141842173.png)

  ​			system执行payload![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141843663.png)

  ​			dns接口查询，并返回信息![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141843582.png)

  ***poc.py中：***

​				判断并增加http协议

​				![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141852593.png)

​				多线程并发

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141853330.png)

​				实现函数

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141855874.png)

​		***加载插件扫描结果：***

​					![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141900306.png)

- #### *Fastjosn*模块

     	***vulscan中：***

  ​			定义域名列表，注册调用POC类同上

  ​			分别实现*url*和*域名*扫描模块![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141908727.png)

  ​			定义报告结构列表，遍历并返回![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110142023230.png)

  ​			开始扫描

  ​		***url*扫描**

  ​			判断类型，get则跳过，判断插入类型，不是json则跳过![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141913225.png)

  ​			替换并添加*json*请求头![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141914688.png)

  ​			*dnslog*接口查询![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141920421.png)

  ​		***域名扫描：***

  ​				对没有参数的域名，做一次扫描 , 如果是 repeater 来的包, GET 第一次会转POST扫描一遍![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141924014.png)

  ​				产生*dnglog*，遍历*poc*	![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141925618.png)			*dnslog*接口查询![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141927866.png)

  ​				

  ​		***poc.py中：***

  ​			*Fastjson*  payload![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141906839.png)

  ​    ***加载插件扫描结果：***

  ​                 ![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141929637.png)

- #### *Jackson*模块

  ​	***vulscan中：***

  ​			*url*扫描![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141937128.png)

  ​			判断类型，get则返回 ，判断插入类型不是json则跳过![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141940929.png)

  ​			遍历poc列表![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141940472.png)

  ​			*dnslog*接口查询![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141941682.png)

  ​	***poc.py中：***

  ​			*Jackson* *payload*![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141934411.png)

  ​	***加载插件扫描结果：***（记得替换参数）

  ![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141943814.png)

  ![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141943521.png)

- #### *Thinkphp*  模块

  ​	***vulscan中：***

  ​			这里用的是非阻塞模式，这比较简单，就不分块了![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141948388.png)

  ​	***poc.py中：***

  ​			PHP payload![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141954650.png)

  ​	***加载插件扫描结果：***

  ![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110141947131.png)

- #### ***越权模块***

  ​		写入被动扫描中

  ​		逻辑漏洞

  ![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110142002919.png)

  ​		代码实现

  ​		![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110142004021.png)

  ​		加载插件扫描结果

  ![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110142023214.png)

- #### *其他代码*

  ​		报告漏洞类![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110142019727.png)

  ​		字符串高亮![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110142020805.png)

  ​		日志配置![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110142022802.png)

### 总结

​	这样就完成了这个*BurpSuite*插件的编写，可以扫描***Shiro***，***Fastjson***，***Jackson***，***Thinkphp***漏洞，以及辅助**越权漏洞**的发现。

