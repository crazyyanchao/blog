---
title: 自动生成时序JSON-REL-CYPHER-TASK脚本
tags: [ONgDB,Cypher,SQL,MySQL]
author: Yc-Ma
show_author_profile: true
key: 2021-02-01-自动生成时序JSON-REL-CYPHER-TASK脚本
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

### 构建节点任务
- 参数
```
@param {merge_label}  合并节点的数据模型标签 eg.HBondOrg
@param {merge_field}  合并节点的字段 eg.hcode
@param {child_labels} JsonArrayString数组,空数组传入[]
@param {jdbc_etl_url} 支持MySQL、Oracle、SqlServer
@param {etl_sql} 抽取数据的SQL
@param {jdbc_task_url} 任务状态表状态锁表位置默认只支持MySQL
@param {task_hcode} eg.HGRAPHTASK(FromLabel)-[RelaName]->(ToLabel)
@param {max_min_sql} 从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间
```
- 构建FROM节点任务的过程【同构图只构建FROM节点任务即可】
```
CALL custom.task.build.graph.from({merge_label},{merge_field},{child_labels},
{jdbc_etl_url},{etl_sql},{jdbc_task_url},{task_hcode},{max_min_sql})
YIELD procedure_name,cypher_task RETURN procedure_name,cypher_task
```
- 构建TO节点任务的过程
```
CALL custom.task.build.graph.to({merge_label},{merge_field},{child_labels},
{jdbc_etl_url},{etl_sql},{jdbc_task_url},{task_hcode},{max_min_sql})
YIELD procedure_name,cypher_task RETURN procedure_name,cypher_task
```
### 构建关系任务
- 参数
```
@param {matchFrom_label}-匹配FROM节点的标签
@param {matchFrom_field}-匹配FROM节点的属性字段【一般为唯一性属性字段】
@param {matchTo_label}-匹配TO节点的标签
@param {matchTo_field}-匹配TO节点的属性字段【一般为唯一性属性字段】
@param {merge_rel_type}-合并的关系类型
@param {distinct_detail_keys}-detail属性的排重机制【可定义多个字段进行排重】【{etl_sql}中定义的关系属性字段】
@param {jdbc_etl_url}-支持MySQL、Oracle、SqlServer
@param {etl_sql}-抽取数据的SQL【必须包含from和to字段】
@param {jdbc_task_url}-任务状态表状态锁表位置默认只支持MySQL
@param {task_hcode}-HGRAPHTASK(FromLabel)-[RelaName]->(ToLabel)
@param {max_min_sql}-从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间
```
- 过程
```
CALL custom.task.build.graph.rel({matchFrom_label},{matchFrom_field},
{matchTo_label},{matchTo_field},{merge_rel_type},{jdbc_etl_url}{etl_sql},
{jdbc_task_url},{task_hcode},{max_min_sql})
YIELD procedure_name,cypher_task RETURN procedure_name,cypher_task
```
### 构建关系任务
- 支持生成时序JSON属性的关系构建任务【在处理时间序列的关系时可以直接合并关系类型，并将时序属性数据存储在一个JSON属性中】
- 参数
- 过程
```
CALL custom.task.build.graph.rel.json({matchFrom_label},{matchFrom_field},{matchTo_label},{matchTo_field},{merge_rel_type},{jdbc_etl_url},{etl_sql},{jdbc_task_url},{task_hcode},{max_min_sql}) YIELD procedure_name,cypher_task RETURN procedure_name,cypher_task
```
- 生成custom.task.build.graph.rel.json过程
```
CALL apoc.custom.asProcedure(
  'task.build.graph.rel',
  'WITH $matchFrom_label AS matchFrom_label,$matchFrom_field AS matchFrom_field,$matchTo_label AS matchTo_label,$matchTo_field AS matchTo_field,$merge_rel_type AS merge_rel_type,$jdbc_etl_url AS jdbc_etl_url,$etl_sql AS etl_sql,$jdbc_task_url AS jdbc_task_url,$task_hcode AS task_hcode,$max_min_sql AS max_min_sql,\'CALL apoc.load.jdbcUpdate(\\\'{jdbc_task_url}\\\',\\\'UPDATE ONGDB_TASK_CHECK_POINT_LOCK updateLock INNER JOIN (SELECT task_lock,hcode FROM ONGDB_TASK_CHECK_POINT_LOCK selectLock WHERE task_lock=0 AND hcode=?) selectLock ON updateLock.hcode=selectLock.hcode SET updateLock.task_lock=1\\\',[\\\'{task_hcode}\\\']) YIELD row AS lock WHERE lock.count>0 WITH lock CALL apoc.load.jdbc(\\\'{jdbc_task_url}\\\',\\\'SELECT DATE_FORMAT(rel_check_point,\\\\\\\'%Y-%m-%d %H:%i:%s\\\\\\\') AS check_point FROM ONGDB_TASK_CHECK_POINT WHERE hcode=?\\\',[\\\'{task_hcode}\\\']) YIELD row WITH apoc.text.join([\\\'\\\\\\\'\\\',row.check_point,\\\'\\\\\\\'\\\'], \\\'\\\') AS check_point,row.check_point AS rawCheckPoint CALL apoc.load.jdbc(\\\'{jdbc_etl_url}\\\',\\\'{max_min_sql}\\\',[rawCheckPoint]) YIELD row WITH row.min AS min,row.max AS max,check_point,rawCheckPoint WITH apoc.coll.union(olab.ids.batch(min,max,10000),[[0,1]]) AS value,check_point,rawCheckPoint UNWIND value AS bactIdList WITH bactIdList[0] AS batchMin,bactIdList[1] AS batchMax,check_point,rawCheckPoint WITH REPLACE(\\\'CALL apoc.load.jdbc(\\\\\\\'{jdbc_etl_url}\\\\\\\', \\\\\\\'{etl_sql}\\\\\\\',[check_point,batchMin,batchMax])\\\',\\\'check_point,batchMin,batchMax\\\',check_point+\\\',\\\'+batchMin+\\\',\\\'+batchMax) AS sqlData,rawCheckPoint CALL apoc.periodic.iterate(sqlData,\\\'MATCH (from:{matchFrom_label} {{matchFrom_field}:row.from}),(to:{matchTo_label} {{matchTo_field}:row.to}) MERGE (from)-[r:{merge_rel_type}]->(to) SET r+=olab.reset.map(row,[\\\\\\\'from\\\\\\\',\\\\\\\'to\\\\\\\'])\\\', {parallel:false,batchSize:10000,iterateList:true}) YIELD batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations WITH batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations,rawCheckPoint WITH SUM(batch.failed) AS batchFailedSize,rawCheckPoint CALL apoc.do.case([batchFailedSize>0,\\\'CALL apoc.load.jdbcUpdate(\\\\\\\'{jdbc_task_url}\\\\\\\',\\\\\\\'UPDATE ONGDB_TASK_CHECK_POINT SET node_check_point=? WHERE hcode=?\\\\\\\',[$rawCheckPoint,\\\\\\\'{task_hcode}\\\\\\\']) YIELD row RETURN row\\\'],\\\'CALL apoc.load.jdbcUpdate(\\\\\\\'{jdbc_task_url}\\\\\\\',\\\\\\\'UPDATE ONGDB_TASK_CHECK_POINT SET rel_check_point=node_check_point WHERE hcode=?\\\\\\\',[\\\\\\\'{task_hcode}\\\\\\\']) YIELD row RETURN row\\\',{rawCheckPoint:rawCheckPoint}) YIELD value WITH value,batchFailedSize,rawCheckPoint CALL apoc.load.jdbcUpdate(\\\'{jdbc_task_url}\\\',\\\'UPDATE ONGDB_TASK_CHECK_POINT_LOCK SET task_lock=0 WHERE hcode=?\\\',[\\\'{task_hcode}\\\']) YIELD row AS releaseLock RETURN {releaseLock:releaseLock,value:value,batchFailedSize:batchFailedSize,rawCheckPoint:rawCheckPoint} AS result;\' AS cypher_task WITH \'task.rel.\'+REPLACE(apoc.text.regexGroups(task_hcode,\'((?!=,)([A-Za-z0-9_\u4e00-\u9fa5]+))+\')[2][0],\'-\',\'_\')+\'.\'+REDUCE(hcode=\'\',hcodeStrs IN apoc.text.regexGroups(task_hcode,\'((?!=,)([A-Za-z0-9_\u4e00-\u9fa5]+))+\') | hcode+hcodeStrs[0]+\'_\') AS relTaskName,olab.replace(cypher_task,[{raw:\'{matchFrom_label}\',rep:matchFrom_label},{raw:\'{matchFrom_field}\',rep:matchFrom_field},{raw:\'{matchTo_label}\',rep:matchTo_label},{raw:\'{matchTo_field}\',rep:matchTo_field},{raw:\'{merge_rel_type}\',rep:merge_rel_type},{raw:\'{jdbc_etl_url}\',rep:jdbc_etl_url},{raw:\'{etl_sql}\',rep:REPLACE(etl_sql,\'\\\'\',\'\\\\\\\\\\\\\\\'\')},{raw:\'{jdbc_task_url}\',rep:jdbc_task_url},{raw:\'{task_hcode}\',rep:task_hcode},{raw:\'{max_min_sql}\',rep:REPLACE(max_min_sql,\'\\\'\',\'\\\\\\\'\')}]) AS cypher_task CALL apoc.custom.asProcedure(relTaskName,cypher_task,\'WRITE\',[[\'result\',\'MAP\']],[],\'构建REL-TASK：{matchFrom_label}-查询FROM节点;{matchFrom_field}-查询FROM节点的唯一性字段;{matchTo_label}-查询TO节点;{matchTo_field}-查询FROM节点的唯一性字段;{merge_rel_type}-合并的关系类型;{jdbc_etl_url}-支持MySQL、Oracle、SqlServer ;{etl_sql}-抽取数据的SQL;{jdbc_task_url}-任务状态表状态锁表位置默认只支持MySQL;{task_hcode}-HGRAPHTASK(FromLabel)-[RelaName]->(ToLabel);{max_min_sql}-从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间\') RETURN \'CALL custom.\'+relTaskName AS procedure_name,cypher_task',
  'WRITE',
  [['procedure_name','STRING'],['cypher_task','STRING']],
  [['matchFrom_label','STRING'],['matchFrom_field','STRING'],['matchTo_label','STRING'],['matchTo_field','STRING'],['merge_rel_type','STRING'],['jdbc_etl_url','STRING'],['etl_sql','STRING'],['jdbc_task_url','STRING'],['task_hcode','STRING'],['max_min_sql','STRING']],
  '构建REL-TASK：{matchFrom_label}-查询FROM节点;{matchFrom_field}-查询FROM节点的唯一性字段;{matchTo_label}-查询TO节点;{matchTo_field}-查询FROM节点的唯一性字段;{merge_rel_type}-合并的关系类型;{jdbc_etl_url}-支持MySQL、Oracle、SqlServer ;{etl_sql}-抽取数据的SQL;{jdbc_task_url}-任务状态表状态锁表位置默认只支持MySQL;{task_hcode}-HGRAPHTASK(FromLabel)-[RelaName]->(ToLabel);{max_min_sql}-从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间'
);
```
### 关系CYPHER-TASK自动生成
- REL-CYPHER-TASK生成
```
@param {matchFrom_label}-匹配FROM节点的标签
@param {matchFrom_field}-匹配FROM节点的属性字段【一般为唯一性属性字段】
@param {matchTo_label}-匹配TO节点的标签
@param {matchTo_field}-匹配TO节点的属性字段【一般为唯一性属性字段】
@param {merge_rel_type}-合并的关系类型
@param {jdbc_etl_url}-支持MySQL、Oracle、SqlServer
@param {etl_sql}-抽取数据的SQL【必须包含from和to字段】
@param {jdbc_task_url}-任务状态表状态锁表位置默认只支持MySQL
@param {task_hcode}-HGRAPHTASK(FromLabel)-[RelaName]->(ToLabel)
@param {max_min_sql}-从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间
WITH 'HBondOrg' AS matchFrom_label,'hcode' AS matchFrom_field,'HEventBond' AS matchTo_label,'hcode' AS matchTo_field,'发行证券' AS merge_rel_type,'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC' AS jdbc_etl_url,'SELECT hcode AS `to`,org_hcode AS `from`,data_source,CONVERT(DATE_FORMAT(hcreatetime,\'%Y%m%d%H%i%S\'),UNSIGNED INTEGER) AS hcreatetime,CONVERT(DATE_FORMAT(hupdatetime,\'%Y%m%d%H%i%S\'),UNSIGNED INTEGER) AS hupdatetime,hisvalid,create_by,update_by FROM HBondOrg WHERE hupdatetime>=? AND huid>=? AND huid<=? AND type=\'发行证券\'' AS etl_sql,'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC' AS jdbc_task_url,'HGRAPHTASK(HBondOrg)-[发行证券]->(HEventBond)' AS task_hcode,'SELECT MIN(huid) AS min,MAX(huid) AS max FROM HBondOrg WHERE hupdatetime>=? AND type=\'发行证券\'' AS max_min_sql
CALL custom.task.build.graph.rel(matchFrom_label,matchFrom_field,matchTo_label,matchTo_field,merge_rel_type,jdbc_etl_url,etl_sql,jdbc_task_url,task_hcode,max_min_sql) YIELD procedure_name,cypher_task RETURN procedure_name,cypher_task
```
- REL-CYPHER-TASK生成【生成`产品`关系】
```
CALL custom.task.rel.json.产品.HGRAPHTASK_HORGProductCalc_产品_产品_()
```


