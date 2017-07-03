---
layout: post
title:  Hive新的安全特性:行过滤Row Filter和列屏蔽Column Masking
date:   2017-07-01
categories: base
tag: base hive Filter Masking security cell ranger  
---
  
[原文Row Filter and Column Masking](https://community.hortonworks.com/articles/58255/row-filter-and-column-masking.html)  
  
简介
------------------------------------
本文介绍了一个新的特性，
介绍了两种新的Ranger安全策略类型：Hive的行过滤```Row Filter``` 和 列屏蔽```Column Masking```。

Article
------------------------------------
本文使用```HDP2.5```版本 

#### 背景:
在这个特性发布之前，Ranger仅允许用户为hive创建直到列级别的访问策略，这里为大家提供了一个启用单元级Cell安全策略的解决方案。

#### 介绍:
这是一个新引入的特性,允许创建一种新的ranger-hive策略。
帮助管理员根据策略中的过滤条件限制用户访问表中的某些特定行，或者全部的/部分的屏蔽某些敏感数据。

下面详细讨论这两种策略类型:

#### 行过滤策略:
行过滤策略允许ranger指定过滤表达式，这样用户只能看到那个属于他自己表上的一些特定的行。
例如：如果用户属于美国，并且查询employee表，我们希望限制他只看到属于美国的员工，那么筛选表达式将是location = 'US'。你可以在HDP2.5 在Hive策略页有一个新的tab页-"Row filter"
我们需要在页面配置项中提供数据库和表名，并且输入行过滤表达式，用于过滤出部分的hive查询的结果。
过滤表达式必须是一个有效的where语句（支持内部条件）
![](https://community.hortonworks.com/storage/attachments/7954-screen-shot-2016-09-25-at-104708-am.png)
-注意: 1) 在这个策略类型中没有列的选项，因为它对列过滤没有意义。

#### 列屏蔽策略:
列屏蔽策略允许Ranger在Hive策略中指定屏蔽条件，用于针对特定用户屏蔽敏感数据。
例如：在银行里银行帐户号码和CVV是客户敏感数据，现在，你可以在Ranger中创建屏蔽策略，针对特定的用户或组隐藏部分或全部的列数据。
你可以在HDP2.5 在Hive策略页有一个新的tab页-"column masking"。
我们需要在页面配置项中提供数据库、表和列名，并在条件下选择屏蔽条件。

当前支持以下掩码条件：
1) Redact:【编辑、编写】
2) Partial Mask: show last 4【部分屏蔽:展示后4个字符】
3) partial Mask: show first 4【部分屏蔽:展示前4个字符】
4) Hash【哈希】
5) Nullify【无效】
6) Unmasked(retain original value)【保留原始值】
7) Date: show only year【时间:仅显示年份】
8) custom【自定义】
![](https://community.hortonworks.com/storage/attachments/7956-screen-shot-2016-09-25-at-110431-am.png)
-注意:

1) 策略类型中不允许使用通配符。
2) 这些策略可以在表或视图上创建。
3) 这些策略在执行查询时的顺序，以在列表中的顺序为准。

例子:
现在让我们举一个例子，尝试行过滤和每个列屏蔽技术：让我们在"Bank"数据库中有一个名为"customer"的表：
```
----+--------------------+--+
| customer.id  | customer.name  | customer.account  | customer.cvv  | customer.dob  | customer.location  |
+--------------+----------------+-------------------+---------------+---------------+--------------------+--+
| 432          | Amit           | 898981931313131   | 432           | 1975-04-01    | Delhi              |
| 493          | John           | 79898193128931    | 234           | 1985-09-11    | Bangalore          |
| 683          | nisar          | 69598193128931    | 121           | 1965-09-11    | Bangalore          |
| 532          | rohan          | 198981931313131   | 402           | 1995-04-01    | Delhi              |
| 400          | Rahul          | 69898193128931    | 159           | 1985-09-10    | Bangalore          |
| 809          | nisar          | 59598193128931    | 096           | 1979-09-11    | Bangalore          | 
```
我们创建行过滤和列屏蔽策略，然后执行"select * from customer;" ，我们会看到不同的结果集。

1) 行策略的例子: 
	针对用户"user1",使用过滤表达式创建一个新的行过滤策略: location = 'Bangalore'
