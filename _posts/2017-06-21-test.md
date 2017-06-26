---
layout: post
title:  Kerberos管理
date:   2017-06-26
categories: security
tag: security kerberos
---
## Kerberos服务器安装/管理
[参考](https://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.6.0/bk_security/content/_kerberos_overview.html)  
[一个kerberos入门](https://steveloughran.gitbooks.io/kerberos_and_hadoop/content/)  
安装环境是ubuntu14.4，安装环境参考[Ambari在本地VM部署集群](https://imaidata.github.io/blog/2017/06/26/Ambari%E5%9C%A8%E6%9C%AC%E5%9C%B0VM%E9%83%A8%E7%BD%B2%E9%9B%86%E7%BE%A4/)。  

### 1.安装KDC Server
在节点u1404上安装Install the KDC Server：
```
$ apt-get install krb5-kdc krb5-admin-server
```
（第一次尝试时，提示krb5-user依赖冲突。用手机当热点执行apt-get update后正常）  
安装过程中出现提示窗口让输入Default Kerberos version 5 realm，保留默认值AMBARI.APACHE.ORG。然后出现两次让输入hostname，都输入的"u1404.ambari.apache.org"。最后提示说这个向导没有自动建立一个kerberos realm，如果想建立就执行命令"krb5_newrealm"。相关说明在/usr/share/doc/krb5-kdc/README.KDC中。  
```
$ krb5_newrealm
master key name 'K/M@AMBARI.APACHE.ORG'
Enter KDC database master key:    (输入两次密码，密码是vagrant)
```
必须使用krb5_newrealm创建realm，否则无法启动下列服务。  
启动KDC server和KDC admin server：
```
$ service krb5-kdc restart                     （如果不执行krb5_newrealm就无法启动这个服务）
$ service krb5-admin-server restart            （如果不执行krb5_newrealm就无法启动这个服务）
```
### 2.创建Kerberos Admin(管理员账号)
通过创建admin主体来建立KDC admin(管理员账号)：
```
$ kadmin.local -q "addprinc root/admin"         (输入两次密码，密码是vagrant)
```
将刚创建的admin主体添加到KDC ACL中：
```
$ echo "*/admin@AMBARI.APACHE.ORG *" >> /etc/krb5kdc/kadm5.acl
$ service krb5-admin-server restart
```
### 3. 简单测试
查看主体清单的方法：在u1404节点（安装KDC的节点）上执行：
```
$ kinit root/admin              (登录管理员账号)
$ kadmin.local
kadmin.local: list_principals  （列出现有主体清单）
HTTP/u1402.ambari.apache.org@AMBARI.APACHE.ORG
HTTP/u1403.ambari.apache.org@AMBARI.APACHE.ORG
K/M@AMBARI.APACHE.ORG
admin/admin@AMBARI.APACHE.ORG
(略)
```
### 4. 新增主体
以交互方式增加主体：
```
$ kadmin.local
kadmin.local:  addprinc webb
WARNING: no policy specified for webb@AMBARI.APACHE.ORG; defaulting to no policy
Enter password for principal "webb@AMBARI.APACHE.ORG":
Re-enter password for principal "webb@AMBARI.APACHE.ORG":
Principal "webb@AMBARI.APACHE.ORG" created.
```
或者以单行命令的方式增加主体：
```
$ kadmin.local -q "addprinc webb@AMBARI.APACHE.ORG"           (增加主体webb)
$ kadmin.local -q "cpw -pw wang webb@AMBARI.APACHE.ORG"       (将webb的密码设置为wang)
```
在执行上述命令前需要先用kinit命令登录管理员账号。  

### 5.生成keytab
为主体webb@AMBARI.APACHE.ORG生成keytab文件，文件名是webb.keytab（当前目录下生成）
```
$ kadmin.local
kadmin.local:  ktadd -k webb.keytab webb@AMBARI.APACHE.ORG
Entry for principal webb@AMBARI.APACHE.ORG with kvno 3, encryption type aes256-cts-hmac-sha1-96 added to keytab WRFILE:webb.keytab.
Entry for principal webb@AMBARI.APACHE.ORG with kvno 3, encryption type arcfour-hmac added to keytab WRFILE:webb.keytab.
Entry for principal webb@AMBARI.APACHE.ORG with kvno 3, encryption type des3-cbc-sha1 added to keytab WRFILE:webb.keytab.
Entry for principal webb@AMBARI.APACHE.ORG with kvno 3, encryption type des-cbc-crc added to keytab WRFILE:webb.keytab.
kadmin.local:  exit
```
使用keytab文件登录（而不是密码）：
```
$ kinit -k -t webb.keytab webb
```
显示keytab文件中主体和加密类型：
```
$ klist -e -k webb.keytab 
Keytab name: FILE:webb.keytab
KVNO Principal
---- --------------------------------------------------------------------------
   2 webb@AMBARI.APACHE.ORG (aes256-cts-hmac-sha1-96)
   2 webb@AMBARI.APACHE.ORG (arcfour-hmac)
   2 webb@AMBARI.APACHE.ORG (des3-cbc-sha1)
   2 webb@AMBARI.APACHE.ORG (des-cbc-crc)
   3 webb@AMBARI.APACHE.ORG (aes256-cts-hmac-sha1-96)
   3 webb@AMBARI.APACHE.ORG (arcfour-hmac)
   3 webb@AMBARI.APACHE.ORG (des3-cbc-sha1)
   3 webb@AMBARI.APACHE.ORG (des-cbc-crc)
```
以上是安装了启用了kerberos的AMBARI下的keytab。而仅安装kerberos后测试的结果是：
```
$  klist -e -k webb.keytab
Keytab name: FILE:webb.keytab
KVNO Principal
---- --------------------------------------------------------------------------
   2 webb@AMBARI.APACHE.ORG (aes256-cts-hmac-sha1-96)
   2 webb@AMBARI.APACHE.ORG (arcfour-hmac)
   2 webb@AMBARI.APACHE.ORG (des3-cbc-sha1)
   2 webb@AMBARI.APACHE.ORG (des-cbc-crc)
```

## kerberos代理
[参考](https://pypi.python.org/pypi/kdcproxy/0.3.1)  
freeipa带有kerberos KDC代理功能。执行ipa-server-install的时候有提示：
```
Configuring the web interface (httpd). Estimated time: 1 minute
  [16/21]: create KDC proxy user
  [17/21]: create KDC proxy config
  [18/21]: enable KDC proxy
```
通过KdcProxy可以让kinit客户端以https协议通过代理连接到KDC。  
首先需要把freeipa的证书复制到客户端所在的机器上：
```
$ cd /etc/ipa
$ openssl X509 -in ca.crt -out ca.pem
$ scp ca.pem root@c7002:/etc/ssl/.    (复制到c7002节点)
```
配置c7002节点的kerberos客户端配置文件(/etc/krb5.conf)：
```
 [libdefaults]
   default_realm = AMBARI.APACHE.ORG

[realms]
   AMBARI.APACHE.ORG = {
     http_anchors = FILE:/etc/ssl/ca.pem
     kdc = https://c7004.ambari.apache.org/KdcProxy
     kpasswd_server = https://c7004.mbari.apache.org/KdcProxy
```
在c7002节点上测试KDC代理。环境变量KRBT_TRACE可以让kerberos调试信息输出到文件(/dev/stdout表示输出到屏幕)：
```
$ env KRB5_TRACE=/dev/stdout kinit admin
[29875] 1497331322.495260: Getting initial credentials for admin@AMBARI.APACHE.ORG
[29875] 1497331322.495434: Sending request (196 bytes) to AMBARI.APACHE.ORG
[29875] 1497331322.495463: Resolving hostname c7004.ambari.apache.org
[29875] 1497331322.619668: TLS certificate name matched "c7004.ambari.apache.org"
[29875] 1497331322.623893: Sending HTTPS request to https 192.168.70.104:443
[29875] 1497331322.629792: Received answer (272 bytes) from https 192.168.70.104:443
[29875] 1497331322.629809: Terminating TCP connection to https 192.168.70.104:443
[29875] 1497331322.630005: Response was not from master KDC
[29875] 1497331322.630206: Received error from KDC: -1765328359/Additional pre-authentication required
[29875] 1497331322.630269: Processing preauth types: 136, 19, 2, 133
[29875] 1497331322.630288: Selected etype info: etype aes256-cts, salt "Ar>LhD/*\>smo3/3", params ""
[29875] 1497331322.630296: Received cookie: MIT
Password for admin@AMBARI.APACHE.ORG:
（略）
```
从上面的调试信息可以看出kinit连接是通过https的443端口连接到KDC的。  
## windows下kerberos认证(非AD)
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
