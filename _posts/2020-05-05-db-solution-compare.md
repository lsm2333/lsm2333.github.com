---
layout: post
category: 数据库
title: 数据库小结
tagline: by Snail
tags: 
  - MySQL
  - 数据库
published: true
date: 2020-05-05 15:58:09 +08:00
---	
趁着五一假期，来回顾下项目中使用到的DB方案吧～
<!--more-->

过去一年，经历过项目技术迁移（从Python+Mongo到Java+Mysql），也在不同的项目中接触了一些数据库方案（MySql，TiDB，MyCat，MongoDB，Elastic Search等）。接下来，从业务场景，各自优劣来做一些个人理解的阐述。

##Mysql vs MongoDB：
去年底到今年过年复工期间，都集中在项目技术迁移（从Python+Mongo到Java+Mysql），下面就从MongoDB和MySql的两个角度来简单总结下。

###MongoDB角度
1. mongo的free-schema，自由的数据格式，以及对json的良好支持，很适合在早期开发中应用。能够让开发者专注于业务迭代。相比之下，mysql需要花费一定时间在表设计上。
2. mongo的free-schema也带来了不便，在后期开发、重构中发现，数据存在多份schema，导致迁移需要考虑多套方案。同时还存在数据重复等数据问题。
3. mongo对于表关联的支持不好，需要先查出一个表的数据，在拿到另一个表中查询。相比之下，mysql这方面更让开发者省心。
4. json支持真的很舒心，先前版本的mysql不支持json，相比之下mongo可以把关联数据都放在一块，用json嵌套起来。存报文的时候尤其方便（正好也是格式常变的，并且多是json）

###MySql角度
1. 语法/接口丰富，而且比较成熟，更加符合团队中开发者的习惯和技能。
2. 对事务的支持，保证了数据的准确性。在重构的时候发现，mongo的某个业务场景下的数据链路不能自圆其说，无法展现完整的数据链，推测是因为当时的mongo版本没有开启事务，导致数据的不准确。
3. 维护ddl需要时间开销。在维护ddl方面需要花费一定时间和精力，设计表格、字段、类型等等。


##选择TiDB的考虑因素
除了Mongo和MySql之外，在项目中还接触了TiDB，接下来做一些个人的理解阐述。

###MySQL的问题
- 随着数据量增大，查询耗时明显增加
- 未来数据量预计达到亿级，需要为未来数据增长带来的不利影响作准备
- 如果简单地做历史数据删除，则在查询时间跨度上有限制

###分库分表的问题（拿mycat举例）
- 在mycat非分片字段查询时，mycat支持较差，会花费较大查询成本。
- 分布式事务方面，tidb可以提供100%数据强一致性，而mycat只保证prepare阶段数据一致性的弱XA事务，不能完全保证数据一致性。
- 维护成本高。数据量持续增大时，tidb只需简单地增加新节点就可以完成水平扩容。而mycat需要重新制定分片规则（不论是基于时间范围，或是hash），扩容时需要停止数据新增服务，并备份数据库数据。

###MongoDB等NoSql的问题
- 语法天差地别，代码改动较大。
- join等跨collection操作支持不好。

###TiDB的好处
- 兼容MySQL语法，迁移时Java端代码改动小。
- 查询速度快，并且在数据量巨大的情况下查询时间也能维持在可接受范围。基于网络资料，数据量越大，这一优势对比MySq更明显。
- TiDB支持自动Sharding，业务段不需要切表操作。也不需要像使用中间件时自己指定sharding key。底层存储会自动将数据均匀分散在集群中，存储空间和性能可以通过增加机器快速实现，降低运维成本。


参考：
- [日均数据量千万级，MySQL、TiDB 两种存储方案的落地对比](https://segmentfault.com/a/1190000008644583?utm_source=tag-newest)
- [tidb与mysql_mycat对比](https://blog.csdn.net/tiandao321/article/details/85760275)