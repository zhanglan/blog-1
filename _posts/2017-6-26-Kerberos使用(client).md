---
layout: post
title:  Kerberos使用(client)
date:   2017-06-26
categories: security
tag: security kerberos
---

### kerberos客户端安装
ubuntu下安装kerberos客户端：
```
$ apt install krb5-user  (如果提示包依赖错误，就用手机上网执行apt-get udpate)
```
centos下安装kerberos客户端：
```
$ yum install krb5-workstation
```

安装过程中，按提示输入下列信息：  
- KDC： ```u1404.ambari.apache.org```  
- realm： ```AMBARI.APACHE.ORG```  
- 管理服务器: ```u1404.ambari.apache.org```  

windows下kerberos客户端的安装和使用比较特殊，会在下文中提及。  

### kerberos客户端的使用
kerberos客户端的常用命令是kinit、klist、kdestroy、kpasswd。
kinit用户登录。登录过程就是客户端从KDC获取票据TGT的过程。登录成功后，票据被缓存在本地。实际上就是建立了安全会话。  
klist用户查看当前票据缓存中内容，可以理解为查看当前会话状态。  
kdestroy用于退出登录，即销毁缓存中票据。  
kpasswd用于修改用户(主体)口令。  
ktutil 客户端工具，可以生成keytab。下一节详述。    
```
$ klist
klist: No credentials cache found (ticket cache FILE:/tmp/krb5cc_0)
```
上面的结果表明缓存文件中没有票据。  
```
$ kinit webb
Password for webb@AMBARI.APACHE.ORG:(输入密码)
$ klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: webb@AMBARI.APACHE.ORG

Valid starting       Expires              Service principal
06/22/2017 08:14:21  06/22/2017 18:14:21  krbtgt/AMBARI.APACHE.ORG@AMBARI.APACHE.ORG
        renew until 06/29/2017 08:14:18
$ kdestroy    (退出，如果再执行klist就会又显示No credentials cache found)
```
改密码（密码从wang改成1）：
```
$ kpasswd webb
Password for webb@AMBARI.APACHE.ORG: wang
Enter new password: 1
Enter it again: 1
Password changed.
```

### 使用keytab
什么时候需要keytab？答案是人无法输入密码的时候，如程序访问kerberized hadoop集群时，需要用密码生成keytab文件，让程序用keytab文件来完成认证过程。  

管理员可以使用kadmin命令生成keytab文件。而普通用户可以用ktuitl命令生成keytab，前提是你得有一个主体和密码。  
```
$ kinit webb@AMBARI.APACHE.ORG
$ ktutil         
ktutil: addent -password -p webb -k 1 -e RC4-HMAC        (输入webb的密码)
ktuitl: list                                             (列出内存中的key)
slot KVNO Principal
---- ---- ---------------------------------------------------------------------
   1    1                   webb@AMBARI.APACHE.ORG
ktutil: wkt webb.keytab                                   (将内存中的key写入keytab文件，生成的文件位于当前目录)
ktutil: q               (退出)
```

使用keytab登录：
```
$ kinit -k -t webb.keytab webb 
```
webb.keytab需要位于当前目录，或者用绝对路径。由于一个keytab文件中可以包含多个主体，所以后面要跟上主体id(webb)。  

### 通过命令行访问启用了kerberos的集群
通过命令行访问启用了kerberos的集群（kerberized cluster），必须首先用kinit登录。
```
$ kdestroy                  (登出)
$ hdfs dfs -ls /tmp
(省略一些java堆栈报错信息)
ls: Failed on local exception: java.io.IOException: javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]; Host Details : local host is: "u1403.ambari.apache.org/192.168.14.103"; destination host is: "u1401.ambari.apache.org":8020;
```
GSS是Generic Security Services的缩写。hdfs向kerbers进程(通过GSS协议)请求TGT票据，kerberos说没有有效凭据。  
```
$ kinit webb
$ hdfs dfs -ls /tmp
Found 17 items
...(省略一些)
drwxr-xr-x   - webb      hdfs          0 2017-05-04 11:33 /tmp/webb
drwx------   - webb2     hdfs          0 2017-05-04 11:36 /tmp/webb2
```

