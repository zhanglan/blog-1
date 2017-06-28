---
layout: post
title:  Knox代理hbase、hdfs、ambari界面
date:   2017-06-28
categories: security
tags: knox hbase hdfs ambari
---

下面三篇社区文章的内容类似，验证和翻译放在一起。    
[Configure Knox to access HBASE UI](https://community.hortonworks.com/articles/81714/configure-knox-to-access-hbase-ui.html)  
[Configure Knox to access HDFS UI](https://community.hortonworks.com/articles/81713/configure-knox-to-access-hdfs-ui.html)  
[Configure Knox to access AmbarI UI](https://community.hortonworks.com/articles/78361/configure-knox-to-access-ambari-ui.html)  

```
$ ssh root@u1401
$ cd /var/lib/knox/data-2.5.3.0-37/services
$ ls -l
drwxr-xr-x  3 knox knox 4096 Apr 27 09:39 ambariui/
drwxr-xr-x  3 knox knox 4096 Apr 27 09:39 hbaseui/
drwxr-xr-x  3 knox knox 4096 Apr 27 09:39 hdfsui/
drwxr-xr-x  3 knox knox 4096 Apr 27 09:39 jobhistoryui/
drwxr-xr-x  3 knox knox 4096 Apr 27 09:39 oozieui/
drwxr-xr-x  3 knox knox 4096 Apr 27 09:39 rangerui/
drwxr-xr-x  3 knox knox 4096 Apr 27 09:39 sparkhistoryui/
drwxr-xr-x  3 knox knox 4096 Apr 27 09:39 yarnui/
(其它目录略)
```
可以看到三篇社区文章中提到的hbaseui、hdfsui、ambariui目录。进到目录深处看看，发现rewrite.xml和service.xml文件都在，不用再下载了：
```
$ cd hbaseui/1.1.0
$ ls -l
-rw-r--r-- 1 knox knox 5218 Nov 30  2016 rewrite.xml
-rw-r--r-- 1 knox knox 2220 Nov 30  2016 service.xml
```
从上面文件清单可以看出，几个目录及子目录的拥有者是knox，组是knox，权限都符合文章中要求。  
在Ambari中前往 Services -> Knox -> configs -> Advanced topology，增加以下配置：
```
        <service>
                <role>HBASEUI</role>
                <url>http://u1403.ambari.apache.org:16010</url>
            </service>
         <service>
                <role>HDFSUI</role>
                <url>http://u1401.ambari.apache.org:50070</url>
          </service>
         <service>
                <role>AMBARIUI</role>
                <url>http://u1401.ambari.apache.org:8080</url>
            </service>
```
提醒注意的是hbase master装在u1403节点，所以上面HBASEUI配置的是u1403。节点FQDN写死不好，更高明的写法应是使用内置变量，暂时不会。  
保存后，重启knox服务。  
三个服务的代理后的界面URI分别是下面这些：
```
https://u1401.ambari.apache.org:8443/gateway/default/hbase/webui
https://u1401.ambari.apache.org:8443/gateway/default/hdfs
https://u1401.ambari.apache.org:8443/gateway/default/ambari
```
可以通过CURL工具测试一下HDFS:
```
$ curl -i -k -u john:johnldap -X GET 'https://u1401.ambari.apache.org:8443/gateway/default/webhdfs/v1/tmp?op=LISTSTATUS'
```
只所以加上-u参数，是因为我测试的knox配置了ShiroProvider认证。测试的结果是以json格式返回了HDFS`/tmp`目录下的对象清单。  

还可以将上述三个URI输入到浏览器中进行测试。通过浏览器实测，三个URI都可以正常代理HBASE/HDFS/AMBARI的界面。但使用代理后的ambari界面似乎不能正确保存修改的配置。  

