---
title: 自动化生成CYPHER-TASK脚本
tags: [ONgDB,Cypher,SQL,MySQL]
author: Yc-Ma
show_author_profile: true
key: 2021-01-15-自动化生成CYPHER-TASK脚本
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

## BUILD-TASK脚本命名规范
### 异构图
- 数据模型
```
(HBondOrg)-[:发行证券]->(HEventBond)
```
- 任务文件夹名称
```
graph-sync-check-point/event/HBondOrg-F-发行证券-T-HEventBond/
```
- 开始节点任务名称
```
node-F-HBondOrg.cql
```
- 结束节点任务名称
```
node-T-HEventBond.cql
```
- 关系任务名称
```
rel-发行证券.cql
```

### 同构图
- 数据模型
```
(HORGGuaranteeV003)-[:担保]->(HORGGuaranteeV003)
```
- 任务文件夹名称
```
graph-sync-check-point/guarantee/HORGGuaranteeV003-F-担保-T-HORGGuaranteeV003/
```
- 节点任务名称
```
node-HORGGuaranteeV003.cql
```
- 关系任务名称
```
rel-担保.cql
```

## 异构图过程设计
### 异构图-TASK脚本生成
- 使用CALL apoc.custom.asProcedure生成下面三个过程
```
CALL custom.task.build.graph.from({merge_label},{merge_field},{child_labels},{properties},{jdbc_etl_url},{etl_sql},{jdbc_task_url},{task_hcode},{TASK_CHECK_POINT_TABLE},{TASK_CHECK_POINT_LOCK_TABLE}) -> CALL custom.task.node.node-F-HBondOrg.HBondOrg-F-发行证券-T-HEventBond()
CALL custom.task.build.graph.to({merge_label},{merge_field},{child_labels},{properties},{jdbc_etl_url},{etl_sql},{jdbc_task_url},{task_hcode},{TASK_CHECK_POINT_TABLE},{TASK_CHECK_POINT_LOCK_TABLE}) -> CALL custom.task.node.node-T-HEventBond.HBondOrg-F-发行证券-T-HEventBond()
CALL custom.task.build.graph.rel({matchFrom_label},{matchFrom_field},{matchTo_label},{matchTo_field},{properties},{jdbc_etl_url},{etl_sql},{jdbc_task_url},{task_hcode},{TASK_CHECK_POINT_TABLE},{TASK_CHECK_POINT_LOCK_TABLE}) -> CALL custom.task.node.rel-发行证券.HBondOrg-F-发行证券-T-HEventBond()
```
- 以一个异构图为例：(HBondOrg)-[:发行证券]->(HEventBond)
- 使用CALL custom.task.build.graph.*()生成下面三个过程
- 节点过程
```
CALL custom.task.node.node_F_HBondOrg.HGRAPHTASK_HBondOrg_发行证券_HEventBond_()
CALL custom.task.node.node_T_HEventBond.HGRAPHTASK_HBondOrg_发行证券_HEventBond_()
```
- 关系过程
```
CALL custom.task.node.rel_发行证券.HGRAPHTASK_HBondOrg_发行证券_HEventBond_()
```

