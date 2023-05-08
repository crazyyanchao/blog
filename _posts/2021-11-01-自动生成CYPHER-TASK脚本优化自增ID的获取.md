---
title: 自动生成CYPHER-TASK脚本优化自增ID的获取
tags: [ONgDB,Cypher,SQL,MySQL,Neo4j]
author: Yc-Ma
show_author_profile: true
key: 2021-11-01-自动生成CYPHER-TASK脚本优化自增ID的获取
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}


## 变量冲突调整
```
文本中下列变量在使用时去掉双下划线`--`【因为与GitHubIO.blog编译冲突】
{--{merge_field}
{--{matchFrom_field}
{--{matchTo_field}
```

~~## 版本V-1.0.1~~
```
支持自定义设置batch size大小！
```
```
在该版本中优化自增ID的获取方式，避免无效查询浪费性能。【2021-01-15-自动生成CYPHER-TASK脚本】在这个版本中，自增ID的获取默认是区间自增，但是当区间中有无效ID产生则获取不到数据。
```
```
在升级版本中继续针对MySQL，对使用自增ID批量处理的逻辑进行优化，引入ID游标机制。游标文件统一保存在MySQL库表中，使用图数据任务ID进行关联。
```

~~## 关键步骤~~
> 任务开始时每次循环，从MySQL游标库表获取对应图数据任务ID关联的下一个MAX ID即可

**【初始获取MIN ID为-1，每次保存MAX ID，新的循环开始时获取MAX ID赋值为MIN ID即可】**

~~## 游标ID表设计~~
```
CREATE TABLE `ONGDB_TASK_CHECK_POINT_CURSOR` (
  `huid` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '自增主键',
  `hcode` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '代码：HGRAPHTASK(FromLabel)-[RelType]->(ToLabel)；与ONGDB_TASK_CHECK_POINT表的HCODE关联',
  `cursor_id` bigint(20) NOT NULL DEFAULT '-1' COMMENT '-1表示默认的任务开始时的【最小自增ID】MIN ID',
  `hcreatetime` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `hupdatetime` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `create_by` varchar(24) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '创建人',
  `update_by` varchar(24) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '更新人',
  `hisvalid` int(11) NOT NULL DEFAULT '1' COMMENT '逻辑删除标记：0-无效；1-有效',
  `src` varchar(32) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL DEFAULT 'ART' COMMENT '数据来源标记',
  PRIMARY KEY (`huid`) USING BTREE,
  UNIQUE KEY `unique_key` (`hcode`) USING BTREE COMMENT '唯一索引',
  KEY `updateTime` (`hupdatetime`) USING BTREE,
  KEY `hisvalid` (`hisvalid`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=742715711 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci ROW_FORMAT=DYNAMIC COMMENT='ONgDB DAG TASK CURSOR【图数据构建任务游标表】【在使用游标ID自增排序逻辑生成TASK时，使用该表保存游标ID】'
```

~~## 获取游标ID~~
```
CALL apoc.load.jdbc('jdbc:mysql://testlab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.alibaba.com.cn:3306/analytics_graph_data?user=dev&password=testlabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC', 'SELECT huid FROM analytics_graph_data.HORGGuaranteeV001') YIELD row RETURN row.cursor_id AS batchMin;
```

```
olab.ids.batch.cursor(获取游标,获取当前游标的下一个游标,10000)
```

```
计算IDS的自动分布
```

```
// 获取当前游标
CALL apoc.load.jdbc('jdbc:mysql://testlab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.alibaba.com.cn:3306/analytics_graph_data?user=dev&password=testlabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC', 'SELECT cursor_id FROM analytics_graph_data.ONGDB_TASK_CHECK_POINT_CURSOR WHERE hcode=?',['HGRAPHTASK(HORGGuaranteeV001)-[担保]->(HORGGuaranteeV001)'])
	YIELD row
    WITH row.cursor_id AS batchMin
// 获取当前游标的下一个游标
```

~~## 更新游标ID~~
```
CALL apoc.load.jdbcUpdate('jdbc:mysql://testlab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.alibaba.com.cn:3306/analytics_graph_data?user=dev&password=testlabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC','UPDATE analytics_graph_data.ONGDB_TASK_CHECK_POINT_CURSOR SET cursor_id=? WHERE hcode=?',[-1,'HGRAPHTASK(HORGGuaranteeV001)-[担保]->(HORGGuaranteeV001)']) YIELD row RETURN row;
```

## 版本V-1.0.2
```
`task.build.graph.from`增加一个`batch`参数，支持自定义{batch}大小【即每个SQL命中的数据量大小-该值为一个预估值：目的为在ID范围较大时，提高SQL执行效率】
```
```
【构建任务时，任务名称保持不变】
```

