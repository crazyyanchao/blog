---
title: ONgDB使用apoc提交后台cypher任务
tags: [ONgDB,Apoc]
author: Yc-Ma
show_author_profile: true
key: 2020-06-16-ONgDB使用apoc提交后台cypher任务
---

Here's the table of contents:
1. TOC
{:toc}

## apoc服务器扩展
[Neo4j图数据库高级应用系列 / 服务器扩展指南 APOC(4.1) - 查询任务管理](https://blog.csdn.net/GraphWay/article/details/93711152)

## 提交查询任务
```
CALL apoc.periodic.submit('writeTest','MATCH (n) with collect(n) as n
CALL apoc.algo.pageRank(n) YIELD node,score SET node.score=score')
```

## 查看所有后台运行的查询任务
```
CALL apoc.periodic.list()
```
## 取消任务
```
CALL apoc.periodic.cancel
```



