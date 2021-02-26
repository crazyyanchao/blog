---
title: 自动生成CYPHER-TASK脚本二
tags: [ONgDB,Cypher,SQL,MySQL]
author: Yc-Ma
show_author_profile: true
key: 2021-02-04-自动生成CYPHER-TASK脚本二
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

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
```
WITH 'HBondOrg' AS merge_label,'hcode' AS merge_field,NULL AS child_labels,'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC' AS jdbc_etl_url,'SELECT org_hcode AS hcode,org_name AS name,credit_code,label,CONVERT(DATE_FORMAT(hcreatetime,\'%Y%m%d%H%i%S\'),UNSIGNED INTEGER) AS hcreatetime,CONVERT(DATE_FORMAT(hupdatetime,\'%Y%m%d%H%i%S\'),UNSIGNED INTEGER) AS hupdatetime,hisvalid,create_by,update_by FROM HBondOrg WHERE hupdatetime>=? AND huid>=? AND huid<=? AND type=\'发行证券\'' AS etl_sql,'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC' AS jdbc_task_url,'HGRAPHTASK(HBondOrg)-[发行证券]->(HEventBond)' AS task_hcode,'SELECT MIN(huid) AS min,MAX(huid) AS max FROM HBondOrg WHERE hupdatetime>=? AND type=\'发行证券\'' AS max_min_sql

WITH
custom.node.mysql.etlSql([rawFieldName,mapingFieldName]) AS etl_sql,
custom.node.mysql.maxMinAutoId({tabel},{hupdatetimeFieldName},'filterWhereSqlStr') AS max_min_sql
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



