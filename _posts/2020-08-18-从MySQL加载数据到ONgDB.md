---
title: 从MySQL加载数据到ONgDB
tags: [ONgDB,MySQL]
author: Yc-Ma
show_author_profile: true
key: 2020-08-18-从MySQL加载数据到ONgDB
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

## 下载MySQL-DRIVE加载到插件列表

## 加载注册驱动
```
CALL apoc.load.driver("com.mysql.jdbc.Driver")
```

## 读取数据
```
CALL apoc.load.jdbc('jdbc:mysql://localhost:3306/database?user=user&password=password&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC', 'SELECT * FROM TABLE LIMIT 10') YIELD row
```

## 使用批次/多事务提交方式读取远程JDBC数据源并更新图库
```
CALL apoc.periodic.iterate('CALL apoc.load.jdbc('jdbc:mysql://localhost:3306/database?user=user&password=password&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC', 'SELECT * FROM TABLE LIMIT 10')','MERGE (n:HORGShareHold {hcode:row.hcode}) SET n.name=row.name', {parallel:true,batchSize:1000}) YIELD  batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations RETURN batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations;
```

