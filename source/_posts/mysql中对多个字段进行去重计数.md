---
title: mysql中对多个字段进行去重计数
date: 2017-11-30 18:45:45
tags: [mysql,count distinct]
categories:
- sql相关
---

今天遇到了一个奇怪的bug，举例如下，假设有一张表table

| colA | colB | colC |
| ---- | ---- | ---- |
| a1   | b1   | NULL |
| a2   | b2   | NULL |
| a3   | b3   | NULL |

当我们使用`SELECT COUNT(DISTINCT colA,colB,colC) FROM talbe`进行查询时，返回的竟然是0。但是我们使用`SELECT DISTINCT colA,colB,colC FROM talbe`可以正常获取三行数据，后来在[Stack Overflow](https://stackoverflow.com/questions/18968963/select-countdistinct-error-on-multiple-columns)上看到SQL_Server中count只支持三种格式

```
COUNT(*)
COUNT(colName)
COUNT(DISTINCT colName)
```

而查询mysql相关文档没看到这种说法，官方文档中只有count(*)的示例，所以只能认为也是相同的，最终将查询代码改为

```
SELECT COUNT(*) FROM (SELECT DISTINCT colA,colB,colC FROM talbe) A
```

成功的解决了问题，所以在多维度去重的情况下要特别注意，使用正确的语法。