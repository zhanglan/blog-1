---
layout: post
title:  SaltStack运维工具基本用法
date:   2017-06-29
categories: base
tag: base,install
---
saltstack简介
------------------------------------

SaltStack是一个服务器基础架构集中化管理平台，具备配置管理、远程执行、监控等功能，一般可以理解为简化版的puppet和加强版的func。SaltStack基于Python语言实现，结合轻量级消息队列（ZeroMQ）与Python第三方模块（Pyzmq、PyCrypto、Pyjinjia2、python-msgpack和PyYAML等）构建。
通过部署SaltStack环境，我们可以在成千上万台服务器上做到批量执行命令，根据不同业务特性进行配置集中化管理、分发文件、采集服务器数据、操作系统基础及软件包管理等，SaltStack是运维人员提高工作效率、规范业务配置与操作的利器。

安装
------------------------------------

首先安装基本的fodora epel yum 源
yum install -y http://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-9.noarch.rpm

通过下面命令检查是否成功
yum repolist

接着开始安装 salt master 和 salt minion，注意 master 是服务器端，minion是客户端，我们的服务器列表如下：
10.10.250.221	c7301
10.10.250.222	c7302
10.10.250.223	c7303
10.10.250.224	c7304
10.10.250.225	c7305
10.10.250.226	c7306
10.10.250.227	c7307
10.10.250.228	c7308

在c7301上安装master和minion，其他c7302~c7308都只安装minion。

安装命令如下：
在master上安装salt-master
yum install salt-master

在minion上安装salt-minion
yum install salt-minion

配置
------------------------------------

上面安装成功之后开始配置部分，在master上需要修改 /etc/salt/master 这个配置文件，如下：
interface: 10.10.250.221    #这里填写master的地址
auto_accept: True

需要传输文件还需做下面配置：
file_roots:
  base:
    - /srv/salt


在minion端，修改 /etc/salt/minion 这个配置文件，如下：
master: 10.10.250.221				#这里填写master的地址
id: open02									#填写minion的主机名

修改上面的配置文件之后，就可以分别启动 master 和 minion服务了
service salt-master start
service salt-minion start

创建主机开机启动服务
systemctl enable salt-master.service
systemctl enable salt-minion.service

服务启动之后，在master执行下面命令就能获取到minion客户端了
salt-key -L

salt-key -A

常用的命令
------------------------------------

测试minion是否连通
salt '*' test.ping	# * 代表所有minion客户端，也可以指定特定的某一个minion，salt 'cdd7302' test.ping

[root@c7301 ~]# salt '*' test.ping
c7301:
    True
c7307:
    True
c7303:
    True
c7306:
    True
c7305:
    True
c7308:
    True
c7304:
    True
c7302:
    True


在客户端上执行命令
salt 'c7302' cmd.run 'ls /home'
[root@c7301 ~]# salt 'c7302' cmd.run 'ls /home'
c7302:
    accumulo
    activity_analyzer
    ambari-qa
    ams
    atlas
    druid
    flume
    hbase
    hcat
    hdfs
    hive
    hue
    infra-solr
    kafka
    kms
    knox
    livy
    logsearch
    mahout
    ranger
    spark
    sqoop
    storm
    tez
    zeppelin
    zookeeper


用salt打通主机之间ssh互信
------------------------------------

在master上执行下面命令，生成ssh密钥
ssh-keygen -t rsa

另外创建传输目录
mkdir -pv /srv/salt/ssh

将密钥文件拷贝到传输目录下
cp /root/.ssh/id_rsa.pub /srv/salt/ssh/id_rsa.pub

添加某一个主机的信任，例如添加c7302，执行下面命令
salt 'c7301' ssh.set_known_host root 10.10.250.222

salt 'c7302' ssh.set_auth_key_from_file root salt://ssh/id_rsa.pub

看看是否通过
ssh 10.10.250.222
[root@c7301 ~]# ssh 10.10.250.222
Last login: Fri Jun 30 08:38:22 2017 from 10.10.11.138
[root@c7302 ~]#

作为运维工具所有服务器上安装通用工具，例如安装wget
------------------------------------

salt '*' cmd.run 'yum install -y wget'



