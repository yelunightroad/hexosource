---
title: mysql与postgresql的区别
date: 2017-11-14 23:11:19
tags: [mysql,postgresql,greenplum]
categories:
- sql相关
---

在工作中由于数据量太大时mysql负载的压力，我们引入了greenplum做我们大数据量表格的数据库，选择greenplum的原因是它在解决大数据存储的同时，尽可能的保持了和mysql一样的操作逻辑（同是关系型数据库，使用sql查询），减小了我们的迁移成本，但是尽管如此，greenplum基于的postgresql引擎和mysql还是存在一定的差异，此文对目前遇到的问题进行了记录，以防止以后再次出现同样的问题。

1. 反引号的区别

   在mysql中，对于保留字使用反引号来加以区分，例如想查询一个表中的select字段，由于select是保留字我们需写成

   ``SELECT `select` FROM table ``

   而在postgresql中不存在这种写法，当identifier为保留字时，则使用双引号

   `SELECT "select" FROM table`

2. limit的差别

   在mysql中limit可以间简写为`limit 10,100`，其等价于`limit 100 offset 10`而在postgresql中只能采用第二种标准的sql方式

3. 对if语句的支持

   mysql支持if 语句，例如`if(x>1,1,0)`而postgresql只支持case when语句

4. 大小写

   mysql中是不区分大小写的，你查询一个字段，写大小写都可以，而在postgresql中是区分大小写的，如果想要字段名大些，必须用双引号将其扩起，由于一些历史原因mysql中使用了大写的字段名，查询也没有问题，但是到postgresql中就失败了，所以建议在sql中，采用下划线分割，不要使用驼峰命名发

以上就是目前遇到的mysql迁移greenplum过程中遇到的一些问题，可以看到大部分是因为mysql在标准sql的基础上给我们提供了更加方便的解决方式，但这种方式在迁移数据库时会给我们带来意想不到的麻烦，所以除非肯定只用一种数据库，否则还是建议按照标准sql书写