### 通过REST API访问启用了kerberos的集群
```
$ curl --version
Features:  GSS-Negotiate (其它特性略)
$ curl -v -i --negotiate -u : http://u1401.ambari.apache.org:50070/webhdfs/v1/tmp/webb?op=LISTSTATUS
> GET /webhdfs/v1/tmp/webb?op=LISTSTATUS HTTP/1.1
< HTTP/1.1 401 Authentication required
< WWW-Authenticate: Negotiate
< Set-Cookie: hadoop.auth=; Path=/; HttpOnly
* Server auth using GSS-Negotiate with user ''
> GET /webhdfs/v1/tmp/webb?op=LISTSTATUS HTTP/1.1
> Authorization: Negotiate YIICgAYJKoZIhvcSAQICAQBuggJvMIICa6ADAgEFoQMCAQ6iBwMFACAAAACjggF+YYIBejCCAXagAwIBBaETGxFBTUJBUkkuQVBBQ0hFLk9SR6IqMCigAwIBA6EhMB8bBEhUVFAbF3UxNDAxLmFtYmFyaS5hcGFjaGUub3Jno4IBLDCCASigAwIBEqEDAgEBooIBGgSCARbj+QAnGHQabWxjkiDMUBF+7xxVJMPFPupO+52yR1UNtkGn96CGaYQLuNeCuY8hG99fi3GjFex1QAEoOJxpWeVCHflprJj14vR/mIVNmcgUEv5qjAw8osZCqdM4bmMQk/zwR1En38AFTza5nP/nUG5iblNYIzE5NCufUCK6XO8CPR1FxtPTcgMm5XucWTjsLpLCf3K15xjDkWoEhPRSkhZ0hOnuNJh+/99Bk+b158KytTrgCXazqWxQZ/9VRM1e5Q6eF56Xer6EMpFiNQg2esrA0AcUCYxfFdWu/I7lDClfVNldD9YARYR15J0qOW2ahLBQLeyRdayX3f88uA5Cgqerb1qlVqNHIOCEk+VXktUREvjels38RaSB0zCB0KADAgESooHIBIHFLOg8rw9hK23hGcOUilryj1kHw4jn3zh39kUFVqSRXywCis7wzeJZPEh0XOyV7C3Hb6ZDD8sFN3BeGY3RpSA4UAoGHMZHsidRKMGblblknfXX74XjsAVmBMXq9BPnYL9GmuqW5HEBB97QArorcFWJbUA5Hwyzh2R6sfSMxi0q7M+QGE4yEdg5rrzBFSNeQF6a20sRyAzEj3lYqVcq1PSnlVCKoYlpf3bkFtdmE3tvd7fvEdRP237v1xDiaQRN6r2fs5cvPeI=
< HTTP/1.1 200 OK
< WWW-Authenticate: Negotiate YGoGCSqGSIb3EgECAgIAb1swWaADAgEFoQMCAQ+iTTBLoAMCARKiRARCmg3tGs/VORtw83iOQbGGE/ctFEWAorVtPkgEOv+++fRX76t23FNYNU+JnSJoxJNQDS+BaXvsax4NkeIdFhp4JYcN
< Set-Cookie: hadoop.auth="u=webb&p=webb@AMBARI.APACHE.ORG&t=kerberos&e=1498565926445&s=xJ6CE/K7I6iblHPTZoka5zlvbPg="; Path=/; HttpOnly
<
{"FileStatuses":{"FileStatus":[
{"accessTime":1493947036270,"blockSize":134217728,"childrenNum":0,"fileId":22980,"group":"hdfs","length":3,"modificationTime":1493947036592,"owner":"webb","pathSuffix":"t1.txt","permission":"644","replication":3,"storagePolicy":0,"type":"FILE"}
]}}
```
通过curl --version命令可以看到curl内置了GSS-Negotiate(通过GSS进行协商认证)，用--negotiate参数可以告诉curl使用Negotiate认证，后面的-u和冒号参数是配套和必须的。-v和-i参数是为了看清楚整个交互过程。大于号`>`表示发出信息，小于号`<`表示收到的信息。  
curl实际发出了两次请求。第一次的响应码是401，响应头中放置了`WWW-Authenticate: Negotiate`，代表服务器想使用协商认证（集群启用kerberos的结果），Set-Cookie使相关值变成空串`""`。  
curl发现了响应头的`GSS-Negotiate`，并且还有`--negotiate`参数，它就kerberos进取通过GSS协议请求票据。请求成功后发送了第二次请求，第二次请求头中的`Authorization: Negotiate`后面的应为票据信息。服务器验证票据通过后，发回了状态码200的响应。响应头中`WWW-Authenticate: Negotiate`的作用不清楚。  
最后是json格式的HDFS中`/tmp/webb`目录下的内容（有个叫t1.txt的文件）。

### 通过浏览器访问启用了kerberos的集群