### 原始查询样例
- 增加两个参数：`ods_batch_size`和`commit_batch_size`
```
WITH 'Label' AS merge_label,'hcode' AS merge_field,NULL AS child_labels,'jdbc_etl_url' AS jdbc_etl_url,'etl_sql' AS etl_sql,'jdbc_task_url' AS jdbc_task_url,'task_hcode' AS task_hcode,'max_min_sql' AS max_min_sql,1000000 AS ods_batch_size,100 AS commit_batch_size,
    'CALL apoc.load.jdbcUpdate(\'{jdbc_task_url}\',\'UPDATE ONGDB_TASK_CHECK_POINT_LOCK updateLock INNER JOIN (SELECT task_lock,hcode FROM ONGDB_TASK_CHECK_POINT_LOCK selectLock WHERE task_lock=0 AND hcode=?) selectLock ON updateLock.hcode=selectLock.hcode SET updateLock.task_lock=1\',[\'{task_hcode}\']) YIELD row AS lock WHERE lock.count>0 WITH lock CALL apoc.load.jdbc(\'{jdbc_task_url}\',\'SELECT DATE_FORMAT(node_check_point,\\\'%Y-%m-%d %H:%i:%s\\\') AS check_point,DATE_FORMAT(NOW(),\\\'%Y-%m-%d %H:%i:%s\\\') AS currentTime FROM ONGDB_TASK_CHECK_POINT WHERE hcode=?\',[\'{task_hcode}\']) YIELD row WITH apoc.text.join([\'\\\'\',row.check_point,\'\\\'\'], \'\') AS check_point,row.currentTime AS currentTime,row.check_point AS rawCheckPoint CALL apoc.load.jdbc(\'{jdbc_etl_url}\',\'{max_min_sql}\',[rawCheckPoint]) YIELD row WITH row.min AS min,row.max AS max,check_point,currentTime,rawCheckPoint WITH apoc.coll.union(olab.ids.batch(min,max,{ods_batch_size}),[[0,1]]) AS value,check_point,currentTime,rawCheckPoint UNWIND value AS bactIdList WITH bactIdList[0] AS batchMin,bactIdList[1] AS batchMax,check_point,currentTime,rawCheckPoint WITH REPLACE(\'CALL apoc.load.jdbc(\\\'{jdbc_etl_url}\\\', \\\'{etl_sql}\\\',[check_point,batchMin,batchMax])\',\'check_point,batchMin,batchMax\',check_point+\',\'+batchMin+\',\'+batchMax) AS sqlData,currentTime,rawCheckPoint CALL apoc.periodic.iterate(sqlData,\'MERGE (n:{merge_label} {--{merge_field}:row.{merge_field}}) SET n+=row WITH n,row CALL apoc.create.addLabels(n,apoc.coll.union(apoc.convert.fromJsonList(row.label),{child_labels})) YIELD node RETURN node\', {parallel:false,batchSize:{commit_batch_size},iterateList:true}) YIELD batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations WITH batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations,currentTime,rawCheckPoint WITH SUM(batch.failed) AS batchFailedSize,currentTime,rawCheckPoint CALL apoc.do.case([batchFailedSize<1,\'CALL apoc.load.jdbcUpdate(\\\'{jdbc_task_url}\\\',\\\'UPDATE ONGDB_TASK_CHECK_POINT SET node_check_point=?,rel_check_point=? WHERE hcode=?\\\',[$currentTime,$rawCheckPoint,\\\'{task_hcode}\\\']) YIELD row RETURN row;\'],\'\',{currentTime:currentTime,rawCheckPoint:rawCheckPoint}) YIELD value WITH value,batchFailedSize,currentTime,rawCheckPoint CALL apoc.load.jdbcUpdate(\'{jdbc_task_url}\',\'UPDATE ONGDB_TASK_CHECK_POINT_LOCK SET task_lock=0 WHERE hcode=?\',[\'{task_hcode}\']) YIELD row AS releaseLock RETURN {releaseLock:releaseLock,value:value,batchFailedSize:batchFailedSize,currentTime:currentTime,rawCheckPoint:rawCheckPoint} AS result;' AS cypher_task
WITH 'task.node.'+REPLACE('node-F-'+apoc.text.regexGroups(task_hcode,'((?!=,)([A-Za-z0-9_一-龥]+))+')[1][1],'-','_')+'.'+REDUCE(hcode='',hcodeStrs IN apoc.text.regexGroups(task_hcode,'((?!=,)([A-Za-z0-9_一-龥]+))+') | hcode+hcodeStrs[0]+'_') AS nodeTaskName,
     olab.replace(cypher_task,[{raw:'{merge_label}',rep:merge_label},{raw:'{merge_field}',rep:merge_field},{raw:'{child_labels}',rep:child_labels},{raw:'{jdbc_etl_url}',rep:jdbc_etl_url},{raw:'{etl_sql}',rep:REPLACE(etl_sql,'\'','\\\\\\\'')},{raw:'{jdbc_task_url}',rep:jdbc_task_url},{raw:'{task_hcode}',rep:task_hcode},{raw:'{max_min_sql}',rep:REPLACE(max_min_sql,'\'','\\\'')},{raw:'{ods_batch_size}',rep:ods_batch_size},{raw:'{commit_batch_size}',rep:commit_batch_size}]) AS cypher_task
RETURN nodeTaskName,cypher_task
```

