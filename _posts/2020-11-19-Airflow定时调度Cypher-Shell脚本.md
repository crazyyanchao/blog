---
title: Airflow定时调度Cypher-Shell脚本
tags: [Airflow,Cypher]
author: Yc-Ma
show_author_profile: true
key: 2020-11-19-Airflow定时调度Cypher-Shell脚本
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

### Cron设置
```
// 1 1 * * * 每天一点
// 1 7 * * * 每天七点
// */1 * * * * 每分钟
// */30 * * * * 每三十分钟
// 0 */12 * * *  每隔12个小时
```
### 执行Cypher脚本每天同步更新数据
每天同步前一天到现在的数据，暂不支持历史数据，需要手动跑
```
#!/bin/bash
DATE=$(date -d "1 day ago" "+%Y-%m-%d %H:%M:%S")
CYPHER="CALL apoc.periodic.iterate('CALL apoc.load.jdbc(\'jdbc:mysql://testlab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.alibaba.com.cn:3306/analytics_graph_data?user=dev&password=testlabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC\', \'SELECT hcode,name,data_source,CONVERT(DATE_FORMAT(hcreatetime,\\\\\\\\\'%Y%m%d%H%i%S\\\\\\\\\'),UNSIGNED INTEGER) AS hcreatetime,CONVERT(DATE_FORMAT(hupdatetime,\\\\\\\\\'%Y%m%d%H%i%S\\\\\\\\\'),UNSIGNED INTEGER) AS hupdatetime,hisvalid,create_by,update_by FROM HCCGPBidOrg WHERE hupdatetime>=\\\\\\\\\'${DATE}\\\\\\\\\'\')','MERGE (n:HCCGPBidOrg {hcode:row.hcode}) SET n+=row,n:HORG', {parallel:false,batchSize:1000}) YIELD batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations RETURN batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations;"

su - ongdb <<EOF
testlab%pro

echo "MySQL TO ONgDB: Node"
echo "${CYPHER}"

/home/ongdb/ongdb-enterprise-3.5.17/bin/cypher-shell -a bolt+routing://10.20.12.173:7687 -u neo4j -p testlab%pro --format plain "${CYPHER}"

EOF
```
### 执行Cypher脚本每天同步更新数据【执行CYPHER文件】
每天同步前一天到现在的数据，暂不支持历史数据，需要手动跑
```
#!/bin/bash
su - ongdb <<EOF
testlab%pro

echo "MySQL TO ONgDB: Node"

cat /home/ongdb/ongdb-enterprise-3.5.17/graph-sync/test.cql | /home/ongdb/ongdb-enterprise-3.5.17/bin/cypher-shell -a bolt+routing://10.20.12.173:7687 -u neo4j -p testlab%pro

EOF
```
```
#!/bin/bash
su - ongdb <<EOF
testlab%pro

echo "MySQL TO ONgDB: Relationship"

cat /home/ongdb/ongdb-enterprise-3.5.17/graph-sync/test.cql | /home/ongdb/ongdb-enterprise-3.5.17/bin/cypher-shell -a bolt+routing://10.20.12.173:7687 -u neo4j -p testlab%pro

EOF
```

### test.cql脚本内容【每次运行处理一天前的数据】
```
CALL apoc.periodic.iterate('CALL apoc.load.jdbc(\'jdbc:mysql://testlab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.alibaba.com.cn:3306/data?user=dev&password=data&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC\', \'SELECT hcode,name,credit_code,label,CONVERT(DATE_FORMAT(hcreatetime,\\\'%Y%m%d%H%i%S\\\'),UNSIGNED INTEGER) AS hcreatetime,CONVERT(DATE_FORMAT(hupdatetime,\\\'%Y%m%d%H%i%S\\\'),UNSIGNED INTEGER) AS hupdatetime,hisvalid,create_by,update_by FROM HORGGuaranteeV002 WHERE hupdatetime>=DATE_SUB(NOW(),INTERVAL 1 DAY)\')',
'MERGE (n:HORGGuaranteeV002 {hcode:row.hcode}) SET n+=row WITH n,row CALL apoc.create.addLabels(n,apoc.convert.fromJsonList(row.label)) YIELD node RETURN node', {parallel:false,batchSize:1000}) YIELD batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations
RETURN batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations;
```

### 调度超大图数据的TASK实现方案【单个TASK的实现方案】
- 1、获取上一个checkpoint time【存储在MySQL中】【(from)-[r]->(to):变量from/r/to都需要存储】【节点开始结束标签和关系类型作为关系的唯一标准】
- 2、【(from)-[r]->(to):执行CYPHER任务】【将第一个执行的构建节点的CYPHER的运行结束时的系统时间保存到MySQL作为下一个checkpoint time】

