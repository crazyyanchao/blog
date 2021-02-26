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

## 变量冲突调整
```
文本中下列变量在使用时去掉双下划线`--`【因为与GitHubIO.blog编译冲突】
{--{merge_field}
```

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
CALL custom.task.build.graph.from({merge_label},{merge_field},{child_labels},{jdbc_etl_url},{etl_sql},{jdbc_task_url},{task_hcode},{max_min_sql}) YIELD procedure_name,cypher_task RETURN procedure_name,cypher_task
CALL custom.task.build.graph.to({merge_label},{merge_field},{child_labels},{jdbc_etl_url},{etl_sql},{jdbc_task_url},{task_hcode},{max_min_sql}) YIELD procedure_name,cypher_task RETURN procedure_name,cypher_task
CALL custom.task.build.graph.rel({matchFrom_label},{matchFrom_field},{matchTo_label},{matchTo_field},{merge_rel_type},{jdbc_etl_url},{etl_sql},{jdbc_task_url},{task_hcode},{max_min_sql}) YIELD procedure_name,cypher_task RETURN procedure_name,cypher_task
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
CALL custom.task.rel.发行证券.HGRAPHTASK_HBondOrg_发行证券_HEventBond_()
```

### 生成custom.task.build.graph.*过程
- 生成task.build.graph.from过程【构建FROM-CYPHER-TASK脚本的过程】
```
@param {merge_label}-合并节点的数据模型标签
@param {merge_field}-合并节点的字段
@param {child_labels}-JsonArrayString数组,空数组传入[]
@param {jdbc_etl_url}-支持MySQL、Oracle、SqlServer
@param {etl_sql}-抽取数据的SQL
@param {jdbc_task_url}-任务状态表状态锁表位置默认只支持MySQL
@param {task_hcode}-HGRAPHTASK(FromLabel)-[RelaName]->(ToLabel)
@param {max_min_sql}-从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间
CALL apoc.custom.asProcedure(
  'task.build.graph.from',
  'WITH $merge_label AS merge_label,$merge_field AS merge_field,$child_labels AS child_labels,$jdbc_etl_url AS jdbc_etl_url,$etl_sql AS etl_sql,$jdbc_task_url AS jdbc_task_url,$task_hcode AS task_hcode,$max_min_sql AS max_min_sql,\'CALL apoc.load.jdbcUpdate(\\\'{jdbc_task_url}\\\',\\\'UPDATE ONGDB_TASK_CHECK_POINT_LOCK updateLock INNER JOIN (SELECT task_lock,hcode FROM ONGDB_TASK_CHECK_POINT_LOCK selectLock WHERE task_lock=0 AND hcode=?) selectLock ON updateLock.hcode=selectLock.hcode SET updateLock.task_lock=1\\\',[\\\'{task_hcode}\\\']) YIELD row AS lock WHERE lock.count>0 WITH lock CALL apoc.load.jdbc(\\\'{jdbc_task_url}\\\',\\\'SELECT DATE_FORMAT(node_check_point,\\\\\\\'%Y-%m-%d %H:%i:%s\\\\\\\') AS check_point,DATE_FORMAT(NOW(),\\\\\\\'%Y-%m-%d %H:%i:%s\\\\\\\') AS currentTime FROM ONGDB_TASK_CHECK_POINT WHERE hcode=?\\\',[\\\'{task_hcode}\\\']) YIELD row WITH apoc.text.join([\\\'\\\\\\\'\\\',row.check_point,\\\'\\\\\\\'\\\'], \\\'\\\') AS check_point,row.currentTime AS currentTime,row.check_point AS rawCheckPoint CALL apoc.load.jdbc(\\\'{jdbc_etl_url}\\\',\\\'{max_min_sql}\\\',[rawCheckPoint]) YIELD row WITH row.min AS min,row.max AS max,check_point,currentTime,rawCheckPoint WITH apoc.coll.union(olab.ids.batch(min,max,10000),[[0,1]]) AS value,check_point,currentTime,rawCheckPoint UNWIND value AS bactIdList WITH bactIdList[0] AS batchMin,bactIdList[1] AS batchMax,check_point,currentTime,rawCheckPoint WITH REPLACE(\\\'CALL apoc.load.jdbc(\\\\\\\'{jdbc_etl_url}\\\\\\\', \\\\\\\'{etl_sql}\\\\\\\',[check_point,batchMin,batchMax])\\\',\\\'check_point,batchMin,batchMax\\\',check_point+\\\',\\\'+batchMin+\\\',\\\'+batchMax) AS sqlData,currentTime,rawCheckPoint CALL apoc.periodic.iterate(sqlData,\\\'MERGE (n:{merge_label} {--{merge_field}:row.{merge_field}}) SET n+=row WITH n,row CALL apoc.create.addLabels(n,apoc.coll.union(apoc.convert.fromJsonList(row.label),{child_labels})) YIELD node RETURN node\\\', {parallel:false,batchSize:10000,iterateList:true}) YIELD batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations WITH batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations,currentTime,rawCheckPoint WITH SUM(batch.failed) AS batchFailedSize,currentTime,rawCheckPoint CALL apoc.do.case([batchFailedSize<1,\\\'CALL apoc.load.jdbcUpdate(\\\\\\\'{jdbc_task_url}\\\\\\\',\\\\\\\'UPDATE ONGDB_TASK_CHECK_POINT SET node_check_point=?,rel_check_point=? WHERE hcode=?\\\\\\\',[$currentTime,$rawCheckPoint,\\\\\\\'{task_hcode}\\\\\\\']) YIELD row RETURN row;\\\'],\\\'\\\',{currentTime:currentTime,rawCheckPoint:rawCheckPoint}) YIELD value WITH value,batchFailedSize,currentTime,rawCheckPoint CALL apoc.load.jdbcUpdate(\\\'{jdbc_task_url}\\\',\\\'UPDATE ONGDB_TASK_CHECK_POINT_LOCK SET task_lock=0 WHERE hcode=?\\\',[\\\'{task_hcode}\\\']) YIELD row AS releaseLock RETURN {releaseLock:releaseLock,value:value,batchFailedSize:batchFailedSize,currentTime:currentTime,rawCheckPoint:rawCheckPoint} AS result;\' AS cypher_task WITH \'task.node.\'+REPLACE(\'node-F-\'+apoc.text.regexGroups(task_hcode,\'((?!=,)([A-Za-z0-9_\u4e00-\u9fa5]+))+\')[1][1],\'-\',\'_\')+\'.\'+REDUCE(hcode=\'\',hcodeStrs IN apoc.text.regexGroups(task_hcode,\'((?!=,)([A-Za-z0-9_\u4e00-\u9fa5]+))+\') | hcode+hcodeStrs[0]+\'_\') AS nodeTaskName,olab.replace(cypher_task,[{raw:\'{merge_label}\',rep:merge_label},{raw:\'{merge_field}\',rep:merge_field},{raw:\'{child_labels}\',rep:child_labels},{raw:\'{jdbc_etl_url}\',rep:jdbc_etl_url},{raw:\'{etl_sql}\',rep:REPLACE(etl_sql,\'\\\'\',\'\\\\\\\\\\\\\\\'\')},{raw:\'{jdbc_task_url}\',rep:jdbc_task_url},{raw:\'{task_hcode}\',rep:task_hcode},{raw:\'{max_min_sql}\',rep:REPLACE(max_min_sql,\'\\\'\',\'\\\\\\\'\')}]) AS cypher_task CALL apoc.custom.asProcedure(nodeTaskName,cypher_task,\'WRITE\',[[\'result\',\'MAP\']],[],\'构建FROM-NODE-TASK：{merge_label}-合并节点的数据模型标签;{merge_field}-合并节点的字段;{child_labels}-JsonArrayString数组,空数组传入[] ;{jdbc_etl_url}-支持MySQL、Oracle、SqlServer ;{etl_sql}-抽取数据的SQL;{jdbc_task_url}-任务状态表状态锁表位置默认只支持MySQL ;{task_hcode}-HGRAPHTASK(FromLabel)-[RelaName]->(ToLabel);{max_min_sql}-从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间\') RETURN \'CALL custom.\'+nodeTaskName AS procedure_name,cypher_task',
  'WRITE',
  [['procedure_name','STRING'],['cypher_task','STRING']],
  [['merge_label','STRING'],['merge_field','STRING'],['child_labels','STRING'],['jdbc_etl_url','STRING'],['etl_sql','STRING'],['jdbc_task_url','STRING'],['task_hcode','STRING'],['max_min_sql','STRING']],
  '构建FROM-NODE-TASK：{merge_label}-合并节点的数据模型标签;{merge_field}-合并节点的字段;{child_labels}-JsonArrayString数组,空数组传入[] ;{jdbc_etl_url}-支持MySQL、Oracle、SqlServer ;{etl_sql}-抽取数据的SQL ;{jdbc_task_url}-任务状态表状态锁表位置默认只支持MySQL ;{task_hcode}-HGRAPHTASK(FromLabel)-[RelaName]->(ToLabel);{max_min_sql}-从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间'
);
```
- 生成task.build.graph.to过程【构建TO-CYPHER-TASK脚本的过程】
```
@param {merge_label}-合并节点的数据模型标签
@param {merge_field}-合并节点的字段
@param {child_labels}-JsonArrayString数组,空数组传入[]
@param {jdbc_etl_url}-支持MySQL、Oracle、SqlServer
@param {etl_sql}-抽取数据的SQL
@param {jdbc_task_url}-任务状态表状态锁表位置默认只支持MySQL
@param {task_hcode}-HGRAPHTASK(FromLabel)-[RelaName]->(ToLabel)
@param {max_min_sql}-从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间
CALL apoc.custom.asProcedure(
  'task.build.graph.to',
  'WITH $merge_label AS merge_label,$merge_field AS merge_field,$child_labels AS child_labels,$jdbc_etl_url AS jdbc_etl_url,$etl_sql AS etl_sql,$jdbc_task_url AS jdbc_task_url,$task_hcode AS task_hcode,$max_min_sql AS max_min_sql,\'CALL apoc.load.jdbcUpdate(\\\'{jdbc_task_url}\\\',\\\'UPDATE ONGDB_TASK_CHECK_POINT_LOCK updateLock INNER JOIN (SELECT task_lock,hcode FROM ONGDB_TASK_CHECK_POINT_LOCK selectLock WHERE task_lock=0 AND hcode=?) selectLock ON updateLock.hcode=selectLock.hcode SET updateLock.task_lock=1\\\',[\\\'{task_hcode}\\\']) YIELD row AS lock WHERE lock.count>0 WITH lock CALL apoc.load.jdbc(\\\'{jdbc_task_url}\\\',\\\'SELECT DATE_FORMAT(rel_check_point,\\\\\\\'%Y-%m-%d %H:%i:%s\\\\\\\') AS check_point,DATE_FORMAT(NOW(),\\\\\\\'%Y-%m-%d %H:%i:%s\\\\\\\') AS currentTime FROM ONGDB_TASK_CHECK_POINT WHERE hcode=?\\\',[\\\'{task_hcode}\\\']) YIELD row WITH apoc.text.join([\\\'\\\\\\\'\\\',row.check_point,\\\'\\\\\\\'\\\'], \\\'\\\') AS check_point,row.currentTime AS currentTime,row.check_point AS rawCheckPoint CALL apoc.load.jdbc(\\\'{jdbc_etl_url}\\\',\\\'{max_min_sql}\\\',[rawCheckPoint]) YIELD row WITH row.min AS min,row.max AS max,check_point,currentTime,rawCheckPoint WITH apoc.coll.union(olab.ids.batch(min,max,10000),[[0,1]]) AS value,check_point,currentTime,rawCheckPoint UNWIND value AS bactIdList WITH bactIdList[0] AS batchMin,bactIdList[1] AS batchMax,check_point,currentTime,rawCheckPoint WITH REPLACE(\\\'CALL apoc.load.jdbc(\\\\\\\'{jdbc_etl_url}\\\\\\\', \\\\\\\'{etl_sql}\\\\\\\',[check_point,batchMin,batchMax])\\\',\\\'check_point,batchMin,batchMax\\\',check_point+\\\',\\\'+batchMin+\\\',\\\'+batchMax) AS sqlData,currentTime,rawCheckPoint CALL apoc.periodic.iterate(sqlData,\\\'MERGE (n:{merge_label} {--{merge_field}:row.{merge_field}}) SET n+=row WITH n,row CALL apoc.create.addLabels(n,apoc.coll.union(apoc.convert.fromJsonList(row.label),{child_labels})) YIELD node RETURN node\\\', {parallel:false,batchSize:10000,iterateList:true}) YIELD batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations WITH batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations,currentTime,rawCheckPoint WITH SUM(batch.failed) AS batchFailedSize,currentTime,rawCheckPoint CALL apoc.do.case([batchFailedSize>0,\\\'CALL apoc.load.jdbcUpdate(\\\\\\\'{jdbc_task_url}\\\\\\\',\\\\\\\'UPDATE ONGDB_TASK_CHECK_POINT SET node_check_point=rel_check_point WHERE hcode=?\\\\\\\',[\\\\\\\'{task_hcode}\\\\\\\']) YIELD row RETURN row;\\\'],\\\'\\\',{}) YIELD value WITH value,batchFailedSize,currentTime,rawCheckPoint CALL apoc.load.jdbcUpdate(\\\'{jdbc_task_url}\\\',\\\'UPDATE ONGDB_TASK_CHECK_POINT_LOCK SET task_lock=0 WHERE hcode=?\\\',[\\\'{task_hcode}\\\']) YIELD row AS releaseLock RETURN {releaseLock:releaseLock,value:value,batchFailedSize:batchFailedSize,currentTime:currentTime,rawCheckPoint:rawCheckPoint} AS result;\' AS cypher_task WITH \'task.node.\'+REPLACE(\'node-T-\'+apoc.text.regexGroups(task_hcode,\'((?!=,)([A-Za-z0-9_\u4e00-\u9fa5]+))+\')[3][0],\'-\',\'_\')+\'.\'+REDUCE(hcode=\'\',hcodeStrs IN apoc.text.regexGroups(task_hcode,\'((?!=,)([A-Za-z0-9_\u4e00-\u9fa5]+))+\') | hcode+hcodeStrs[0]+\'_\') AS nodeTaskName,olab.replace(cypher_task,[{raw:\'{merge_label}\',rep:merge_label},{raw:\'{merge_field}\',rep:merge_field},{raw:\'{child_labels}\',rep:child_labels},{raw:\'{jdbc_etl_url}\',rep:jdbc_etl_url},{raw:\'{etl_sql}\',rep:REPLACE(etl_sql,\'\\\'\',\'\\\\\\\\\\\\\\\'\')},{raw:\'{jdbc_task_url}\',rep:jdbc_task_url},{raw:\'{task_hcode}\',rep:task_hcode},{raw:\'{max_min_sql}\',rep:REPLACE(max_min_sql,\'\\\'\',\'\\\\\\\'\')}]) AS cypher_task CALL apoc.custom.asProcedure(nodeTaskName,cypher_task,\'WRITE\',[[\'result\',\'MAP\']],[],\'构建TO-NODE-TASK：{merge_label}-合并节点的数据模型标签;{merge_field}-合并节点的字段;{child_labels}-JsonArrayString数组,空数组传入[] ;{jdbc_etl_url}-支持MySQL、Oracle、SqlServer ;{etl_sql}-抽取数据的SQL ;{jdbc_task_url}-任务状态表状态锁表位置默认只支持MySQL ;{task_hcode}-HGRAPHTASK(FromLabel)-[RelaName]->(ToLabel);{max_min_sql}-从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间\') RETURN \'CALL custom.\'+nodeTaskName AS procedure_name,cypher_task',
  'WRITE',
  [['procedure_name','STRING'],['cypher_task','STRING']],
  [['merge_label','STRING'],['merge_field','STRING'],['child_labels','STRING'],['jdbc_etl_url','STRING'],['etl_sql','STRING'],['jdbc_task_url','STRING'],['task_hcode','STRING'],['max_min_sql','STRING']],
  '构建TO-NODE-TASK：{merge_label}-合并节点的数据模型标签;{merge_field}-合并节点的字段;{child_labels}-JsonArrayString数组,空数组传入[] ;{jdbc_etl_url}-支持MySQL、Oracle、SqlServer ;{etl_sql}-抽取数据的SQL ;{jdbc_task_url}-任务状态表状态锁表位置默认只支持MySQL ;{task_hcode}-HGRAPHTASK(FromLabel)-[RelaName]->(ToLabel);{max_min_sql}-从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间'
);
```
- 生成task.build.graph.rel过程【构建REL-CYPHER-TASK脚本的过程】
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
CALL apoc.custom.asProcedure(
  'task.build.graph.rel',
  'WITH $matchFrom_label AS matchFrom_label,$matchFrom_field AS matchFrom_field,$matchTo_label AS matchTo_label,$matchTo_field AS matchTo_field,$merge_rel_type AS merge_rel_type,$jdbc_etl_url AS jdbc_etl_url,$etl_sql AS etl_sql,$jdbc_task_url AS jdbc_task_url,$task_hcode AS task_hcode,$max_min_sql AS max_min_sql,\'CALL apoc.load.jdbcUpdate(\\\'{jdbc_task_url}\\\',\\\'UPDATE ONGDB_TASK_CHECK_POINT_LOCK updateLock INNER JOIN (SELECT task_lock,hcode FROM ONGDB_TASK_CHECK_POINT_LOCK selectLock WHERE task_lock=0 AND hcode=?) selectLock ON updateLock.hcode=selectLock.hcode SET updateLock.task_lock=1\\\',[\\\'{task_hcode}\\\']) YIELD row AS lock WHERE lock.count>0 WITH lock CALL apoc.load.jdbc(\\\'{jdbc_task_url}\\\',\\\'SELECT DATE_FORMAT(rel_check_point,\\\\\\\'%Y-%m-%d %H:%i:%s\\\\\\\') AS check_point FROM ONGDB_TASK_CHECK_POINT WHERE hcode=?\\\',[\\\'{task_hcode}\\\']) YIELD row WITH apoc.text.join([\\\'\\\\\\\'\\\',row.check_point,\\\'\\\\\\\'\\\'], \\\'\\\') AS check_point,row.check_point AS rawCheckPoint CALL apoc.load.jdbc(\\\'{jdbc_etl_url}\\\',\\\'{max_min_sql}\\\',[rawCheckPoint]) YIELD row WITH row.min AS min,row.max AS max,check_point,rawCheckPoint WITH apoc.coll.union(olab.ids.batch(min,max,10000),[[0,1]]) AS value,check_point,rawCheckPoint UNWIND value AS bactIdList WITH bactIdList[0] AS batchMin,bactIdList[1] AS batchMax,check_point,rawCheckPoint WITH REPLACE(\\\'CALL apoc.load.jdbc(\\\\\\\'{jdbc_etl_url}\\\\\\\', \\\\\\\'{etl_sql}\\\\\\\',[check_point,batchMin,batchMax])\\\',\\\'check_point,batchMin,batchMax\\\',check_point+\\\',\\\'+batchMin+\\\',\\\'+batchMax) AS sqlData,rawCheckPoint CALL apoc.periodic.iterate(sqlData,\\\'MATCH (from:{matchFrom_label} {{matchFrom_field}:row.from}),(to:{matchTo_label} {{matchTo_field}:row.to}) MERGE (from)-[r:{merge_rel_type}]->(to) SET r+=olab.reset.map(row,[\\\\\\\'from\\\\\\\',\\\\\\\'to\\\\\\\'])\\\', {parallel:false,batchSize:10000,iterateList:true}) YIELD batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations WITH batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations,rawCheckPoint WITH SUM(batch.failed) AS batchFailedSize,rawCheckPoint CALL apoc.do.case([batchFailedSize>0,\\\'CALL apoc.load.jdbcUpdate(\\\\\\\'{jdbc_task_url}\\\\\\\',\\\\\\\'UPDATE ONGDB_TASK_CHECK_POINT SET node_check_point=? WHERE hcode=?\\\\\\\',[$rawCheckPoint,\\\\\\\'{task_hcode}\\\\\\\']) YIELD row RETURN row\\\'],\\\'CALL apoc.load.jdbcUpdate(\\\\\\\'{jdbc_task_url}\\\\\\\',\\\\\\\'UPDATE ONGDB_TASK_CHECK_POINT SET rel_check_point=node_check_point WHERE hcode=?\\\\\\\',[\\\\\\\'{task_hcode}\\\\\\\']) YIELD row RETURN row\\\',{rawCheckPoint:rawCheckPoint}) YIELD value WITH value,batchFailedSize,rawCheckPoint CALL apoc.load.jdbcUpdate(\\\'{jdbc_task_url}\\\',\\\'UPDATE ONGDB_TASK_CHECK_POINT_LOCK SET task_lock=0 WHERE hcode=?\\\',[\\\'{task_hcode}\\\']) YIELD row AS releaseLock RETURN {releaseLock:releaseLock,value:value,batchFailedSize:batchFailedSize,rawCheckPoint:rawCheckPoint} AS result;\' AS cypher_task WITH \'task.rel.\'+REPLACE(apoc.text.regexGroups(task_hcode,\'((?!=,)([A-Za-z0-9_\u4e00-\u9fa5]+))+\')[2][0],\'-\',\'_\')+\'.\'+REDUCE(hcode=\'\',hcodeStrs IN apoc.text.regexGroups(task_hcode,\'((?!=,)([A-Za-z0-9_\u4e00-\u9fa5]+))+\') | hcode+hcodeStrs[0]+\'_\') AS relTaskName,olab.replace(cypher_task,[{raw:\'{matchFrom_label}\',rep:matchFrom_label},{raw:\'{matchFrom_field}\',rep:matchFrom_field},{raw:\'{matchTo_label}\',rep:matchTo_label},{raw:\'{matchTo_field}\',rep:matchTo_field},{raw:\'{merge_rel_type}\',rep:merge_rel_type},{raw:\'{jdbc_etl_url}\',rep:jdbc_etl_url},{raw:\'{etl_sql}\',rep:REPLACE(etl_sql,\'\\\'\',\'\\\\\\\\\\\\\\\'\')},{raw:\'{jdbc_task_url}\',rep:jdbc_task_url},{raw:\'{task_hcode}\',rep:task_hcode},{raw:\'{max_min_sql}\',rep:REPLACE(max_min_sql,\'\\\'\',\'\\\\\\\'\')}]) AS cypher_task CALL apoc.custom.asProcedure(relTaskName,cypher_task,\'WRITE\',[[\'result\',\'MAP\']],[],\'构建REL-TASK：{matchFrom_label}-查询FROM节点;{matchFrom_field}-查询FROM节点的唯一性字段;{matchTo_label}-查询TO节点;{matchTo_field}-查询FROM节点的唯一性字段;{merge_rel_type}-合并的关系类型;{jdbc_etl_url}-支持MySQL、Oracle、SqlServer ;{etl_sql}-抽取数据的SQL;{jdbc_task_url}-任务状态表状态锁表位置默认只支持MySQL;{task_hcode}-HGRAPHTASK(FromLabel)-[RelaName]->(ToLabel);{max_min_sql}-从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间\') RETURN \'CALL custom.\'+relTaskName AS procedure_name,cypher_task',
  'WRITE',
  [['procedure_name','STRING'],['cypher_task','STRING']],
  [['matchFrom_label','STRING'],['matchFrom_field','STRING'],['matchTo_label','STRING'],['matchTo_field','STRING'],['merge_rel_type','STRING'],['jdbc_etl_url','STRING'],['etl_sql','STRING'],['jdbc_task_url','STRING'],['task_hcode','STRING'],['max_min_sql','STRING']],
  '构建REL-TASK：{matchFrom_label}-查询FROM节点;{matchFrom_field}-查询FROM节点的唯一性字段;{matchTo_label}-查询TO节点;{matchTo_field}-查询FROM节点的唯一性字段;{merge_rel_type}-合并的关系类型;{jdbc_etl_url}-支持MySQL、Oracle、SqlServer ;{etl_sql}-抽取数据的SQL;{jdbc_task_url}-任务状态表状态锁表位置默认只支持MySQL;{task_hcode}-HGRAPHTASK(FromLabel)-[RelaName]->(ToLabel);{max_min_sql}-从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间'
);
```
- 移除过程
```
CALL apoc.custom.removeProcedure('task.build.graph.from')
```

