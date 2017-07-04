---
layout: post
title: 在Kerberized集群中配置hive视图
date:  2017-07-03
categories: security
tags: ambari hive
---
测试环境centos7.3。[原文](https://community.hortonworks.com/articles/40658/configure-hive-view-for-kerberized-cluster.html)  
### 未启用kerberos的集群

用admin用户登录Ambari，前往Ambari UI -> Views(格子图标) -> Hive View。每次进入视图前，ambari会启动服务检查。检查到HiveServer时报错：
```
Service 'userhome' check failed: File does not exist: /user/admin
```
已上报错说明，Hive认为每个用户在HDFS中都应该有HOME目录，默认是/user/<用户id>。而admin用户在HDFS中没有创建HOME目录。下面创建这个目录。  
```
$ su - hdfs                                 (切换用户为hdfs，直接用root会报Permission denied)
$ hdfs dfs -mkdir /user/admin
$ hdfs dfs -chown admin /user/admin         (HDFS不管admin用户是否存在)
$ hdfs dfs -chmod 755 /user/admin
```
之后再进入Hive View或Hive View 2.0都正常了。  

### kerberos的集群
安装KDC，并启用kerberos可以参考博文[Kerberos管理(admin)](https://imaidata.github.io/blog/kerberos_admin/)的centos部分。  
启用kerberos后重启所有服务，但App Timeline Server启用失败，报告HDFS处于安全模式，无法改权限。由于处于非生产环境，手工关闭HDFS的安全模式：
```
$ kinit -kt /etc/security/keytabs/hdfs.headless.keytab hdfs-hdp2610        (hdp2610是集群id)
$ hdfs dfsadmin -safemode leave
```
### 启用kerberos的集群下测试hive视图
由于前文的操作已经为admin用户创建了HDFS的`/user/admin`目录，在启用kerberos后Hive视图和Hive视图2.0均检测通过，正常显示：  
![](https://community.hortonworks.com/storage/attachments/5122-screen-shot-2016-06-19-at-94111-pm.png)  

原文中有有多个关于`hadoop.proxyuser`的配置，但Ambari已经自动配置好了，实测根本不需要改。当然按原文那样改也没啥，只是权限放的比较宽而已。  
