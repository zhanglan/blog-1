---
layout: post
title:  windows下用kerberos客户端和火狐登录HDP域
date:   2017-06-27
categories: security
tag: security,kerberos,spnego
---

原文：[User authentication from Windows Workstation to HDP Realm Using MIT Kerberos Client (with Firefox)](https://community.hortonworks.com/articles/28537/user-authentication-from-windows-workstation-to-hd.html)  
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
