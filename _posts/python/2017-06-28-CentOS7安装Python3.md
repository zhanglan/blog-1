---  
layout: post  
#标题配置  
title:  CentOS7.3安装Python3.6  
#时间配置  
date:   2017-06-26 13:58:00 +0800  
#大类配置，比如大数据，数据挖掘，数据分析等  
categories: 平台组  
#小类配置，比如hive，spark，HDFS等  
tag: Python  
---  
  
* content  
{:toc}  
  
 
 
# 下载python3.6编译安装
到python官网下载https://www.python.org
下载最新版源码，使用make altinstall，如果使用make install，在系统中将会有两个不同版本的Python在/usr/bin/目录中。这将会导致很多问题，而且不好处理。

` wget  https://www.python.org/ftp/python/3.6.0/Python-3.6.0.tgz
` tar -xzvf Python-3.6.0.tgz -C  /tmp
` cd  /tmp/Python-3.6.0/

把Python3.6安装到 /usr/local 目录

`  ./configure --prefix=/usr/local
`  make
`  make altinstall

python3.6程序的执行文件：/usr/local/bin/python3.6
python3.6应用程序目录：/usr/local/lib/python3.6
pip3的执行文件：/usr/local/bin/pip3.6
pyenv3的执行文件：/usr/local/bin/pyenv-3.6

# 更改/usr/bin/python链接
` cd /usr/bin
` mv  python python.backup
` ln -s /usr/local/bin/python3.6 /usr/bin/python
` ln -s /usr/local/bin/python3.6 /usr/bin/python3

# 更改yum脚本的python依赖
` cd /usr/bin
` ls yum*
```
yum
yum-config-manager
yum-debug-restore
yum-groups-manager
yum-builddep
yum-debug-dump
yumdownloader
```

更改以上文件头为
` #!/usr/bin/python 改为 #!/usr/bin/python2

# 修改gnome-tweak-tool配置文件
` vi /usr/bin/gnome-tweak-tool
` #!/usr/bin/python 改为 #!/usr/bin/python2

# 修改urlgrabber配置文件
` vi /usr/libexec/urlgrabber-ext-down
` #!/usr/bin/python 改为 #!/usr/bin/python2
