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

