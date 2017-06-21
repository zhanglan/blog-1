---
layout: post
title:  评论系统之云跟帖
date:   2017-06-08 00:00:00 +0800
categories: document
tag: 教程
---

* content
{:toc}

网易云跟帖
====================================

由于云跟帖跟多说有一些小区别。对多说来说，安装完成之后，注册站点跟实际使用域名并没有硬性的要求必须一致，所以写完之后无论在什么地方都可以执行，而网易的这个云跟帖不同的是，会在每次页面加载的时候先获取当前访问的域名，然后跟你注册云跟帖时候的域名做检查，一致的话会放行并正确显示评论系统，不一致就会直接返回错误码，并且在页面不会显示任何内容。

所以，以后想要使用本Jekyll模板的小盆友们，`一定要记得修改自己的云跟帖导入代码`，才能让自己博客的评论系统生效。具体步骤如下：

登录云跟帖
---------------------

使用自己的账号登录[https://gentie.163.com/](https://gentie.163.com/)，然后点击右上角的`后台管理`

![/styles/images/yungentie/01.png]({{ '/styles/images/yungentie/01.png' | prepend: site.baseurl  }})

初始化
---------------------

如果是第一次登录云跟帖会提示完善站点基本信息。

![/styles/images/yungentie/01.png]({{ '/styles/images/yungentie/02.png' | prepend: site.baseurl  }})

完成之后点击`获取代码`，选择合适的皮肤进行设置，本博客是蓝色，固定位置。

![/styles/images/yungentie/01.png]({{ '/styles/images/yungentie/03.png' | prepend: site.baseurl  }})

复制引入代码
---------------------

点击`WEB代码`, 在右侧的`跟贴完整模块代码(Web单独版)`中复制展示框中的代码

![/styles/images/yungentie/01.png]({{ '/styles/images/yungentie/04.png' | prepend: site.baseurl  }})

修改模板代码
---------------------

修改`_includes/LessOrMore/comments-providers/yungentie`文件中的内容，将文件中的代码全部删除，并粘贴刚刚从云跟帖网站复制的代码。

![/styles/images/yungentie/01.png]({{ '/styles/images/yungentie/05.png' | prepend: site.baseurl  }})