当前的hadoop界面(UI)的实现基本上都是html+ajax(REST)，原理上与前面讲的curl发出REST请求基本是一样的。  
当浏览器收到401返回码和响应头的`WWW-Authenticate: Negotiate`信息后，它会与本地的kerberos进程进行交互，获取票据。从安全性的考虑，浏览器不能收到`WWW-Authenticate: Negotiate`响应头就发送票据给对方，一般要手工设置“可信任域”。不同的浏览器设置方式不同。本文使用火狐浏览器进行测试。  
不同的操作系统下，情况也会不同。现在流行的桌面操作系统有：  
- linux桌面  
- MAC OS  
- windows  
linux桌面下的测试比较简单；MAC OS没找到办法；windows下的测试下面有专门的一章。  

ubuntu桌面系统自带的浏览器就是火狐。首先要配置“可信任域”。在火狐地址栏输入：about:config并回车。然后在search输入框中输入"negotiate"，双击配置项network.negotiate-auth.trusted-uris，输入```.ambari.apache.org```(注意别忽略最前面的点)。  
ubuntu桌面系统下组合键`ctrl+alt+f2`可以切换到命令行方式(f1到f6都行)，组合键`ctrl+alt+f7`返回图形界面。在命令行方式下用kinit获取票据，然后在地址栏输入http://u1401.ambari.apache.org:50070/webhdfs/v1/tmp/webb?op=LISTSTATUS并回车。  
如果一切正常，浏览器中会显示一个json串。浏览器进行kerberos协商认证的原理与curl是一样的。如果用kdestroy命令登出，再用浏览器访问同一个url，会报错。  

### windows下kerberos认证(非AD)
[原文](https://community.hortonworks.com/articles/28537/user-authentication-from-windows-workstation-to-hd.html)  
在windows10专业版下进行的测试，win10已经加入了AD域。  
- windows域是HOME.LANGCHAO.COM
- HDP集群域是AMBARI.APACHE.ORG  

测试步骤：  

1. 安装MIT Kerberos客户端(windows版)  
2. 安装windows版火狐浏览器  
3. 在火狐中启用Kerberos支持  
4. 通过Kerberos客户端获得kerberos票据  
5. 通过火狐打开HDP界面  

#### 1.安装MIT Kerberos客户端(windows版)
64位版下载地址：[http://web.mit.edu/kerberos/dist/kfw/4.0/kfw-4.0.1-amd64.msi](http://web.mit.edu/kerberos/dist/kfw/4.0/kfw-4.0.1-amd64.msi)  
安装的默认目录是```C:\Program Files\MIT\Kerberos```。  
将HDP的/etc/krb5.conf复制到上述目录下，改名为krb5.ini。  
在windows下添加两个环境变量：  
```
KRB5_CONFIG=C:\Program Files\MIT\Kerberos\krb5.ini
KRB5CCNAME=c:\temp\krb5cache
```
上面的c:\temp目录必须有写权限。  
![](https://community.hortonworks.com/storage/attachments/3561-1.png)  
增加完环境变量后重启windows。  

#### 2.安装windows版火狐浏览器  
火狐浏览器下载地址：  
[https://www.mozilla.org/en-US/firefox/new/](https://www.mozilla.org/en-US/firefox/new)  

#### 3.在火狐中启用Kerberos支持
在火狐中地址栏输入```about:config```并回车，然后搜索和设置下面的参数：
```
network.negotiate-auth.trusted-uris = .ambari.apache.org
network.negotiate-auth.using-native-gsslib = false
network.negotiate-auth.gsslib = C:\Program Files\MIT\Kerberos\bin\gssapi32.dll
network.auth.use-sspi = false
network.negotiate-auth.allow-non-fqdn = true
```
#### 4.通过Kerberos客户端获得kerberos票据  
安装完kerberos客户端后windows桌面上会创建叫“MIT Kerberos Ticket Manager”的快捷方式。双击快捷方式进入kerberos客户端。由于windows已经加入了域，会显示：
```
> wbwang@HOME.LANGCHAO.COM                   6月 16  20.14（9 h, 13 m remaining)
```  
点击“Get Ticket”按钮，在弹出窗口中输入kerberos主体和密码，主体如```webb@AMBARI.APACHE.ORG```。  
（实测中曾经报错kerberos 5:clock skew too great，在KDC主机上安装NTP服务后解决，时间调整后ambari-server也要重启）  
当成功获取票据后，kerberos客户端会显示两个票据：
```
> webb@AMBARI.APACHE.ORG                     6月 16 20:37 (9 h, 31 m remaining)
> wbwang@HOME.LANGCHAO.COM                   6月 16 20:14（9 h, 13 m remaining)
```
#### 5.通过火狐打开HDP界面
打开火狐，输入Ambari地址，如：```http://u1401.ambari.apache.org:8080```，逐次点击HDFS ->  Quick Links -> NameNode UI -> Utilities -> Browse the file system 或者直接打开```http://u1401.ambari.apache.org:50070/explorer.html```，就可以看到HDFS中文件清单了。  
