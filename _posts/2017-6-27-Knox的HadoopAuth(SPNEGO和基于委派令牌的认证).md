---
layout: post
title:  Knox的HadoopAuth(SPNEGO和基于委派令牌的认证)
date:   2017-06-27
categories: knox
tag: knox,spnego,hadoopauth
---
本文参考了[Hadoop Auth (SPNEGO and delegation token based authentication) with Apache Knox](https://community.hortonworks.com/articles/85550/hadoop-auth-spnego-and-delegation-token-based-auth.html)。  
本节介绍使用Apache Knox配置Hadoop Auth（SPNEGO和基于代理令牌的身份验证）。
[Hadoop Auth](https://hadoop.apache.org/docs/stable/hadoop-auth/index.html)是一个Java库，可以为HTTP请求启用Kerberos SPNEGO身份验证。它在对受保护的资源进行身份验证后，成功认证后，Hadoop Auth创建一个带有认证令牌，用户名，用户主体，认证类型和到期时间的签名HTTP Cookie。该cookie用于所有后续的HTTP客户端请求，以访问受保护的资源，直到cookie过期。

鉴于Apache Knox的可插拔身份验证提供程序，只需少量配置更改即可轻松地使用Apache Knox设置Hadoop Auth。  
假设已经有一个正常工作的Hadoop集群与Apache Knox，而且集群是Kerberized。  

#### 配置
要在Apache Knox中使用Hadoop Auth，我们需要更新Knox拓扑。Hadoop Auth配置为提供程序，因此我们需要通过提供程序参数进行配置。Apache Knox使用与Apache Hadoop相同的配置参数，可以预期其类似的行为。要使用Ambari更新Knox拓扑，请转到Knox - > Configs - > Advanced拓扑结构。  
以下是Apache Knox拓扑文件中HadoopAuth提供程序代码段的示例（如果通过ambari修改knox的配置文件，在配置文件的Advanced atopology小节中）：
```
<provider>
  <role>authentication</role>
  <name>HadoopAuth</name>
  <enabled>true</enabled>
  <param>
    <name>config.prefix</name>
    <value>hadoop.auth.config</value>
  </param>
  <param>
    <name>hadoop.auth.config.signature.secret</name>
    <value>knox-signature-secret</value>
  </param>
  <param>
    <name>hadoop.auth.config.type</name>
    <value>kerberos</value>
  </param>
  <param>
    <name>hadoop.auth.config.simple.anonymous.allowed</name>
    <value>false</value>
  </param>
  <param>
    <name>hadoop.auth.config.token.validity</name>
    <value>1800</value>
  </param>
  <param>
    <name>hadoop.auth.config.cookie.domain</name>
    <value>ambari.apache.org</value>
  </param>
  <param>
    <name>hadoop.auth.config.cookie.path</name>
    <value>gateway/default</value>
  </param>
  <param>
    <name>hadoop.auth.config.kerberos.principal</name>
    <value>HTTP/u1401.ambari.apache.org@AMBARI.APACHE.ORG</value>
  </param>
  <param>
    <name>hadoop.auth.config.kerberos.keytab</name>
    <value>/etc/security/keytabs/spnego.service.keytab</value>
  </param>
  <param>
    <name>hadoop.auth.config.kerberos.name.rules</name>
    <value>DEFAULT</value>
  </param>
</provider>
```
（需要额外说明的是config.prefix参数(默认是hadoop.auth.config)，它指定了多个参数的前缀）  
以下是需要最少更新的参数：
 1. hadoop.auth.config.signature.secret - 这是用于在hadoop.auth cookie中签署委托令牌的秘密。需要在给定群集中的Knox网关的所有实例中使用相同的秘密。否则，委托令牌将失败验证，每个请求将重复身份验证。
 2. cookie.domain - 用于存储身份验证令牌的HTTP cookie的域（例如mycompany.com）
 3. hadoop.auth.config.kerberos.principal - Web应用程序Kerberos主体名称。Kerberos主体名称必须以HTTP / ...开头。
 4. hadoop.auth.config.kerberos.keytab - 包含上面指定的kerberos主体的凭据的keytab文件的路径。

如果您使用Ambari，您将不得不重新启动Knox，这是一个Ambari要求，如果拓扑在Ambari之外更新，则不需要重新启动（Apache Knox每次更新拓扑时间戳时重新加载拓扑）。
如果通过ambari更新参数，需要在ambari界面中点击Save按钮保存，并通过橙黄色按钮重启相关服务。之后会发现```/usr/hdp/current/knox-server/conf/topologies/```目录下的default.xml修改更新了。 

#### 测试

 1. 让我们创建一个用户'guest'与组'用户'。请注意，由于属性“hadoop.proxyuser.knox.groups = users”选择了组用户。
```
$ useradd guest -u 1590 -g users
```
 2. 用'kadmin.local'命令增加主体：
```
$ kadmin.local -q "addprinc guest/u1401.ambari.apache.org”
```
 3. 用kinit登录：
```
$ kinit guest/u1401.ambari.apache.org@AMBARI.APACHE.ORG
```
用kinit登录后，kerberos会话就可以跨客户端请求使用，如curl。以下curl命令可用于从HDFS请求目录列表，同时通过-negotiate标志与SPNEGO进行身份验证。首先是不通过knox网关，直接访问启用了kerberos的HDFS(取HDFS中/tmp目录下对象清单)：
```
$ curl -k -i --negotiate -u : http://u1401.ambari.apache.org:50070/webhdfs/v1/tmp?op=LISTSTATUS
```
通过knox网关访问HDFS:
```
$ curl -k -i --negotiate -u : https://u1401.ambari.apache.org:8443/gateway/default/webhdfs/v1/tmp?op=LISTSTATUS
HTTP/1.1 401 Authentication required
Date: Fri, 24 Feb 2017 14:19:25 GMT
WWW-Authenticate: Negotiate
Set-Cookie: hadoop.auth=; Path=gateway/default; Domain=ambari.apache.org; Secure; HttpOnly
Content-Type: text/html; charset=ISO-8859-1
Cache-Control: must-revalidate,no-cache,no-store
Content-Length: 320
Server: Jetty(9.2.15.v20160210)
 
HTTP/1.1 200 OK
Date: Fri, 24 Feb 2017 14:19:25 GMT
WWW-Authenticate: Negotiate YGwGCSqGSIb3EgECAgIAb10wW6ADAgEFoQMCAQ+iTzBNoAMCARCiRgRE26GeVRA0WkP7eb3csszuxUnSBDFK0NWH2+ai5pFY1onksiVOqjLkY8YS1xF5CshT4IwfrOHz6ivG6218X6oOSb0oCaU=
Set-Cookie: hadoop.auth="u=guest&p=guest/u1401.ambari.apache.org@AMBARI.APACHE.ORG&t=kerberos&e=1487947765114&s=fNpq9FYy2DA19Rah7586rgsAieI="; Path=gateway/default; Domain=ambari.apache.org; Secure; HttpOnly
Cache-Control: no-cache
Expires: Fri, 24 Feb 2017 14:19:25 GMT
Date: Fri, 24 Feb 2017 14:19:25 GMT
Pragma: no-cache
Expires: Fri, 24 Feb 2017 14:19:25 GMT
Date: Fri, 24 Feb 2017 14:19:25 GMT
Pragma: no-cache
Content-Type: application/json; charset=UTF-8
X-FRAME-OPTIONS: SAMEORIGIN
Server: Jetty(6.1.26.hwx)
Content-Length: 276
 
{"FileStatuses":{"FileStatus":[{"accessTime":0,"blockSize":0,"childrenNum":1,"fileId":16398,"group":"hdfs","length":0,"modificationTime":1487855904191,"owner":"hdfs","pathSuffix":"entity-file-history","permission":"755","replication":0,"storagePolicy":0,"type":"DIRECTORY"}]}}
```
服务器首先返回401(Authentication required)，并在响应头上放置了```WWW-Authenticate: Negotiate```标志。这个Negotiate表示要挑战浏览器（收到这个挑战后，在windows的chrome表现为弹出让输入用户名口令的窗口）。而在linux下的curl的则调用了自己的GSS-Negotiate功能来计算出一个请求来再次调用同一个URL，如果在curl中加入--verbose参数可以看到请求增加的内容：
```
> GET /webhdfs/v1/tmp?op=LISTSTATUS HTTP/1.1
> Authorization: Negotiate YIIC4QYJKoZIhvcSAQICAQBuggLQMIICzKADAgEFoQMCAQ6iBwMFACAAAACjggGcYYIBmDCCAZSgAwIBBaETGxFBTUJBUkkuQVBBQ0hFLk9SR6IqMCigAwIBA6EhMB8bBEhUVFAbF3UxNDAxLmFtYmFyaS5hcGFjaGUub3Jno4IBSjCCAUagAwIBEKEDAgEBooIBOASCATQsvtJkb821p1278/N+uJkQLdSHxwFjikXItktYEUizskJu5l4BkQhs/SPF0Zq0QrSxzMMB9x7WO90w1edyM8lv5oaMNPs7nzTOIYDW52K47NdIY/TwDScNlFubWAg/Aq5b9wxrLrJ7r+G7J9DheYLUTglgztIV0jqFlWFgxr6ZGtkGHx4QdMDeGQDmcdeGEQWlNYbrkO1D5iMCLWySZe3ijBZ77DU22F5ukS6BiCAVbAVouRPCTe22ey19kknygqvsc8TcVS7/5kKcNL1CbZsiJUxwaEeDRNjEFCmRp4and0hO3l2iAc0P+hsCuTBz2oEtQaUV4tCgJ78V7sFRHcT+un/nfLd7P246odeOFOR/e4KOBMH56+WpyabbbH4cZJ3wFcB4dldkj0BKZftIG5pRV2v2g6SCARUwggERoAMCARCiggEIBIIBBFbDBLfvzjDA97j0RqFkxWG7GP97uDOSO3sJ5sgdajePDG/nvP+TBMelraUiJFPe/MZX9aInDyrod4RTTZ4oqoYyyrYte2wuczSuuzcJd1tpzDkSq64EtC9ZU/Ir9Ix0OrFtgmL+Cq4YbrB3FRtzq59xRaiPzDAnwAwG57f6lrBOKpwCLflOiNj7cKiDuDhczSM3p+G0ZE1o9VkDhP5RkXBD8+vSrAZtqM6HrmJ+BukmpNvPMDXGN9eijgfMDC2g1IY2afViC8a2ZiuLVvKnBc8yn+bkwXxejJXMEzz9t0z/qWCF8RPrsRnRHsT87wRuqXrdzT8Awsb6R0qHJhmbfduRyd3C
> User-Agent: curl/7.35.0
```

上述请求还可以这样发送：
```
$ curl -c c.txt -k -i --negotiate -u : "https://u1401.ambari.apache.org:8443/gateway/default/webhdfs/v1/tmp?op=LISTSTATUS" 
$  curl -b c.txt -k -i -H 'WWW-Authenticate: Negotiate YGwGCSqGSIb3EgECAgIAb10wW6ADAgEFoQMCAQ+iTzBNoAMCARCiRgREyTPm8S3VhWLJgckAfCtBtaF4ppYYN+LXFDVA4bu9q/zQo1MXAo2A2OaIoaOKVNil3NGXh/oIYcalWb6baxsSeF8giT8=' "https://u1401.ambari.apache.org:8443/gateway/default/webhdfs/v1/tmp?op=LISTSTATUS"
```
`curl -c c.txt`参数表示把http响应中的Set-Cookie字段写入了本地的c.txt文件。`curl -b c.txt`表示带着c.txt文件中的cookie内容发送请求，并利用`-H`参数设置了令牌。  