### 节点CYPHER-TASK生成
- 入参说明
```
@param {merge_label}  合并节点的数据模型标签 eg.HBondOrg
@param {merge_field}  合并节点的字段 eg.hcode
@param {child_labels} JsonArrayString数组,空数组传入[] eg.apoc.coll.union(apoc.convert.fromJsonList(row.label),{child_labels})
@param {jdbc_etl_url} 支持MySQL、Oracle、SqlServer eg.jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC
@param {etl_sql} 抽取数据的SQL eg.SELECT org_hcode AS hcode,org_name AS name,credit_code,label,CONVERT(DATE_FORMAT(hcreatetime,'%Y%m%d%H%i%S'),UNSIGNED INTEGER) AS hcreatetime,CONVERT(DATE_FORMAT(hupdatetime,'%Y%m%d%H%i%S'),UNSIGNED INTEGER) AS hupdatetime,hisvalid,create_by,update_by FROM HBondOrg WHERE hupdatetime>=? AND huid>=? AND huid<=? AND type='发行证券'
@param {jdbc_task_url} 任务状态表状态锁表位置默认只支持MySQL eg.jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC
@param {task_hcode} eg.HGRAPHTASK(FromLabel)-[RelaName]->(ToLabel)
@param {max_min_sql} 从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间 eg.SELECT MIN(huid) AS min,MAX(huid) AS max FROM HBondOrg WHERE hupdatetime>=? AND type='发行证券'
```
```
@param {merge_label}  【必选】合并节点的数据模型标签
@param {merge_field}  【必选】【一般为HCODE】合并节点的字段，该标签下的唯一属性
@param {child_labels} 【可选】【数据模型带过来的标签】STRING类型的JSON数组
@param {jdbc_etl_url} 【必选】【支持MySQL、Oracle、SqlServer】
@param {etl_sql} 【必选】【如有额外标签字段必须重命名为label】【必须包含{merge_field}字段】【SQL必须包含自动更新时间和自增ID】SELECT org_hcode AS hcode,org_name AS name,credit_code,label,CONVERT(DATE_FORMAT(hcreatetime,'%Y%m%d%H%i%S'),UNSIGNED INTEGER) AS hcreatetime,CONVERT(DATE_FORMAT(hupdatetime,'%Y%m%d%H%i%S'),UNSIGNED INTEGER) AS hupdatetime,hisvalid,create_by,update_by FROM HBondOrg WHERE hupdatetime>=? AND huid>=? AND huid<=? AND type='发行证券'
@param {jdbc_task_url} 【必选】【任务状态表状态锁表位置默认只支持MySQL】【提前在连接指定的库中创建ONGDB_TASK_CHECK_POINT和ONGDB_TASK_CHECK_POINT_LOCK表】
@param {task_hcode} 【必选】【格式要求：HGRAPHTASK(FromLabel)-[REL]->(ToLabel)】
@param {max_min_sql} 【必选】【必须包含max和min字段】【必须包含自动更新时间字段】从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间 SELECT MIN(huid) AS min,MAX(huid) AS max FROM HBondOrg WHERE hupdatetime>=? AND type='发行证券'
```
- FROM-CYPHER-TASK生成【生成`HBondOrg`节点】
```
WITH 'HBondOrg' AS merge_label,'hcode' AS merge_field,NULL AS child_labels,'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC' AS jdbc_etl_url,'SELECT org_hcode AS hcode,org_name AS name,credit_code,label,CONVERT(DATE_FORMAT(hcreatetime,\'%Y%m%d%H%i%S\'),UNSIGNED INTEGER) AS hcreatetime,CONVERT(DATE_FORMAT(hupdatetime,\'%Y%m%d%H%i%S\'),UNSIGNED INTEGER) AS hupdatetime,hisvalid,create_by,update_by FROM HBondOrg WHERE hupdatetime>=? AND huid>=? AND huid<=? AND type=\'发行证券\'' AS etl_sql,'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC' AS jdbc_task_url,'HGRAPHTASK(HBondOrg)-[发行证券]->(HEventBond)' AS task_hcode,'SELECT MIN(huid) AS min,MAX(huid) AS max FROM HBondOrg WHERE hupdatetime>=? AND type=\'发行证券\'' AS max_min_sql
CALL custom.task.build.graph.from(merge_label,merge_field,child_labels,jdbc_etl_url,etl_sql,jdbc_task_url,task_hcode,max_min_sql) YIELD procedure_name,cypher_task RETURN procedure_name,cypher_task
```
- TO-CYPHER-TASK生成【生成`HEventBond`节点】
```
WITH 'HEventBond' AS merge_label,'hcode' AS merge_field,NULL AS child_labels,'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC' AS jdbc_etl_url,'SELECT hcode,name,data_source,CONVERT(DATE_FORMAT(hcreatetime,\'%Y%m%d%H%i%S\'),UNSIGNED INTEGER) AS hcreatetime,CONVERT(DATE_FORMAT(hupdatetime,\'%Y%m%d%H%i%S\'),UNSIGNED INTEGER) AS hupdatetime,hisvalid,create_by,update_by FROM HBondOrg WHERE hupdatetime>=? AND huid>=? AND huid<=? AND type=\'发行证券\'' AS etl_sql,'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC' AS jdbc_task_url,'HGRAPHTASK(HBondOrg)-[发行证券]->(HEventBond)' AS task_hcode,'SELECT MIN(huid) AS min,MAX(huid) AS max FROM HBondOrg WHERE hupdatetime>=? AND type=\'发行证券\'' AS max_min_sql
CALL custom.task.build.graph.to(merge_label,merge_field,child_labels,jdbc_etl_url,etl_sql,jdbc_task_url,task_hcode,max_min_sql) YIELD procedure_name,cypher_task RETURN procedure_name,cypher_task
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
- REL-CYPHER-TASK生成【生成`发行证券`关系】
```
CALL custom.task.rel.发行证券.HGRAPHTASK_HBondOrg_发行证券_HEventBond_()
```

## 同构图过程设计
### 同构图-TASK脚本生成
- 使用CALL apoc.custom.asProcedure生成下面两个过程【节点只构建FROM-CYPHER-TASK即可】
```
【与异构图相同】CALL custom.task.build.graph.from({merge_label},{merge_field},{child_labels},{jdbc_etl_url},{etl_sql},{jdbc_task_url},{task_hcode},{max_min_sql})
【与异构图相同】CALL custom.task.build.graph.rel({matchFrom_label},{matchFrom_field},{matchTo_label},{matchTo_field},{merge_rel_type},{jdbc_etl_url},{etl_sql},{jdbc_task_url},{task_hcode},{max_min_sql})
```
- 以一个异构图为例：(HORGGuaranteeV003)-[:担保]->(HORGGuaranteeV003)
- 使用CALL custom.task.build.graph.*()生成下面两个过程
- 节点过程
```
CALL custom.task.node.node_F_HORGGuaranteeV003.HGRAPHTASK_HORGGuaranteeV003_担保_HORGGuaranteeV003_()
```
- 关系过程
```
CALL custom.task.rel.担保.HGRAPHTASK_HORGGuaranteeV003_担保_HORGGuaranteeV003_()
```
### 节点CYPHER-TASK生成
- 入参说明
```
@param {merge_label}  【必选】合并节点的数据模型标签
@param {merge_field}  【必选】【一般为HCODE】合并节点的字段，该标签下的唯一属性
@param {child_labels} 【可选】【数据模型带过来的标签】STRING类型的JSON数组
@param {jdbc_etl_url} 【必选】【支持MySQL、Oracle、SqlServer】
@param {etl_sql} 【必选】【如有额外标签字段必须重命名为label】【必须包含{merge_field}字段】【SQL必须包含自动更新时间和自增ID】eg.SELECT org_hcode AS hcode,org_name AS name,credit_code,label,CONVERT(DATE_FORMAT(hcreatetime,'%Y%m%d%H%i%S'),UNSIGNED INTEGER) AS hcreatetime,CONVERT(DATE_FORMAT(hupdatetime,'%Y%m%d%H%i%S'),UNSIGNED INTEGER) AS hupdatetime,hisvalid,create_by,update_by FROM HBondOrg WHERE hupdatetime>=? AND huid>=? AND huid<=? AND type='发行证券'
@param {jdbc_task_url} 【必选】【任务状态表状态锁表位置默认只支持MySQL】【提前在连接指定的库中创建ONGDB_TASK_CHECK_POINT和ONGDB_TASK_CHECK_POINT_LOCK表】
@param {task_hcode} 【必选】【eg.格式要求：HGRAPHTASK(FromLabel)-[REL]->(ToLabel)】
@param {max_min_sql} 【必选】【必须包含max和min字段】【必须包含自动更新时间字段】从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间 eg.SELECT MIN(huid) AS min,MAX(huid) AS max FROM HBondOrg WHERE hupdatetime>=? AND type='发行证券'
```
- FROM-WITH-TO-CYPHER-TASK生成【生成`HORGGuaranteeV003`节点】
```
WITH 'HORGGuaranteeV003' AS merge_label,'hcode' AS merge_field,NULL AS child_labels,'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC' AS jdbc_etl_url,'SELECT hcode,name,credit_code,label,CONVERT(DATE_FORMAT(hcreatetime,\'%Y%m%d%H%i%S\'),UNSIGNED INTEGER) AS hcreatetime,CONVERT(DATE_FORMAT(hupdatetime,\'%Y%m%d%H%i%S\'),UNSIGNED INTEGER) AS hupdatetime,hisvalid,create_by,update_by FROM HORGGuaranteeV003 WHERE hupdatetime>=? AND huid>=? AND huid<=?' AS etl_sql,'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC' AS jdbc_task_url,'HGRAPHTASK(HORGGuaranteeV003)-[担保]->(HORGGuaranteeV003)' AS task_hcode,'SELECT MIN(huid) AS min,MAX(huid) AS max FROM HORGGuaranteeV003 WHERE hupdatetime>=?' AS max_min_sql
CALL custom.task.build.graph.from(merge_label,merge_field,child_labels,jdbc_etl_url,etl_sql,jdbc_task_url,task_hcode,max_min_sql) YIELD procedure_name,cypher_task RETURN procedure_name,cypher_task
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
WITH 'HORGGuaranteeV003' AS matchFrom_label,'hcode' AS matchFrom_field,'HORGGuaranteeV003' AS matchTo_label,'hcode' AS matchTo_field,'担保' AS merge_rel_type,'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC' AS jdbc_etl_url,'SELECT `from`,`to`,guarantee_detail,guarantee_detail_size,CONVERT(DATE_FORMAT(hcreatetime,\'%Y%m%d%H%i%S\'),UNSIGNED INTEGER) AS hcreatetime,CONVERT(DATE_FORMAT(hupdatetime,\'%Y%m%d%H%i%S\'),UNSIGNED INTEGER) AS hupdatetime,hisvalid,create_by,update_by FROM HORGGuarantee_GuarV003 WHERE hupdatetime>=? AND huid>=? AND huid<=?' AS etl_sql,'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC' AS jdbc_task_url,'HGRAPHTASK(HORGGuaranteeV003)-[担保]->(HORGGuaranteeV003)' AS task_hcode,'SELECT MIN(huid) AS min,MAX(huid) AS max FROM HORGGuarantee_GuarV003 WHERE hupdatetime>=?' AS max_min_sql
CALL custom.task.build.graph.rel(matchFrom_label,matchFrom_field,matchTo_label,matchTo_field,merge_rel_type,jdbc_etl_url,etl_sql,jdbc_task_url,task_hcode,max_min_sql) YIELD procedure_name,cypher_task RETURN procedure_name,cypher_task
```
- REL-CYPHER-TASK生成【生成`担保`关系】
```
CALL custom.task.rel.担保.HGRAPHTASK_HORGGuaranteeV003_担保_HORGGuaranteeV003_()
```

