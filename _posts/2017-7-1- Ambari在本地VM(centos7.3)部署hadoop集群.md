---
layout: post
title:  Ambari在本地VM(centos7.3)部署hadoop集群
date:   2017-06-26
categories: base
tag: Ambari,Vagrant,install
permalink: /ambari_centos/
---
本文描述了如何在windows下的虚拟机(centos7.3)中使用Apache Ambari部署HDP(hadoop)集群。在ubuntu14下安装HDP参考[这个](https://imaidata.github.io/blog/ambari_centos/)。  
本文参考了apache官网的这个文章[Installing a cluster with Ambari (with local VMs)](https://cwiki.apache.org/confluence/display/AMBARI/Quick+Start+Guide)。文章写的应是苹果操作系统下(MAC OS)的操作，在windows下会略有不同。  
### 准备
windows下调试linux需要安装终端软件，推荐[Git Bash](https://git-scm.com/)。还需要安装虚拟机软件[VirtualBox](https://www.virtualbox.org/wiki/Downloads)和虚拟化管理软件[Vagrant](http://downloads.vagrantup.com/)。其中VirtualBox还是需要安装扩展包(官网同一下载页面上有链接)。   

安装好上述3个软件后，在windows下创建一个工作目录c:\vagrant，注意选择一个空间足够大的分区，怎么也得几十G的空闲。用资源管理器进入这个新建的目录，在鼠标右键菜单中选择```git bash```，打开git bash窗口。用git克隆github库 “ambari-vagrant” ：
```
$ git clone https://github.com/u39kun/ambari-vagrant.git   (下载ambari官方提供的vagrant配置文件，[参考](https://cwiki.apache.org/confluence/display/AMBARI/Quick+Start+Guide))
$ cd ambari-vagrant/centos7.3
$ ./up.sh 3                 (启动3个VM)
```
如果下载box慢，可以启动前在Vagrantfile中增加本地box链接：
```
  config.vm.box = "bento/centos-7.3"
  config.vm.box_url = "http://repo.imaicloud.com/virtualbox.box"
```
当前目录下有个hosts文件，把这个文件内容添加到`c:/windows/system32/drivers/etc/hosts`文件中。  

vagrant的其他命令可参考[这里](https://github.com/wbwangk/wbwangk.github.io/wiki/virtualbox-vagrant-gitbash%E5%85%A5%E9%97%A8)。  
(vagrant会在windows上增加一个叫VirtualBox Host-Only Network的虚拟网卡（ip是192.68.14.1），不要禁用这个网卡。在vagrant配置文件中定义的private网络就是Host-Only网络。)  

为了加快安装速度和减少错误，要为centos7.3添加国内的163源：
```
$ vagrant ssh c7301                  (密码vagrant)
$ sudo su - root                     (切换为root用户)
$ wget -O /etc/yum.repos.d/CentOS7-Base-163.repo http://mirrors.163.com/.help/CentOS7-Base-163.repo
$ yum update -y
```

使用官方源安装HDP慢且容易出错，所以还要添加Ambari和HDP的本地源：  
```
$ wget -O /etc/yum.repos.d/ambari.repo http://repo.imaicloud.com/ambari/centos7/2.x/updates/2.5.1.0/ambari.repo
$ wget -O /etc/yum.repos.d/hdp.repo http://repo.imaicloud.com/HDP/centos7/2.x/updates/2.6.1.0/hdp.repo
$ yum update -y
```
c7302和c7303节点也要添加163源和本地的Ambari、HDP源。  

### VM间互信（免密码登录）
参考[SSH的入门](https://github.com/wbwangk/wbwangk.github.io/wiki/SSH%E5%85%A5%E9%97%A8)。  
首先，在3个VM上为root用户创建密码：
```
$ vagrant ssh c7301
$ passwd root                 （会提示输入密码）
```
3个VM都要执行上述操作。  
在c7301上生成密钥对，用到openssh的命令：
```
$ ssh-keygen    (生成密钥对，回车3次)
$ cd ~/.ssh
$ ls -l
-rw------- 1 root root  409 Mar 23 10:18 authorized_keys
-rw------- 1 root root 1675 Mar 23 11:04 id_rsa
-rw-r--r-- 1 root root  392 Mar 23 11:04 id_rsa.pub
```
id_rsa是私钥，id_rsa.pub是公钥。  
复制公钥到c7302和c7303，在c7301上执行：
```
$ ssh-copy-id c7302    （输入yes并输入c7302的root密码）
$ ssh-copy-id c7303    （输入yes并输入c7303的root密码）
$ ssh c7302             (尝试一下免密码登录c7302，输入exit返回)
```
### 安装ambari-server
在c7301上执行：
```
$ yum install ambari-server -y  
$ ambari-server setup -s         (会自动安装jdk1.8，173M，较耗时)
$ ambari-server start
```
在windows下的浏览器中输入```http://c7301.ambari.apache.org:8080```就可以调出ambari的界面，初始用户名口令是`admin/admin`。确保windows的hosts文件中有`c7301.ambari.apache.org`的IP地址定义。   

## 利用ambari安装hadoop

用浏览器打开```http://c7301.ambari.apache.org:8080```，登录密码admin/admin。
 1. 点击页面上“Launch Install Winzard”按钮  
 2. 命名集群，如输入HDP2610。  
 3. 选择版本，选择最新的HDP-2.6。选择Local Repository，删除其他库，仅留下redhat7，在Base URL处分别输入：
```
http://repo.imaicloud.com/HDP/centos7/2.x/updates/2.6.1.0
http://repo.imaicloud.com/HDP-UTILS-1.1.0.21/repos/centos7
```
上述两个base url来自`/etc/yum.repos.d/hdp.repo`中的定义。  
 4. 安装选项，在目标主机中输入：
```
c7301.ambari.apache.org
c7302.ambari.apache.org
c7303.ambari.apache.org
```
还需要提供SSH Private Key。这个key存在于~/.ssh/目录下，可以这样查看：
```
$ cat ~/.ssh/id_rsa          （~代表了```/root```，即root用户的HOME目录）
```
把cat命令输出文件内容到屏幕，鼠标选择内容复制，粘贴到浏览器窗口的ssh private key输入框中(带上BEGIN和END行)。  
 5. 确认主机，问题点可能是免密码SSH或agent未启动。出问题就点击红框查看出错日志，逐个解决。  
 6. 选择服务，初学者可以选择HDFS、ZooKeeper、Ambari Metrics这三个服务。如果只选一个ZooKeeper会安装失败，原因未知。    
 7. 分配主机(master)，可以调整每个主机安装的服务，默认即可。  
 8. 分配从机(slave)和客户端，默认即可。  
 9. 配置服务，有红色提示，设定Grafana管理员密码，其他默认。  
 10. 审核，点击部署按钮。  
 11. 安装、启动和测试，显示部署的进度框。这一步耗时较长，也容易出错。需要自己看日志解决。  
 12. 总结，点完成按钮
浏览器切换到了面板页面。在服务页面上，点击Quick Links可以进入各个服务的界面（如果有的话）。  

（安装ambari-agent那一步总出错，手工下载[ambari-agent-2.5.1.0-159.x86_64.rpm]( http://repo.imaicloud.com/ambari/centos7/2.x/updates/2.5.1.0/ambari/ambari-agent-2.5.1.0-159.x86_64.rpm)到本地，然后执行`yum install ambari-agent-2.5.1.0-159.x86_64.rpm`）

## 测试hadoop集群
参考[HDFS的官方文档]进行HDFS的测试：
```
$ curl http://192.168.14.102:50070/webhdfs/v1/?op=LISTSTATUS
{"FileStatuses":{"FileStatus":[
{"accessTime":0,"blockSize":0,"childrenNum":1,"fileId":16386,"group":"hdfs","length":0,"modificationTime":1490698439789,"owner":"hdfs","pathSuffix":"tmp","permission":"777","replication":0,"storagePolicy":0,"type":"DIRECTORY"},
{"accessTime":0,"blockSize":0,"childrenNum":1,"fileId":16387,"group":"hdfs","length":0,"modificationTime":1490698426134,"owner":"hdfs","pathSuffix":"user","permission":"755","replication":0,"storagePolicy":0,"type":"DIRECTORY"}
]}}
```
上述响应中tmp和user是HDFS中的两个目录。查询tmp目录下的内容：
```
curl http://192.168.14.102:50070/webhdfs/v1/tmp?op=LISTSTATUS
```
在u1402或u1403主机上使用命令行测试HDFS：
```
$ hdfs dfs -ls /   (反斜杠表示显示根目录下的文件清单)
Found 2 items
drwxrwxrwx   - hdfs   hdfs            0 2017-03-28 15:57 /tmp
drwxr-xr-x   - hdfs   hdfs            0 2017-03-28 18:05 /user
```
还可以用浏览器访问这个地址来显示HDFS（估计这个UI是HDP实现的）的文件清单：
```
http://192.168.14.102:50070/explorer.html#/
```
## 虚拟机重启
如果hadoop集群出现一些问题，将所有虚拟机重启有时可以解决问题。  
在git bash窗口下：  
```
$ vagrant halt                        (当前目录下的所有虚拟机关机)
$ vagrant up u1401 u1402 u1403        (启动这三个节点的虚拟机)
```
等所有虚拟机重启后，进入Ambari界面，发现所有服务都停止了。点击左侧下边的`Actions`按钮，选择`Start All`，等一会儿，所有的服务就都启动了。  

## 集群备份
利用虚拟机快照可以备份整个hadoop集群。  
在git bash窗口中执行：
```
$ vagrant snapshot save u1401 u1401snap      (保存u1401快照)
$ vagrant snapshot save u1402 u1402snap      (保存u1402快照)
$ vagrant snapshot save u1403 u1403snap      (保存u1403快照)
```
如果某种原因集群损坏了，想恢复之前的备份，则执行：
```
$ vagrant snapshot restore u1401 u1401snap         (恢复u1401)
$ vagrant snapshot restore u1402 u1402snap         (恢复u1402)
$ vagrant snapshot restore u1403 u1403snap         (恢复u1403)
```
u1401是虚拟机id，u1401snap是快照文件名。  

## 新增主机
在windows下的git bash中启动一个新主机u1404：
```
$ vagrant up u1404
$ vagrant ssh u1404
$ sudo su - root
```
在u1404中进行前文说明的免密码SSH的一系列操作。在u1401中执行：
```
$ ssh-copy-id u1404
```
通过浏览器在ambari server的顶部菜单中选择Hosts，然后在“Actions”下拉按钮中选择“+Add New Hosts”。其他的与前文介绍的安装向导类似。只是在第4步的主机选项中只输入新增的主机名：
```
u1404.ambari.apache.org
```
其他的一路默认值。

## 添加服务实例
添加新的zookeeper服务实例到u1404上。通过浏览器在ambari server管理界面上，点击顶部菜单“Services”，通过“Service Actions”下拉按钮点击“+Add Zookeeper Server”。

### heartbeat失去
[参考](http://www-01.ibm.com/support/docview.wss?uid=swg21984577)  
u1402出现heartbeat alerts。在u1402上查看日志：
```
$ cat /var/log/ambari-agent/ambari-agent.log | grep ERROR
ERROR 2017-03-29 06:13:28,491 Controller.py:415 - Unable to reconnect to https://u1401.ambari.apache.org:8441/agent/v1/heartbeat/u1402.ambari.apache.org (attempts=725, details=local variable 'data' referenced before assignment)
```
我参考的连接修改了u1401(ambari-server)和u1402(ambari-agent)的本地语言设置为：
```
$ cat /etc/default/locale
LANG="en_US.UTF-8"
LANGUAGE="en_US:UTF-8"
```
修改/usr/lib/python2.6/site-packages/ambari_simplejson/encoder.py， 修改为：
```
 s = s.decode('utf-8', errors='ignore')
```
同时修改了u1401和u1402，然后重启了ambari-server和ambari-agent，问题解决。

## 其他
如果在调试过程中linux出现外网不通了，可以查看一下/etc/resolv.conf这个文件的内容，一般应为：
```
nameserver 10.0.2.3
```
如果没有这一行的定义（有时显示8.8.8.8），那是因为ambari官方提供的vagrant的脚本，会替换操作系统的resolv.conf，而替换后的结果在国内不能用。

sed命令可以用于文件中字符串替换，命令格式借鉴了vi中的替换：
```
$ sed -i "s/{被替换的内容}/{替换为}/" /etc/ssh/sshd_config
```
vagrant plugin install vagrant-vbguest
http://stackoverflow.com/questions/30175290/laravel-homestead-vagrant-vboxsf-not-available-issue
[这个文档](https://github.com/wbwangk/wbwangk.github.io/wiki/Ambari%E6%B5%8B%E8%AF%95)是在调试ambari过程中我写的一个备忘。

## 使用pdsh批量安装VM
安装的主力是一个开源工具pdsh([parallel distributed shell](http://sourceforge.net/projects/pdsh))。pdsh能远程地在主机上执行命令行或文件中命令。pdsh发行版中包含的pdcp命令能分发复制文件。  

要使用pdsh，必须先实现免密码SSH多台VM。免密码SSH的实现可参考[SSH的入门](https://github.com/wbwangk/wbwangk.github.io/wiki/SSH%E5%85%A5%E9%97%A8)。   
前提：u1401的root用户已经实现了免密码ssh到u1401、u1402、u1403。需要强调的是，u1401对于自己也要实现免密码ssh。  
在u1401上安装pdsh：
```
$ apt install pdsh
```
为了同时在多台机器上执行命令，首先创建一个all_hosts的文本文件：
```
u1401
u1402
u1403
```
测试一下用pdsh加载all_hosts，然后在上述三台VM执行date命令：
```
$ pdsh -R ssh -w ^all_hosts date    (-R指定rcmd模块为ssh)
u1401: Wed Mar 29 01:02:16 UTC 2017
u1402: Wed Mar 29 01:03:35 UTC 2017
u1403: Wed Mar 29 01:05:26 UTC 2017
```
利用pdsh将三个VM的源更改为163源：
```
$ pdsh -R ssh -w ^all_hosts sed -i "s/us.archive.ubuntu.com/mirrors.163.com/" /etc/apt/sources.list
$ pdsh -R ssh -w ^all_hosts sed -i "s/security.ubuntu.com/mirrors.163.com/" /etc/apt/sources.list
```
利用pdsh在所有三个VM上添加ambari和hadoop的本地源。在u1401上执行：
```
$ pdsh -R ssh -w ^all_hosts wget -P /etc/apt/sources.list.d http://repo.imaicloud.com/AMBARI-2.4.2.0/ubuntu14/2.4.2.0-136/ambari.list 
$ pdsh -R ssh -w ^all_hosts wget -P /etc/apt/sources.list.d http://repo.imaicloud.com/hdp/HDP/ubuntu14/HDP.list
$ pdsh -R ssh -w ^all_hosts wget -P /etc/apt/sources.list.d http://repo.imaicloud.com/hdp/HDP-UTILS-1.1.0.21/repos/ubuntu14/HDP-UTILS.list
$ pdsh -R ssh -w ^all_hosts apt-key adv --recv-keys --keyserver keyserver.ubuntu.com B9733A7A07513CAD 
$ pdsh -R ssh -w ^all_hosts apt-get update -y
```
根据屏幕提示就可以看到三台VM上逐次被添加了本地源和逐次进行更新。  
如果需要批量执行其他命令，可以参照执行。  

# centos6.8下部署HDP
现在repo.imaicloud.com下已经有了HDP的centos6本地源（原来只有ubuntu14的源）。HDP本地源部署参考[这个](https://github.com/wbwangk/wbwangk.github.io/wiki/%E6%90%AD%E5%BB%BAHDP%E6%9C%AC%E5%9C%B0%E6%BA%90)。  
### vagrant启动3个虚拟机
在git bash窗口，输入以下命令：
```
$ git clone https://github.com/u39kun/ambari-vagrant.git   (下载ambari官方提供的vagrant配置文件，[参考](https://cwiki.apache.org/confluence/display/AMBARI/Quick+Start+Guide))
$ cd ambari-vagrant/centos6.8
$ vi bootstrap.sh                (也可以用windows的文本编辑器修改这个文件)
```
注释掉bootstrap.sh的下面几行：
```
#cp /vagrant/resolv.conf /etc/resolv.conf
#sudo dd if=/dev/zero of=/swapfile bs=1024 count=1024k
#sudo mkswap /swapfile
#sudo swapon /swapfile
#echo "/swapfile       none    swap    sw      0       0" >> /etc/fstab
```
去掉swapon功能的原因是它总报错。然后启动三个虚拟机：
```
$ ./up.sh 3
```
三个VM的主机名分别是c6801/c6802/c6803。  

### centos6下的环境准备
计划在三台VM(c6801/c6802/c6803)上部署HDP。其中，c6801上安装apache ambari server。  
所有三个VM都要添加ambari和hadoop的本地源：  
```
$ su - root               (切换为root用户，提示符变成#号)
$ wget -O /etc/yum.repos.d/ambari.repo http://repo.imaicloud.com/AMBARI-2.4.2.0/centos6/2.4.2.0-136/ambari.repo
$ wget -O /etc/yum.repos.d/hdp.repo http://repo.imaicloud.com/HDP/centos6/2.x/updates/2.5.3.0/hdp.repo
$ yum update -y
```
c6802和c6803都添加本地源，并更新。

### 免密码SSH
在c6801上生成密钥对，用到openssh的命令：
```
$ ssh-keygen    (生成密钥对，回车3次)
$ ssh-copy-id c6801    （输入yes并输入c6801的root密码，默认是vagrant，下同）
$ ssh-copy-id c6802
$ ssh-copy-id c6803
```
不同于ubuntu，centos6下ssh服务默认允许密码登录，所以不用修改ssh配置文件和重启ssh服务。  

### 安装ambari-server
在c6801上执行：
```
$ yum install ambari-server
$ ambari-server setup -s        (下载和部署JDK较耗时)
$ ambari-server start
```
在windows下的浏览器中输入```http://192.168.14.101:8080```或```http://c6801.ambari.apache.org```就可以调出ambari的界面，用```admin/admin```登录。  

## 利用ambari安装hadoop

用浏览器打开```http://c6801.ambari.apache.org:8080```，登录密码admin/admin。
 1. 点击页面上“Launch Install Winzard”按钮  
 2. 命名集群，如输入hdp2。  
 3. 选择版本，选择最新的HDP-2.5。选择Local Repository，删除其他库，仅留下redhat6(centos6)，在Base URL处分别输入：
```
http://repo.imaicloud.com/HDP/centos6/2.x/updates/2.5.3.0
http://repo.imaicloud.com/HDP-UTILS-1.1.0.21/repos/centos6
```
上述base url的定义来自文件```/etc/yum.repos.d/hdp.repo```。 
 4. 安装选项，在目标主机中输入：
```
c6801.ambari.apache.org
c6802.ambari.apache.org
c6803.ambari.apache.org
```
还需要提供SSH Private Key。这个key存在于~/.ssh/目录下，可以这样查看：
```
$ cat ~/.ssh/id_rsa          （~代表了```/root```，即root用户的HOME目录）
```
把cat命令输出文件内容到屏幕，鼠标选择内容复制，粘贴到浏览器窗口的ssh private key输入框中。  
 5. 确认主机，问题点可能是免密码SSH或agent未启动。出问题就点击红框查看出错日志，逐个解决。  
 6. 选择服务，初学者可以选择HDFS、ZooKeeper、Ambari Metrics这三个服务。如果只选一个ZooKeeper会安装失败，原因未知。    
 7. 分配主机(master)，可以调整每个主机安装的服务，默认即可。  
 8. 分配从机(slave)和客户端，默认即可。  
 9. 配置服务，有红色提示，设定Grafana管理员密码，其他默认。  
 10. 审核，点击部署按钮。  
 11. 安装、启动和测试，显示部署的进度框。这一步耗时较长，也容易出错。需要自己看日志解决。  
 12. 总结，点完成按钮
浏览器切换到了面板页面。在服务页面上，点击Quick Links可以进入各个服务的界面（如果有的话）。  