### 生成custom.task.build.graph.batch.*过程
- 生成task.build.graph.from.batch过程【构建FROM-CYPHER-TASK脚本的过程】
```
@param {merge_label}-合并节点的数据模型标签
@param {merge_field}-合并节点的字段
@param {child_labels}-JsonArrayString数组,空数组传入[]
@param {jdbc_etl_url}-支持MySQL、Oracle、SqlServer
@param {etl_sql}-抽取数据的SQL
@param {jdbc_task_url}-任务状态表状态锁表位置默认只支持MySQL
@param {task_hcode}-HGRAPHTASK(FromLabel)-[RelaName]->(ToLabel)
@param {max_min_sql}-从ETL源表加载最大最小自增ID，源库必须要有自增ID【使用该字段升序排序】和自动更新时间
@param {ods_batch_size}-从源库加载时每个SQL递增的ID尺寸大小【推荐设置为：10000】【task.build.graph.from默认值：10000】
@param {commit_batch_size}-图数据库批量提交的事物规模大小【推荐设置为：1000、10000】【task.build.graph.from默认值：10000】
CALL apoc.custom.asProcedure(
  'task.build.graph.from.batch',
  'WITH $merge_label AS merge_label,$merge_field AS merge_field,$child_labels AS child_labels,$jdbc_etl_url AS jdbc_etl_url,$etl_sql AS etl_sql,$jdbc_task_url AS jdbc_task_url,$task_hcode AS task_hcode,$max_min_sql AS max_min_sql,$ods_batch_size AS ods_batch_size,$commit_batch_size AS commit_batch_size,\'CALL apoc.load.jdbcUpdate(\\\'{jdbc_task_url}\\\',\\\'UPDATE ONGDB_TASK_CHECK_POINT_LOCK updateLock INNER JOIN (SELECT task_lock,hcode FROM ONGDB_TASK_CHECK_POINT_LOCK selectLock WHERE task_lock=0 AND hcode=?) selectLock ON updateLock.hcode=selectLock.hcode SET updateLock.task_lock=1\\\',[\\\'{task_hcode}\\\']) YIELD row AS lock WHERE lock.count>0 WITH lock CALL apoc.load.jdbc(\\\'{jdbc_task_url}\\\',\\\'SELECT DATE_FORMAT(node_check_point,\\\\\\\'%Y-%m-%d %H:%i:%s\\\\\\\') AS check_point,DATE_FORMAT(NOW(),\\\\\\\'%Y-%m-%d %H:%i:%s\\\\\\\') AS currentTime FROM ONGDB_TASK_CHECK_POINT WHERE hcode=?\\\',[\\\'{task_hcode}\\\']) YIELD row WITH apoc.text.join([\\\'\\\\\\\'\\\',row.check_point,\\\'\\\\\\\'\\\'], \\\'\\\') AS check_point,row.currentTime AS currentTime,row.check_point AS rawCheckPoint CALL apoc.load.jdbc(\\\'{jdbc_etl_url}\\\',\\\'{max_min_sql}\\\',[rawCheckPoint]) YIELD row WITH row.min AS min,row.max AS max,check_point,currentTime,rawCheckPoint WITH apoc.coll.union(olab.ids.batch(min,max,{ods_batch_size}),[[0,1]]) AS value,check_point,currentTime,rawCheckPoint UNWIND value AS bactIdList WITH bactIdList[0] AS batchMin,bactIdList[1] AS batchMax,check_point,currentTime,rawCheckPoint WITH REPLACE(\\\'CALL apoc.load.jdbc(\\\\\\\'{jdbc_etl_url}\\\\\\\', \\\\\\\'{etl_sql}\\\\\\\',[check_point,batchMin,batchMax])\\\',\\\'check_point,batchMin,batchMax\\\',check_point+\\\',\\\'+batchMin+\\\',\\\'+batchMax) AS sqlData,currentTime,rawCheckPoint CALL apoc.periodic.iterate(sqlData,\\\'MERGE (n:{merge_label} {--{merge_field}:row.{merge_field}}) SET n+=row WITH n,row CALL apoc.create.addLabels(n,apoc.coll.union(apoc.convert.fromJsonList(row.label),{child_labels})) YIELD node RETURN node\\\', {parallel:false,batchSize:{commit_batch_size},iterateList:true}) YIELD batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations WITH batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations,currentTime,rawCheckPoint WITH SUM(batch.failed) AS batchFailedSize,currentTime,rawCheckPoint CALL apoc.do.case([batchFailedSize<1,\\\'CALL apoc.load.jdbcUpdate(\\\\\\\'{jdbc_task_url}\\\\\\\',\\\\\\\'UPDATE ONGDB_TASK_CHECK_POINT SET node_check_point=?,rel_check_point=? WHERE hcode=?\\\\\\\',[$currentTime,$rawCheckPoint,\\\\\\\'{task_hcode}\\\\\\\']) YIELD row RETURN row;\\\'],\\\'\\\',{currentTime:currentTime,rawCheckPoint:rawCheckPoint}) YIELD value WITH value,batchFailedSize,currentTime,rawCheckPoint CALL apoc.load.jdbcUpdate(\\\'{jdbc_task_url}\\\',\\\'UPDATE ONGDB_TASK_CHECK_POINT_LOCK SET task_lock=0 WHERE hcode=?\\\',[\\\'{task_hcode}\\\']) YIELD row AS releaseLock RETURN {releaseLock:releaseLock,value:value,batchFailedSize:batchFailedSize,currentTime:currentTime,rawCheckPoint:rawCheckPoint} AS result;\' AS cypher_task WITH \'task.node.\'+REPLACE(\'node-F-\'+apoc.text.regexGroups(task_hcode,\'((?!=,)([A-Za-z0-9_\u4e00-\u9fa5]+))+\')[1][1],\'-\',\'_\')+\'.\'+REDUCE(hcode=\'\',hcodeStrs IN apoc.text.regexGroups(task_hcode,\'((?!=,)([A-Za-z0-9_\u4e00-\u9fa5]+))+\') | hcode+hcodeStrs[0]+\'_\') AS nodeTaskName,olab.replace(cypher_task,[{raw:\'{merge_label}\',rep:merge_label},{raw:\'{merge_field}\',rep:merge_field},{raw:\'{child_labels}\',rep:child_labels},{raw:\'{jdbc_etl_url}\',rep:jdbc_etl_url},{raw:\'{etl_sql}\',rep:REPLACE(etl_sql,\'\\\'\',\'\\\\\\\\\\\\\\\'\')},{raw:\'{jdbc_task_url}\',rep:jdbc_task_url},{raw:\'{task_hcode}\',rep:task_hcode},{raw:\'{max_min_sql}\',rep:REPLACE(max_min_sql,\'\\\'\',\'\\\\\\\'\')},{raw:\'{ods_batch_size}\',rep:ods_batch_size},{raw:\'{commit_batch_size}\',rep:commit_batch_size}]) AS cypher_task CALL apoc.custom.asProcedure(nodeTaskName,cypher_task,\'WRITE\',[[\'result\',\'MAP\']],[],\'构建FROM-NODE-TASK：{merge_label}-合并节点的数据模型标签;{merge_field}-合并节点的字段;{child_labels}-JsonArrayString数组,空数组传入[] ;{jdbc_etl_url}-支持MySQL、Oracle、SqlServer ;{etl_sql}-抽取数据的SQL;{jdbc_task_url}-任务状态表状态锁表位置默认只支持MySQL ;{task_hcode}-HGRAPHTASK(FromLabel)-[RelaName]->(ToLabel);{max_min_sql}-从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间\') RETURN \'CALL custom.\'+nodeTaskName AS procedure_name,cypher_task',
  'WRITE',
  [['procedure_name','STRING'],['cypher_task','STRING']],
  [['merge_label','STRING'],['merge_field','STRING'],['child_labels','STRING'],['jdbc_etl_url','STRING'],['etl_sql','STRING'],['jdbc_task_url','STRING'],['task_hcode','STRING'],['max_min_sql','STRING'],['ods_batch_size','INTEGER'],['commit_batch_size','INTEGER']],
  '构建FROM-NODE-TASK：{merge_label}-合并节点的数据模型标签;{merge_field}-合并节点的字段;{child_labels}-JsonArrayString数组,空数组传入[] ;{jdbc_etl_url}-支持MySQL、Oracle、SqlServer ;{etl_sql}-抽取数据的SQL ;{jdbc_task_url}-任务状态表状态锁表位置默认只支持MySQL ;{task_hcode}-HGRAPHTASK(FromLabel)-[RelaName]->(ToLabel);{max_min_sql}-从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间;{ods_batch_size}-从源库加载时每个SQL递增的ID尺寸大小【推荐设置为：10000】【task.build.graph.from默认值：10000】;{commit_batch_size}-图数据库批量提交的事物规模大小【推荐设置为：1000、10000】【task.build.graph.from默认值：10000】'
);
```
- 生成task.build.graph.to.batch过程【构建TO-CYPHER-TASK脚本的过程】
```
@param {merge_label}-合并节点的数据模型标签
@param {merge_field}-合并节点的字段
@param {child_labels}-JsonArrayString数组,空数组传入[]
@param {jdbc_etl_url}-支持MySQL、Oracle、SqlServer
@param {etl_sql}-抽取数据的SQL
@param {jdbc_task_url}-任务状态表状态锁表位置默认只支持MySQL
@param {task_hcode}-HGRAPHTASK(FromLabel)-[RelaName]->(ToLabel)
@param {max_min_sql}-从ETL源表加载最大最小自增ID，源库必须要有自增ID【使用该字段升序排序】和自动更新时间
@param {ods_batch_size}-从源库加载时每个SQL递增的ID尺寸大小【推荐设置为：10000】【task.build.graph.to默认值：10000】
@param {commit_batch_size}-图数据库批量提交的事物规模大小【推荐设置为：1000、10000】【task.build.graph.to默认值：10000】
CALL apoc.custom.asProcedure(
  'task.build.graph.to.batch',
  'WITH $merge_label AS merge_label,$merge_field AS merge_field,$child_labels AS child_labels,$jdbc_etl_url AS jdbc_etl_url,$etl_sql AS etl_sql,$jdbc_task_url AS jdbc_task_url,$task_hcode AS task_hcode,$max_min_sql AS max_min_sql,$ods_batch_size AS ods_batch_size,$commit_batch_size AS commit_batch_size,\'CALL apoc.load.jdbcUpdate(\\\'{jdbc_task_url}\\\',\\\'UPDATE ONGDB_TASK_CHECK_POINT_LOCK updateLock INNER JOIN (SELECT task_lock,hcode FROM ONGDB_TASK_CHECK_POINT_LOCK selectLock WHERE task_lock=0 AND hcode=?) selectLock ON updateLock.hcode=selectLock.hcode SET updateLock.task_lock=1\\\',[\\\'{task_hcode}\\\']) YIELD row AS lock WHERE lock.count>0 WITH lock CALL apoc.load.jdbc(\\\'{jdbc_task_url}\\\',\\\'SELECT DATE_FORMAT(rel_check_point,\\\\\\\'%Y-%m-%d %H:%i:%s\\\\\\\') AS check_point,DATE_FORMAT(NOW(),\\\\\\\'%Y-%m-%d %H:%i:%s\\\\\\\') AS currentTime FROM ONGDB_TASK_CHECK_POINT WHERE hcode=?\\\',[\\\'{task_hcode}\\\']) YIELD row WITH apoc.text.join([\\\'\\\\\\\'\\\',row.check_point,\\\'\\\\\\\'\\\'], \\\'\\\') AS check_point,row.currentTime AS currentTime,row.check_point AS rawCheckPoint CALL apoc.load.jdbc(\\\'{jdbc_etl_url}\\\',\\\'{max_min_sql}\\\',[rawCheckPoint]) YIELD row WITH row.min AS min,row.max AS max,check_point,currentTime,rawCheckPoint WITH apoc.coll.union(olab.ids.batch(min,max,{ods_batch_size}),[[0,1]]) AS value,check_point,currentTime,rawCheckPoint UNWIND value AS bactIdList WITH bactIdList[0] AS batchMin,bactIdList[1] AS batchMax,check_point,currentTime,rawCheckPoint WITH REPLACE(\\\'CALL apoc.load.jdbc(\\\\\\\'{jdbc_etl_url}\\\\\\\', \\\\\\\'{etl_sql}\\\\\\\',[check_point,batchMin,batchMax])\\\',\\\'check_point,batchMin,batchMax\\\',check_point+\\\',\\\'+batchMin+\\\',\\\'+batchMax) AS sqlData,currentTime,rawCheckPoint CALL apoc.periodic.iterate(sqlData,\\\'MERGE (n:{merge_label} {--{merge_field}:row.{merge_field}}) SET n+=row WITH n,row CALL apoc.create.addLabels(n,apoc.coll.union(apoc.convert.fromJsonList(row.label),{child_labels})) YIELD node RETURN node\\\', {parallel:false,batchSize:{commit_batch_size},iterateList:true}) YIELD batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations WITH batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations,currentTime,rawCheckPoint WITH SUM(batch.failed) AS batchFailedSize,currentTime,rawCheckPoint CALL apoc.do.case([batchFailedSize>0,\\\'CALL apoc.load.jdbcUpdate(\\\\\\\'{jdbc_task_url}\\\\\\\',\\\\\\\'UPDATE ONGDB_TASK_CHECK_POINT SET node_check_point=rel_check_point WHERE hcode=?\\\\\\\',[\\\\\\\'{task_hcode}\\\\\\\']) YIELD row RETURN row;\\\'],\\\'\\\',{}) YIELD value WITH value,batchFailedSize,currentTime,rawCheckPoint CALL apoc.load.jdbcUpdate(\\\'{jdbc_task_url}\\\',\\\'UPDATE ONGDB_TASK_CHECK_POINT_LOCK SET task_lock=0 WHERE hcode=?\\\',[\\\'{task_hcode}\\\']) YIELD row AS releaseLock RETURN {releaseLock:releaseLock,value:value,batchFailedSize:batchFailedSize,currentTime:currentTime,rawCheckPoint:rawCheckPoint} AS result;\' AS cypher_task WITH \'task.node.\'+REPLACE(\'node-T-\'+apoc.text.regexGroups(task_hcode,\'((?!=,)([A-Za-z0-9_\u4e00-\u9fa5]+))+\')[3][0],\'-\',\'_\')+\'.\'+REDUCE(hcode=\'\',hcodeStrs IN apoc.text.regexGroups(task_hcode,\'((?!=,)([A-Za-z0-9_\u4e00-\u9fa5]+))+\') | hcode+hcodeStrs[0]+\'_\') AS nodeTaskName,olab.replace(cypher_task,[{raw:\'{merge_label}\',rep:merge_label},{raw:\'{merge_field}\',rep:merge_field},{raw:\'{child_labels}\',rep:child_labels},{raw:\'{jdbc_etl_url}\',rep:jdbc_etl_url},{raw:\'{etl_sql}\',rep:REPLACE(etl_sql,\'\\\'\',\'\\\\\\\\\\\\\\\'\')},{raw:\'{jdbc_task_url}\',rep:jdbc_task_url},{raw:\'{task_hcode}\',rep:task_hcode},{raw:\'{max_min_sql}\',rep:REPLACE(max_min_sql,\'\\\'\',\'\\\\\\\'\')},{raw:\'{ods_batch_size}\',rep:ods_batch_size},{raw:\'{commit_batch_size}\',rep:commit_batch_size}]) AS cypher_task CALL apoc.custom.asProcedure(nodeTaskName,cypher_task,\'WRITE\',[[\'result\',\'MAP\']],[],\'构建TO-NODE-TASK：{merge_label}-合并节点的数据模型标签;{merge_field}-合并节点的字段;{child_labels}-JsonArrayString数组,空数组传入[] ;{jdbc_etl_url}-支持MySQL、Oracle、SqlServer ;{etl_sql}-抽取数据的SQL ;{jdbc_task_url}-任务状态表状态锁表位置默认只支持MySQL ;{task_hcode}-HGRAPHTASK(FromLabel)-[RelaName]->(ToLabel);{max_min_sql}-从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间\') RETURN \'CALL custom.\'+nodeTaskName AS procedure_name,cypher_task',
  'WRITE',
  [['procedure_name','STRING'],['cypher_task','STRING']],
  [['merge_label','STRING'],['merge_field','STRING'],['child_labels','STRING'],['jdbc_etl_url','STRING'],['etl_sql','STRING'],['jdbc_task_url','STRING'],['task_hcode','STRING'],['max_min_sql','STRING'],['ods_batch_size','INTEGER'],['commit_batch_size','INTEGER']],
  '构建TO-NODE-TASK：{merge_label}-合并节点的数据模型标签;{merge_field}-合并节点的字段;{child_labels}-JsonArrayString数组,空数组传入[] ;{jdbc_etl_url}-支持MySQL、Oracle、SqlServer ;{etl_sql}-抽取数据的SQL ;{jdbc_task_url}-任务状态表状态锁表位置默认只支持MySQL ;{task_hcode}-HGRAPHTASK(FromLabel)-[RelaName]->(ToLabel);{max_min_sql}-从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间;{ods_batch_size}-从源库加载时每个SQL递增的ID尺寸大小【推荐设置为：10000】【task.build.graph.from默认值：10000】;{commit_batch_size}-图数据库批量提交的事物规模大小【推荐设置为：1000、10000】【task.build.graph.from默认值：10000】'
);
```
- 生成task.build.graph.rel.batch过程【构建REL-CYPHER-TASK脚本的过程】
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
@param {max_min_sql}-从ETL源表加载最大最小自增ID，源库必须要有自增ID【使用该字段升序排序】和自动更新时间
@param {ods_batch_size}-从源库加载时每个SQL递增的ID尺寸大小【推荐设置为：10000】【task.build.graph.rel：10000】
@param {commit_batch_size}-图数据库批量提交的事物规模大小【推荐设置为：1000、10000】【task.build.graph.rel默认值：10000】
CALL apoc.custom.asProcedure(
  'task.build.graph.rel.batch',
  'WITH $matchFrom_label AS matchFrom_label,$matchFrom_field AS matchFrom_field,$matchTo_label AS matchTo_label,$matchTo_field AS matchTo_field,$merge_rel_type AS merge_rel_type,$jdbc_etl_url AS jdbc_etl_url,$etl_sql AS etl_sql,$jdbc_task_url AS jdbc_task_url,$task_hcode AS task_hcode,$max_min_sql AS max_min_sql,$ods_batch_size AS ods_batch_size,$commit_batch_size AS commit_batch_size,\'CALL apoc.load.jdbcUpdate(\\\'{jdbc_task_url}\\\',\\\'UPDATE ONGDB_TASK_CHECK_POINT_LOCK updateLock INNER JOIN (SELECT task_lock,hcode FROM ONGDB_TASK_CHECK_POINT_LOCK selectLock WHERE task_lock=0 AND hcode=?) selectLock ON updateLock.hcode=selectLock.hcode SET updateLock.task_lock=1\\\',[\\\'{task_hcode}\\\']) YIELD row AS lock WHERE lock.count>0 WITH lock CALL apoc.load.jdbc(\\\'{jdbc_task_url}\\\',\\\'SELECT DATE_FORMAT(rel_check_point,\\\\\\\'%Y-%m-%d %H:%i:%s\\\\\\\') AS check_point FROM ONGDB_TASK_CHECK_POINT WHERE hcode=?\\\',[\\\'{task_hcode}\\\']) YIELD row WITH apoc.text.join([\\\'\\\\\\\'\\\',row.check_point,\\\'\\\\\\\'\\\'], \\\'\\\') AS check_point,row.check_point AS rawCheckPoint CALL apoc.load.jdbc(\\\'{jdbc_etl_url}\\\',\\\'{max_min_sql}\\\',[rawCheckPoint]) YIELD row WITH row.min AS min,row.max AS max,check_point,rawCheckPoint WITH apoc.coll.union(olab.ids.batch(min,max,{ods_batch_size}),[[0,1]]) AS value,check_point,rawCheckPoint UNWIND value AS bactIdList WITH bactIdList[0] AS batchMin,bactIdList[1] AS batchMax,check_point,rawCheckPoint WITH REPLACE(\\\'CALL apoc.load.jdbc(\\\\\\\'{jdbc_etl_url}\\\\\\\', \\\\\\\'{etl_sql}\\\\\\\',[check_point,batchMin,batchMax])\\\',\\\'check_point,batchMin,batchMax\\\',check_point+\\\',\\\'+batchMin+\\\',\\\'+batchMax) AS sqlData,rawCheckPoint CALL apoc.periodic.iterate(sqlData,\\\'MATCH (from:{matchFrom_label} {--{matchFrom_field}:row.from}),(to:{matchTo_label} {--{matchTo_field}:row.to}) MERGE (from)-[r:{merge_rel_type}]->(to) SET r+=olab.reset.map(row,[\\\\\\\'from\\\\\\\',\\\\\\\'to\\\\\\\'])\\\', {parallel:false,batchSize:{commit_batch_size},iterateList:true}) YIELD batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations WITH batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations,rawCheckPoint WITH SUM(batch.failed) AS batchFailedSize,rawCheckPoint CALL apoc.do.case([batchFailedSize>0,\\\'CALL apoc.load.jdbcUpdate(\\\\\\\'{jdbc_task_url}\\\\\\\',\\\\\\\'UPDATE ONGDB_TASK_CHECK_POINT SET node_check_point=? WHERE hcode=?\\\\\\\',[$rawCheckPoint,\\\\\\\'{task_hcode}\\\\\\\']) YIELD row RETURN row\\\'],\\\'CALL apoc.load.jdbcUpdate(\\\\\\\'{jdbc_task_url}\\\\\\\',\\\\\\\'UPDATE ONGDB_TASK_CHECK_POINT SET rel_check_point=node_check_point WHERE hcode=?\\\\\\\',[\\\\\\\'{task_hcode}\\\\\\\']) YIELD row RETURN row\\\',{rawCheckPoint:rawCheckPoint}) YIELD value WITH value,batchFailedSize,rawCheckPoint CALL apoc.load.jdbcUpdate(\\\'{jdbc_task_url}\\\',\\\'UPDATE ONGDB_TASK_CHECK_POINT_LOCK SET task_lock=0 WHERE hcode=?\\\',[\\\'{task_hcode}\\\']) YIELD row AS releaseLock RETURN {releaseLock:releaseLock,value:value,batchFailedSize:batchFailedSize,rawCheckPoint:rawCheckPoint} AS result;\' AS cypher_task WITH \'task.rel.\'+REPLACE(apoc.text.regexGroups(task_hcode,\'((?!=,)([A-Za-z0-9_\u4e00-\u9fa5]+))+\')[2][0],\'-\',\'_\')+\'.\'+REDUCE(hcode=\'\',hcodeStrs IN apoc.text.regexGroups(task_hcode,\'((?!=,)([A-Za-z0-9_\u4e00-\u9fa5]+))+\') | hcode+hcodeStrs[0]+\'_\') AS relTaskName,olab.replace(cypher_task,[{raw:\'{matchFrom_label}\',rep:matchFrom_label},{raw:\'{matchFrom_field}\',rep:matchFrom_field},{raw:\'{matchTo_label}\',rep:matchTo_label},{raw:\'{matchTo_field}\',rep:matchTo_field},{raw:\'{merge_rel_type}\',rep:merge_rel_type},{raw:\'{jdbc_etl_url}\',rep:jdbc_etl_url},{raw:\'{etl_sql}\',rep:REPLACE(etl_sql,\'\\\'\',\'\\\\\\\\\\\\\\\'\')},{raw:\'{jdbc_task_url}\',rep:jdbc_task_url},{raw:\'{task_hcode}\',rep:task_hcode},{raw:\'{max_min_sql}\',rep:REPLACE(max_min_sql,\'\\\'\',\'\\\\\\\'\')},{raw:\'{ods_batch_size}\',rep:ods_batch_size},{raw:\'{commit_batch_size}\',rep:commit_batch_size}]) AS cypher_task CALL apoc.custom.asProcedure(relTaskName,cypher_task,\'WRITE\',[[\'result\',\'MAP\']],[],\'构建REL-TASK：{matchFrom_label}-查询FROM节点;{matchFrom_field}-查询FROM节点的唯一性字段;{matchTo_label}-查询TO节点;{matchTo_field}-查询FROM节点的唯一性字段;{merge_rel_type}-合并的关系类型;{jdbc_etl_url}-支持MySQL、Oracle、SqlServer ;{etl_sql}-抽取数据的SQL;{jdbc_task_url}-任务状态表状态锁表位置默认只支持MySQL;{task_hcode}-HGRAPHTASK(FromLabel)-[RelaName]->(ToLabel);{max_min_sql}-从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间\') RETURN \'CALL custom.\'+relTaskName AS procedure_name,cypher_task',
  'WRITE',
  [['procedure_name','STRING'],['cypher_task','STRING']],
  [['matchFrom_label','STRING'],['matchFrom_field','STRING'],['matchTo_label','STRING'],['matchTo_field','STRING'],['merge_rel_type','STRING'],['jdbc_etl_url','STRING'],['etl_sql','STRING'],['jdbc_task_url','STRING'],['task_hcode','STRING'],['max_min_sql','STRING'],['ods_batch_size','INTEGER'],['commit_batch_size','INTEGER']],
  '构建REL-TASK：{matchFrom_label}-查询FROM节点;{matchFrom_field}-查询FROM节点的唯一性字段;{matchTo_label}-查询TO节点;{matchTo_field}-查询FROM节点的唯一性字段;{merge_rel_type}-合并的关系类型;{jdbc_etl_url}-支持MySQL、Oracle、SqlServer ;{etl_sql}-抽取数据的SQL;{jdbc_task_url}-任务状态表状态锁表位置默认只支持MySQL;{task_hcode}-HGRAPHTASK(FromLabel)-[RelaName]->(ToLabel);{max_min_sql}-从ETL源表加载最大最小自增ID，源库必须要有自增ID和自动更新时间;{ods_batch_size}-从源库加载时每个SQL递增的ID尺寸大小【推荐设置为：10000】【task.build.graph.from默认值：10000】;{commit_batch_size}-图数据库批量提交的事物规模大小【推荐设置为：1000、10000】【task.build.graph.from默认值：10000】'
);
```