### 生成custom.task.build.graph.*过程
- 生成task.build.graph.from过程【构建FROM-CYPHER-TASK脚本的过程】
```
@param {merge_label}-合并节点的数据模型标签
@param {merge_field}-合并节点的字段
@param {child_labels}-JsonArrayString数组,空数组传入[]
@param {jdbc_etl_url}-支持MySQL、Oracle、SqlServer
@param {etl_sql}-必须包含hcode字段
@param {jdbc_task_url}-任务状态表状态锁表位置默认只支持MySQL
@param {task_hcode}-HGRAPHTASK(HBondOrg)-[发行证券]->(HEventBond)
@param {etl_max_min}-从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间
CALL apoc.custom.asProcedure(
  'task.build.graph.from',
  'WITH $merge_label AS merge_label,$merge_field AS merge_field,$child_labels AS child_labels,$jdbc_etl_url AS jdbc_etl_url,$etl_sql AS etl_sql,$jdbc_task_url AS jdbc_task_url,$task_hcode AS task_hcode,$etl_max_min AS etl_max_min,\'CALL apoc.load.jdbcUpdate(\\\'{jdbc_task_url}\\\',\\\'UPDATE ONGDB_TASK_CHECK_POINT_LOCK updateLock INNER JOIN (SELECT task_lock,hcode FROM ONGDB_TASK_CHECK_POINT_LOCK selectLock WHERE task_lock=0 AND hcode=?) selectLock ON updateLock.hcode=selectLock.hcode SET updateLock.task_lock=1\\\',[\\\'{task_hcode}\\\']) YIELD row AS lock WHERE lock.count>0 WITH lock CALL apoc.load.jdbc(\\\'{jdbc_task_url}\\\',\\\'SELECT DATE_FORMAT(node_check_point,\\\\\\\'%Y-%m-%d %H:%i:%s\\\\\\\') AS check_point,DATE_FORMAT(NOW(),\\\\\\\'%Y-%m-%d %H:%i:%s\\\\\\\') AS currentTime FROM ONGDB_TASK_CHECK_POINT WHERE hcode=?\\\',[\\\'{task_hcode}\\\']) YIELD row WITH apoc.text.join([\\\'\\\\\\\'\\\',row.check_point,\\\'\\\\\\\'\\\'], \\\'\\\') AS check_point,row.currentTime AS currentTime,row.check_point AS rawCheckPoint CALL apoc.load.jdbc(\\\'{jdbc_etl_url}\\\',\\\'{etl_max_min}\\\',[rawCheckPoint]) YIELD row WITH row.min AS min,row.max AS max,check_point,currentTime,rawCheckPoint WITH apoc.coll.union(olab.ids.batch(min,max,10000),[[0,1]]) AS value,check_point,currentTime,rawCheckPoint UNWIND value AS bactIdList WITH bactIdList[0] AS batchMin,bactIdList[1] AS batchMax,check_point,currentTime,rawCheckPoint WITH REPLACE(\\\'CALL apoc.load.jdbc(\\\\\\\'{jdbc_etl_url}\\\\\\\', \\\\\\\'{etl_sql}\\\\\\\',[check_point,batchMin,batchMax])\\\',\\\'check_point,batchMin,batchMax\\\',check_point+\\\',\\\'+batchMin+\\\',\\\'+batchMax) AS sqlData,currentTime,rawCheckPoint CALL apoc.periodic.iterate(sqlData,\\\'MERGE (n:{merge_label} {{merge_field}:row.hcode}) SET n+=row WITH n,row CALL apoc.create.addLabels(n,apoc.coll.union(apoc.convert.fromJsonList(row.label),{child_labels})) YIELD node RETURN node\\\', {parallel:false,batchSize:10000,iterateList:true}) YIELD batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations WITH batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations,currentTime,rawCheckPoint WITH SUM(batch.failed) AS batchFailedSize,currentTime,rawCheckPoint CALL apoc.do.case([batchFailedSize<1,\\\'CALL apoc.load.jdbcUpdate(\\\\\\\'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC\\\\\\\',\\\\\\\'UPDATE ONGDB_TASK_CHECK_POINT SET node_check_point=?,rel_check_point=? WHERE hcode=?\\\\\\\',[$currentTime,$rawCheckPoint,\\\\\\\'{task_hcode}\\\\\\\']) YIELD row RETURN row;\\\'],\\\'\\\',{currentTime:currentTime,rawCheckPoint:rawCheckPoint}) YIELD value WITH value,batchFailedSize,currentTime,rawCheckPoint CALL apoc.load.jdbcUpdate(\\\'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC\\\',\\\'UPDATE ONGDB_TASK_CHECK_POINT_LOCK SET task_lock=0 WHERE hcode=?\\\',[\\\'{task_hcode}\\\']) YIELD row AS releaseLock RETURN {releaseLock:releaseLock,value:value,batchFailedSize:batchFailedSize,currentTime:currentTime,rawCheckPoint:rawCheckPoint} AS result;\' AS cypher_task WITH \'task.node.\'+REPLACE(\'node-F-\'+apoc.text.regexGroups(task_hcode,\'((?!=,)([A-Za-z0-9_\u4e00-\u9fa5]+))+\')[1][1],\'-\',\'_\')+\'.\'+REDUCE(hcode=\'\',hcodeStrs IN apoc.text.regexGroups(task_hcode,\'((?!=,)([A-Za-z0-9_\u4e00-\u9fa5]+))+\') | hcode+hcodeStrs[0]+\'_\') AS nodeTaskName,olab.replace(cypher_task,[{raw:\'{merge_label}\',rep:merge_label},{raw:\'{merge_field}\',rep:merge_field},{raw:\'{child_labels}\',rep:child_labels},{raw:\'{jdbc_etl_url}\',rep:jdbc_etl_url},{raw:\'{etl_sql}\',rep:etl_sql},{raw:\'{jdbc_task_url}\',rep:jdbc_task_url},{raw:\'{task_hcode}\',rep:task_hcode},{raw:\'{etl_max_min}\',rep:etl_max_min}]) AS cypher_task CALL apoc.custom.asProcedure(nodeTaskName,cypher_task,\'WRITE\',[[\'result\',\'MAP\']],[],\'构建FROM-NODE-TASK：{merge_label}-合并节点的数据模型标签;{merge_field}-合并节点的字段;{child_labels}-JsonArrayString数组,空数组传入[] ;{jdbc_etl_url}-支持MySQL、Oracle、SqlServer ;{etl_sql}-必须包含hcode字段 ;{jdbc_task_url}-任务状态表状态锁表位置默认只支持MySQL ;{task_hcode}-HGRAPHTASK(HBondOrg)-[发行证券]->(HEventBond);{etl_max_min}-从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间\') RETURN \'CALL custom.\'+nodeTaskName AS procedure_name,cypher_task',
  'WRITE',
  [['procedure_name','STRING'],['cypher_task','STRING']],
  [['merge_label','STRING'],['merge_field','STRING'],['child_labels','STRING'],['jdbc_etl_url','STRING'],['etl_sql','STRING'],['jdbc_task_url','STRING'],['task_hcode','STRING'],['etl_max_min','STRING']],
  '构建FROM-NODE-TASK：{merge_label}-合并节点的数据模型标签;{merge_field}-合并节点的字段;{child_labels}-JsonArrayString数组,空数组传入[] ;{jdbc_etl_url}-支持MySQL、Oracle、SqlServer ;{etl_sql}-必须包含hcode字段 ;{jdbc_task_url}-任务状态表状态锁表位置默认只支持MySQL ;{task_hcode}-HGRAPHTASK(FromLabel)-[RelaName]->(ToLabel);{etl_max_min}-从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间'
);
```
- 生成task.build.graph.to过程【构建TO-CYPHER-TASK脚本的过程】
```
CALL apoc.custom.asProcedure(
  'task.build.graph.to',
  '',
  'WRITE',
  [['scope','STRING'],['hcode','STRING']],
  [['name','STRING']],
  '构建TO-NODE-TASK'
);
```
- 生成task.build.graph.rel过程【构建REL-CYPHER-TASK脚本的过程】
```
CALL apoc.custom.asProcedure(
  'task.build.graph.rel',
  '',
  'WRITE',
  [['scope','STRING'],['hcode','STRING']],
  [['name','STRING']],
  '构建REL-TASK'
);
```
- 移除过程
```
CALL apoc.custom.removeProcedure('task.build.graph.from')
```