![](https://community.hortonworks.com/storage/attachments/7955-screen-shot-2016-09-25-at-105506-am.png)

a) 用户"user1"执行 "select * from customer;" 得到如下结果:
```
+--------------+----------------+-------------------+---------------+---------------+--------------------+--+
| customer.id  | customer.name  | customer.account  | customer.cvv  | customer.dob  | customer.location  |
+--------------+----------------+-------------------+---------------+---------------+--------------------+--+
| 493          | John           | 79898193128931    | 234           | 1985-09-11    | Bangalore          |
| 683          | nisar          | 69598193128931    | 121           | 1965-09-11    | Bangalore          |
| 400          | Rahul          | 69898193128931    | 159           | 1985-09-10    | Bangalore          |
| 809          | nisar          | 59598193128931    | 096           | 1979-09-11    | Bangalore          |
+--------------+----------------+-------------------+---------------+---------------+--------------------+--+
4 rows selected (0.864 seconds)
```
b)如果用户"user2"执行 "select * from customer;" 因为"user2"不是策略的一部分，因此他可以看到所有的结果：
```
+--------------+----------------+-------------------+---------------+---------------+--------------------+--+
| customer.id  | customer.name  | customer.account  | customer.cvv  | customer.dob  | customer.location  |
+--------------+----------------+-------------------+---------------+---------------+--------------------+--+
| 432          | Amit           | 898981931313131   | 432           | 1975-04-01    | Delhi              |
| 493          | John           | 79898193128931    | 234           | 1985-09-11    | Bangalore          |
| 683          | nisar          | 69598193128931    | 121           | 1965-09-11    | Bangalore          |
| 532          | rohan          | 198981931313131   | 402           | 1995-04-01    | Delhi              |
| 400          | Rahul          | 69898193128931    | 159           | 1985-09-10    | Bangalore          |
| 809          | nisar          | 59598193128931    | 096           | 1979-09-11    | Bangalore          |
+--------------+----------------+-------------------+---------------+---------------+--------------------+--+
```
2)列屏蔽策略的例子: 
	创建一个列屏蔽策略，
	针对用户"user1",在customer表上创建一个具有屏蔽条件的列帐户策略: "Partal mask:show last 4"
create a column masking policy on table customer, column account with masking condition: location = 'Bangalore' for user1
![](https://community.hortonworks.com/storage/attachments/7957-screen-shot-2016-09-25-at-110441-am.png)

a) 用户"user1"执行 "select * from customer;" ,因为用户"user1"为满足列屏蔽条件，所以仅展示account的最后4个数字:
```
+--------------+----------------+-------------------+---------------+---------------+--------------------+--+
| customer.id  | customer.name  | customer.account  | customer.cvv  | customer.dob  | customer.location  |
+--------------+----------------+-------------------+---------------+---------------+--------------------+--+
| 432          | Amit           | xxxxxxxxxxx3131   | 432           | 1975-04-01    | Delhi              |
| 493          | John           | xxxxxxxxxx8931    | 234           | 1985-09-11    | Bangalore          |
| 683          | nisar          | xxxxxxxxxx8931    | 121           | 1965-09-11    | Bangalore          |
| 532          | rohan          | xxxxxxxxxxx3131   | 402           | 1995-04-01    | Delhi              |
| 400          | Rahul          | xxxxxxxxxx8931    | 159           | 1985-09-10    | Bangalore          |
| 809          | nisar          | xxxxxxxxxx8931    | 096           | 1979-09-11    | Bangalore          |
+--------------+----------------+-------------------+---------------+---------------+--------------------+--+
6 rows selected (0.841 seconds)
```
b) 如果用户"user2"执行 "select * from customer;" 因为"user2"不是策略的一部分，因此他可以看到未屏蔽的结果：
```
+--------------+----------------+-------------------+---------------+---------------+--------------------+--+
| customer.id  | customer.name  | customer.account  | customer.cvv  | customer.dob  | customer.location  |
+--------------+----------------+-------------------+---------------+---------------+--------------------+--+
| 432          | Amit           | 898981931313131   | 432           | 1975-04-01    | Delhi              |
| 493          | John           | 79898193128931    | 234           | 1985-09-11    | Bangalore          |
| 683          | nisar          | 69598193128931    | 121           | 1965-09-11    | Bangalore          |
| 532          | rohan          | 198981931313131   | 402           | 1995-04-01    | Delhi              |
| 400          | Rahul          | 69898193128931    | 159           | 1985-09-10    | Bangalore          |
| 809          | nisar          | 59598193128931    | 096           | 1979-09-11    | Bangalore          |
+--------------+----------------+-------------------+---------------+---------------+--------------------+--+
6 rows selected (0.649 seconds)
```
同样，我们也可以尝试其他的屏蔽策略。

下面的wiki页面列出了一些不错的用例，请参考：

[https://cwiki.apache.org/confluence/display/RANGER/Row-level+filtering+and+column-masking+using+Apache+Ranger+policies+in+Apache+Hive](https://cwiki.apache.org/confluence/display/RANGER/Row-level+filtering+and+column-masking+using+Apache+Ranger+policies+in+Apache+Hive)

请对任何问题发表意见。
