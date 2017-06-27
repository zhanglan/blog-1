---
layout: post
title:  kerberized集群下HBase REST服务器的启动和测试
date:   2017-06-27
categories: hbase
tag: hbase,rest,kerberized
---

[原文](https://community.hortonworks.com/articles/91425/howto-start-and-test-hbase-rest-server-in-a-kerber.html)  
先确认你的HDP集群启用了kerberos并安装了hbase。  
检查一下你的hbase服务的keytab，我的hbase服务安装在u1403节点上，所以在u1403节点上执行：
```
$ klist -kt /etc/security/keytabs/hbase.service.keytab
Keytab name: FILE:/etc/security/keytabs/hbase.service.keytab
KVNO Timestamp           Principal
---- ------------------- ------------------------------------------------------
   1 06/20/2017 14:45:57 hbase/u1403.ambari.apache.org@AMBARI.APACHE.ORG    (下略)
```
在Ambari中前往 Hbase -> Configs -> Advanced -> Custom Hbase-Site，增加或修改下列参数：
```
hbase.master.keytab.file=/etc/security/keytabs/hbase.service.keytab
hbase.master.kerberos.principal=hbase/_HOST@AMBARI.APACHE.ORG
hbase.rest.authentication.type=kerberos
hadoop.proxyuser.HTTP.groups=*
hadoop.proxyuser.HTTP.hosts=*
hbase.security.authorization=true
hbase.rest.authentication.kerberos.keytab=/etc/security/keytabs/spnego.service.keytab
hbase.rest.authentication.kerberos.principal=HTTP/_HOST@AMBARI.APACHE.ORG
hbase.security.authentication=kerberos
hbase.rest.kerberos.principal=hbase/_HOST@AMBARI.APACHE.ORG
hbase.rest.keytab.file=/etc/security/keytabs/hbase.service.keytab
```
还可以增加两个可选参数来定义Rest服务的端口：
```
hbase.rest.port = 17000
hbase.rest.info.port = 17050
```
在Ambari中前往HDFS -> Configs -> Advanced -> Custom core-site，增加或修改下列参数：
```
hadoop.proxyuser.HTTP.groups=*
hadoop.proxyuser.HTTP.hosts=*
```
重启受影响的HBase和HDFS服务。  
在安装HBase Master的节点(u1403)上执行：
```
$ su - hbase                        (切换为hbase用户)
$ kinit -kt hbase.service.keytab hbase/u1403.ambari.apache.org@AMBARI.APACHE.ORG
$ /usr/hdp/current/hbase-master/bin/hbase-daemon.sh start rest -p 17000 --infoport 17050       (启动REST服务器)
```
分别在无票据和有票据的情况下，测试REST服务器：
```
$ kdestroy
$ klist
klist: No credentials cache found (ticket cache FILE:/tmp/krb5cc_0)
 
$ curl --negotiate -u : 'http://u1403.ambari.apache.org:17000/status/cluster'
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=ISO-8859-1"/>
<title>Error 401 Authentication required</title>
 
# kinit -kt /etc/security/keytabs/hbase.service.keytab hbase/u1403.ambari.apache.org@AMBARI.APACHE.ORG
# curl --negotiate -u : 'http://u1403.ambari.apache.org:17000/status/cluster'
3 live servers, 0 dead servers, 10.6667 average load
 
3 live servers
    hdp253k1.hdp:16020 1490688381983
        requests=0, regions=11
        heapSizeMB=120        maxHeapSizeMB=502
```
测试过程中层出现hbase RegionServer启动失败的情况，导致curl调用超时。从日志`hbase-hbase-master-u1403.log`上看，报告`Clock skew too great`，推测是三个节点的时间不一致，导致kerberos票据失效。调整了三个节点的时间([参考](https://github.com/wbwangk/wbwangk.github.io/wiki/0%E7%AC%94%E8%AE%B0#%E6%97%B6%E9%97%B4%E5%90%8C%E6%AD%A5ntpd))。重启所有HDP服务，curl终于正确返回结果了。  