### 节点CYPHER-TASK生成
- FROM-CYPHER-TASK生成
```
@param {merge_label}  合并节点的数据模型标签 HBondOrg
@param {merge_field}  合并节点的字段 hcode
@param {child_labels} JsonArrayString数组,空数组传入[] apoc.coll.union(apoc.convert.fromJsonList(row.label),{child_labels})
@param {jdbc_etl_url} 支持MySQL、Oracle、SqlServer jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC
@param {etl_sql} 必须包含hcode字段 SELECT org_hcode AS hcode,org_name AS name,credit_code,label,CONVERT(DATE_FORMAT(hcreatetime,\\\'%Y%m%d%H%i%S\\\'),UNSIGNED INTEGER) AS hcreatetime,CONVERT(DATE_FORMAT(hupdatetime,\\\'%Y%m%d%H%i%S\\\'),UNSIGNED INTEGER) AS hupdatetime,hisvalid,create_by,update_by FROM HBondOrg WHERE hupdatetime>=? AND huid>=? AND huid<=? AND type=\\\'发行证券\\\'
@param {jdbc_task_url} 任务状态表状态锁表位置默认只支持MySQL jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC
@param {task_hcode} HGRAPHTASK(HBondOrg)-[发行证券]->(HEventBond)
@param {etl_max_min} 从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间 SELECT MIN(huid) AS min,MAX(huid) AS max FROM HBondOrg WHERE hupdatetime>=? AND type=\\\'发行证券\\\'
// {etl_sql}:使用七个转义符号
// {etl_max_min}:使用三个转义符号
WITH 'HBondOrg' AS merge_label,'hcode' AS merge_field,NULL AS child_labels,'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC' AS jdbc_etl_url,'SELECT org_hcode AS hcode,org_name AS name,credit_code,label,CONVERT(DATE_FORMAT(hcreatetime,\\\\\\\'%Y%m%d%H%i%S\\\\\\\'),UNSIGNED INTEGER) AS hcreatetime,CONVERT(DATE_FORMAT(hupdatetime,\\\\\\\'%Y%m%d%H%i%S\\\\\\\'),UNSIGNED INTEGER) AS hupdatetime,hisvalid,create_by,update_by FROM HBondOrg WHERE hupdatetime>=? AND huid>=? AND huid<=? AND type=\\\\\\\'发行证券\\\\\\\'' AS etl_sql,'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC' AS jdbc_task_url,'HGRAPHTASK(HBondOrg)-[发行证券]->(HEventBond)' AS task_hcode,'SELECT MIN(huid) AS min,MAX(huid) AS max FROM HBondOrg WHERE hupdatetime>=? AND type=\\\'发行证券\\\'' AS etl_max_min
CALL custom.task.build.graph.from(merge_label,merge_field,child_labels,jdbc_etl_url,etl_sql,jdbc_task_url,task_hcode,etl_max_min) YIELD procedure_name,cypher_task RETURN procedure_name,cypher_task
```
- TO-CYPHER-TASK生成
```
```

### 关系CYPHER-TASK自动生成
```

```

## 同构图过程设计
### 同构图-TASK脚本生成
- 使用CALL apoc.custom.asProcedure生成下面两个过程
```
CALL custom.task.build.graph.fromWithTo()
CALL custom.task.build.graph.rel()
```
- 以一个异构图为例：(HORGGuaranteeV003)-[:担保]->(HORGGuaranteeV003)
- 使用CALL custom.task.build.graph.*()生成下面两个过程
- 节点过程
```
CALL custom.task.node.node_HORGGuaranteeV003.HGRAPHTASK_HORGGuaranteeV003_担保_HORGGuaranteeV003_()
```
- 关系过程
```
CALL custom.task.node.rel_担保.HGRAPHTASK_HORGGuaranteeV003_担保_HORGGuaranteeV003_()
```

