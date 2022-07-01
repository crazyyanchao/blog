---
title: Cypher加载CSV文件
tags: [Cypher]
author: Yc-Ma
show_author_profile: true
key: 2022-07-01-Cypher加载CSV文件
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

## 加载CSV文件
- 读取为数据列表
```
//LOAD CSV FROM 'file:/artists.csv' AS line
LOAD CSV FROM "http://data.neo4j.com/examples/person.csv" AS line
RETURN *
```

- 读取为数据对象
```
//LOAD CSV WITH HEADERS FROM 'file:/artists.csv' AS line
LOAD CSV WITH HEADERS FROM "http://data.neo4j.com/examples/person.csv" AS line
RETURN *
```

- Broswer批量提交
```
:auto USING PERIODIC COMMIT 1000
//LOAD CSV FROM 'file:/artists.csv' AS line
LOAD CSV FROM "http://data.neo4j.com/examples/person.csv" AS line
CREATE (:Artist {name: line[1], year: toInteger(line[2])});
```

- 读取压缩文件
```
LOAD CSV FROM "http://10.10.39.162:8000/neo-import-csv/test.csv" AS line
RETURN *
```

- 读取MinIO的CSV数据
```
LOAD CSV WITH HEADERS FROM 'http://10.20.0.157:31391/mlpipeline/confusion_matrix.csv?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=minio%2F20220701%2F%2Fs3%2Faws4_request&X-Amz-Date=20220701T053641Z&X-Amz-Expires=432000&X-Amz-SignedHeaders=host&X-Amz-Signature=e2e9cf98c9f4c92300d33c4cf8c93c347fc5e09e8b978e4a267c39fab2b681b6' AS line
RETURN *
```

