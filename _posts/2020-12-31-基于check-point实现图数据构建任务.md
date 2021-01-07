---
title: 基于check-point实现图数据构建任务
tags: [ONgDB]
author: Yc-Ma
show_author_profile: true
key: 2020-12-31-基于check-point实现图数据构建任务
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

### ONgDB DAG TASK检查点记录表
```
CREATE TABLE `ONGDB_TASK_CHECK_POINT`  (
  `huid` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '自增主键',
  `hcode` varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '代码：HGRAPHTASK(FromLabel)-[RelType]->(ToLabel)',
  `from` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '名称',
  `relationship` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '关联类型',
  `to` varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT 'MSTR_ORG的hcode',
  `node_check_point` datetime(0) NULL DEFAULT '1900-01-01 00:00:00' COMMENT '节点可以获取检查点时间可更改，关系TASK可以获取检查点时间【一个完整的图数据DAG-TASK必须包含节点和关系构建TASK】',
  `rel_check_point` datetime(0) NULL DEFAULT '1900-01-01 00:00:00' COMMENT '保存更新前node_check_point的值',
  `from_update_check` int(2) NOT NULL DEFAULT 0 COMMENT 'from是否更新检查点：0-否，1-是【from和to是一样的标签则不需要使用此判断】',
  `to_update_check` int(2) NOT NULL DEFAULT 0 COMMENT 'to是否更新了检查点：0-否，1-是【from和to是一样的标签则不需要使用此判断】',
  `description` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '对该检查点任务的具体描述',
  `overall_data_split_cypher` text CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NULL COMMENT '同步全量数据的CYPHER：数据分块方案脚本',
  `overall_data_timezone_cypher` text CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NULL COMMENT '同步全量数据的CYPHER：不设置时间范围的同步脚本',
  `hcreatetime` datetime(0) NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `hupdatetime` datetime(0) NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP(0) COMMENT '更新时间',
  `create_by` varchar(24) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '创建人',
  `update_by` varchar(24) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '更新人',
  `hisvalid` int(11) NOT NULL DEFAULT 1 COMMENT '逻辑删除标记：0-无效；1-有效',
  `src` varchar(32) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL DEFAULT 'ART' COMMENT '数据来源标记',
  PRIMARY KEY (`huid`) USING BTREE,
  UNIQUE INDEX `unique_key_02`(`hcode`) USING BTREE COMMENT '唯一索引',
  UNIQUE INDEX `unique_key_01`(`from`, `to`, `relationship`) USING BTREE COMMENT '唯一索引',
  INDEX `updateTime`(`hupdatetime`) USING BTREE,
  INDEX `name`(`from`) USING BTREE,
  INDEX `hisvalid`(`hisvalid`) USING BTREE,
  INDEX `type`(`relationship`) USING BTREE,
  INDEX `check_point`(`node_check_point`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 742715628 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_0900_ai_ci COMMENT = 'ONgDB DAG TASK检查点记录表' ROW_FORMAT = Dynamic;
```

### 关系的开始节点与结束节点标签相同【标签不同且分为两个任务时需要考虑check point的更新时机】
- 节点任务
```
// 获取检查点时间【跑全量数据时修改CHECK_POINT的时间点为最早的一个时间即可】【数据量高于堆内存限制则必须使用数据分块方案】
CALL apoc.load.jdbcParams('jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC','SELECT DATE_FORMAT(node_check_point,\'%Y-%m-%d %H:%i:%s\') AS check_point,DATE_FORMAT(NOW(),\'%Y-%m-%d %H:%i:%s\') AS currentTime FROM ONGDB_TASK_CHECK_POINT WHERE hcode=?',['HGRAPHTASK(HORGGuaranteeV003)-[担保]->(HORGGuaranteeV003)']) YIELD row WITH apoc.text.join(['\'',row.check_point,'\''], '') AS check_point,row.currentTime AS currentTime,row.check_point AS rawCheckPoint
// 定义SQL获取数据方式
WITH REPLACE('CALL apoc.load.jdbc(\'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC\', \'SELECT
hcode,name,credit_code,label,CONVERT(DATE_FORMAT(hcreatetime,\\\'%Y%m%d%H%i%S\\\'),UNSIGNED INTEGER) AS hcreatetime,CONVERT(DATE_FORMAT(hupdatetime,\\\'%Y%m%d%H%i%S\\\'),UNSIGNED INTEGER) AS hupdatetime,hisvalid,create_by,update_by FROM HORGGuaranteeV003 WHERE hupdatetime>=?\',[check_point])','check_point',check_point) AS sqlData,currentTime,rawCheckPoint
// 批量迭代执行节点构建
CALL apoc.periodic.iterate(sqlData,'MERGE (n:HORGGuaranteeV003 {hcode:row.hcode}) SET n+=row WITH n,row CALL apoc.create.addLabels(n,apoc.convert.fromJsonList(row.label)) YIELD node RETURN node', {parallel:false,batchSize:10000,iterateList:true}) YIELD batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations
WITH batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations,currentTime,rawCheckPoint
// 当操作失败的数据包数量小于1时【即操作全部执行成功】则更新检查点【更新node_check_point为系统时间】【rel_check_point设置为更新前node_check_point的值】
WHERE batch.failed<1
CALL apoc.load.jdbcUpdate('jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC','UPDATE ONGDB_TASK_CHECK_POINT SET node_check_point=?,rel_check_point=? WHERE hcode=?',[currentTime,rawCheckPoint,'HGRAPHTASK(HORGGuaranteeV003)-[担保]->(HORGGuaranteeV003)']) YIELD row RETURN row,batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations,currentTime;
```

- 关系任务
```
// 获取检查点时间【跑全量数据时修改CHECK_POINT的时间点为最早的一个时间即可】【数据量高于堆内存限制则必须使用数据分块方案】
CALL apoc.load.jdbcParams('jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC','SELECT DATE_FORMAT(rel_check_point,\'%Y-%m-%d %H:%i:%s\') AS check_point FROM ONGDB_TASK_CHECK_POINT WHERE hcode=?',['HGRAPHTASK(HORGGuaranteeV003)-[担保]->(HORGGuaranteeV003)']) YIELD row WITH apoc.text.join(['\'',row.check_point,'\''], '') AS check_point
// 定义SQL获取数据方式
WITH REPLACE('CALL apoc.load.jdbc(\'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC\', \'SELECT `from`,`to`,guarantee_detail,guarantee_detail_size,CONVERT(DATE_FORMAT(hcreatetime,\\\'%Y%m%d%H%i%S\\\'),UNSIGNED INTEGER) AS hcreatetime,CONVERT(DATE_FORMAT(hupdatetime,\\\'%Y%m%d%H%i%S\\\'),UNSIGNED INTEGER) AS hupdatetime,hisvalid,create_by,update_by FROM HORGGuarantee_GuarV003 WHERE hupdatetime>=?\',[check_point])','check_point',check_point) AS sqlData
// 批量迭代执行节点构建
CALL apoc.periodic.iterate(sqlData,'MATCH (from:HORGGuaranteeV003 {hcode:row.from}),(to:HORGGuaranteeV003 {hcode:row.to}) MERGE (from)-[r:担保]->(to) SET r+={guarantee_detail_size:row.guarantee_detail_size,guarantee_detail:row.guarantee_detail,hupdatetime:row.hupdatetime,hcreatetime:row.hcreatetime,hisvalid:row.hisvalid,create_by:row.create_by,update_by:row.update_by}', {parallel:false,batchSize:10000,iterateList:true}) YIELD batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations RETURN batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations;
```

### 关系的开始节点与结束节点标签相同-数据分块-任务状态回滚【标签不同且分为两个任务时需要考虑check point的更新时机】
>增加数据分块操作与任务状态回滚操作

- 1. 数据分块：控制加载到内存的数据量，避免占用过多堆内存保证图数据库可靠运行
- 2. 任务状态回滚：回滚到构建节点的任务状态，下一次构建节点关系时从回滚点开始操作
- 3. 任务状态同步：关系TASK和节点TASK状态同步
- 节点任务
```
// 获取检查点时间【跑全量数据时修改CHECK_POINT的时间点为最早的一个时间即可】【数据量高于堆内存限制则必须使用数据分块方案】
CALL apoc.load.jdbc('jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC','SELECT DATE_FORMAT(node_check_point,\'%Y-%m-%d %H:%i:%s\') AS check_point,DATE_FORMAT(NOW(),\'%Y-%m-%d %H:%i:%s\') AS currentTime FROM ONGDB_TASK_CHECK_POINT WHERE hcode=?',['HGRAPHTASK(HORGGuaranteeV003)-[担保]->(HORGGuaranteeV003)']) YIELD row WITH apoc.text.join(['\'',row.check_point,'\''], '') AS check_point,row.currentTime AS currentTime,row.check_point AS rawCheckPoint
// 数据分块-从数据库获取检查点之后最大最小自增ID
CALL apoc.load.jdbc('jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC','SELECT MIN(huid) AS min,MAX(huid) AS max FROM HORGGuaranteeV003 WHERE hupdatetime>=?',[rawCheckPoint]) YIELD row WITH row.min AS min,row.max AS max,check_point,currentTime,rawCheckPoint
// 数据分块-从检查点开始按照指定数据块大小执行数据分块
WITH olab.ids.batch(min,max,100000) AS value,check_point,currentTime,rawCheckPoint
UNWIND value AS bactIdList
WITH bactIdList[0] AS batchMin,bactIdList[1] AS batchMax,check_point,currentTime,rawCheckPoint
// 定义SQL获取数据方式
WITH REPLACE('CALL apoc.load.jdbc(\'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC\', \'SELECT
hcode,name,credit_code,label,CONVERT(DATE_FORMAT(hcreatetime,\\\'%Y%m%d%H%i%S\\\'),UNSIGNED INTEGER) AS hcreatetime,CONVERT(DATE_FORMAT(hupdatetime,\\\'%Y%m%d%H%i%S\\\'),UNSIGNED INTEGER) AS hupdatetime,hisvalid,create_by,update_by FROM HORGGuaranteeV003 WHERE hupdatetime>=? AND huid>=? AND huid<=?\',[check_point,batchMin,batchMax])','check_point,batchMin,batchMax',check_point+','+batchMin+','+batchMax) AS sqlData,currentTime,rawCheckPoint
// 批量迭代执行节点构建
CALL apoc.periodic.iterate(sqlData,'MERGE (n:HORGGuaranteeV003 {hcode:row.hcode}) SET n+=row WITH n,row CALL apoc.create.addLabels(n,apoc.convert.fromJsonList(row.label)) YIELD node RETURN node', {parallel:false,batchSize:10000,iterateList:true}) YIELD batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations
WITH batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations,currentTime,rawCheckPoint
// 当操作失败的数据包数量小于1时【即操作全部执行成功】则更新检查点【更新node_check_point为系统时间】【rel_check_point设置为更新前node_check_point的值】
WITH SUM(batch.failed) AS batchFailedSize,currentTime,rawCheckPoint
WHERE batchFailedSize<1
CALL apoc.load.jdbcUpdate('jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC','UPDATE ONGDB_TASK_CHECK_POINT SET node_check_point=?,rel_check_point=? WHERE hcode=?',[currentTime,rawCheckPoint,'HGRAPHTASK(HORGGuaranteeV003)-[担保]->(HORGGuaranteeV003)']) YIELD row RETURN row,batchFailedSize,currentTime,rawCheckPoint;
```
- 关系任务
```
// 获取检查点时间【跑全量数据时修改CHECK_POINT的时间点为最早的一个时间即可】【数据量高于堆内存限制则必须使用数据分块方案】
CALL apoc.load.jdbc('jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC','SELECT DATE_FORMAT(rel_check_point,\'%Y-%m-%d %H:%i:%s\') AS check_point FROM ONGDB_TASK_CHECK_POINT WHERE hcode=?',['HGRAPHTASK(HORGGuaranteeV003)-[担保]->(HORGGuaranteeV003)']) YIELD row WITH apoc.text.join(['\'',row.check_point,'\''], '') AS check_point,row.check_point AS rawCheckPoint
// 数据分块-从数据库获取检查点之后最大最小自增ID
CALL apoc.load.jdbc('jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC','SELECT MIN(huid) AS min,MAX(huid) AS max FROM HORGGuarantee_GuarV003 WHERE hupdatetime>=?',[rawCheckPoint]) YIELD row WITH row.min AS min,row.max AS max,check_point,rawCheckPoint
// 数据分块-从检查点开始按照指定数据块大小执行数据分块
WITH olab.ids.batch(min,max,100000) AS value,check_point,rawCheckPoint
UNWIND value AS bactIdList
WITH bactIdList[0] AS batchMin,bactIdList[1] AS batchMax,check_point,rawCheckPoint
// 定义SQL获取数据方式
WITH REPLACE('CALL apoc.load.jdbc(\'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC\', \'SELECT `from`,`to`,guarantee_detail,guarantee_detail_size,CONVERT(DATE_FORMAT(hcreatetime,\\\'%Y%m%d%H%i%S\\\'),UNSIGNED INTEGER) AS hcreatetime,CONVERT(DATE_FORMAT(hupdatetime,\\\'%Y%m%d%H%i%S\\\'),UNSIGNED INTEGER) AS hupdatetime,hisvalid,create_by,update_by FROM HORGGuarantee_GuarV003 WHERE hupdatetime>=? AND huid>=? AND huid<=?\',[check_point,batchMin,batchMax])','check_point,batchMin,batchMax',check_point+','+batchMin+','+batchMax) AS sqlData,rawCheckPoint
// 批量迭代执行节点构建
CALL apoc.periodic.iterate(sqlData,'MATCH (from:HORGGuaranteeV003 {hcode:row.from}),(to:HORGGuaranteeV003 {hcode:row.to}) MERGE (from)-[r:担保]->(to) SET r+={guarantee_detail_size:row.guarantee_detail_size,guarantee_detail:row.guarantee_detail,hupdatetime:row.hupdatetime,hcreatetime:row.hcreatetime,hisvalid:row.hisvalid,create_by:row.create_by,update_by:row.update_by}', {parallel:false,batchSize:10000,iterateList:true}) YIELD batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations WITH batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations,rawCheckPoint
// batchFailedSize>0则任务状态回滚【当任意一个批量构建关系的任务失败时回滚任务状态】【回滚：设置node_check_point等于当前的rel_check_point】
// batchFailedSize<=0【成功执行则节点TASK与关系TASK状态同步】正常执行则更新rel_check_point=node_check_point
WITH SUM(batch.failed) AS batchFailedSize,rawCheckPoint
CALL apoc.do.case([batchFailedSize>0,'CALL apoc.load.jdbcUpdate(\'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC\',\'UPDATE ONGDB_TASK_CHECK_POINT SET node_check_point=? WHERE hcode=?\',[$rawCheckPoint,\'HGRAPHTASK(HORGGuaranteeV001)-[担保]->(HORGGuaranteeV001)\']) YIELD row RETURN row'],'CALL apoc.load.jdbcUpdate(\'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC\',\'UPDATE ONGDB_TASK_CHECK_POINT SET rel_check_point=node_check_point WHERE hcode=?\',[\'HGRAPHTASK(HORGGuaranteeV001)-[担保]->(HORGGuaranteeV001)\']) YIELD row RETURN row',{rawCheckPoint:rawCheckPoint}) YIELD value RETURN value;
```

### 关系的开始节点与结束节点标签相同-数据分块-任务状态回滚【标签不同且分为两个任务时需要考虑check point的更新时机】
>增加数据分块操作与任务状态回滚操作

- 1. 数据分块：控制加载到内存的数据量，避免占用过多堆内存保证图数据库可靠运行
- 2. 任务状态回滚：回滚到构建节点的任务状态，下一次构建节点关系时从回滚点开始操作
- 3. 任务状态同步：关系TASK-CHECK-POINT和节点TASK-CHECK-POINT状态同步
- 4. 任务状态锁：【图数据构建任务状态锁】【保证某一时刻关系的DAG中TASK运行的唯一性】
```
## 调度系统执行逻辑
>大规模重复并发执行写入操作会导致图数据库服务堆积大量的写入请求导致服务性能下降甚至宕机，因此TASK锁机制设计非常重要，必须保证在同一时刻写入任务不可重复执行；检查点机制的设计保证了数据同步的一致性和完整性；TASK占用过多系统内存【尤其在处理大量数据时】图数据库服务会存在宕机风险，数据分块方案的设计很好的避免了这个问题。
### 异构图数据构建任务执行逻辑
- 每个任务都需要获取锁然后执行数据构建逻辑，不管构建逻辑是否成功执行TASK结束时必须释放锁
- [FROM-NODE-TASK]负责锁的node_check-point更新以及后续任务的rel_check_point同步
- [TO-NODE-TASK]负责node_check-point的回滚
- [REL-TASK]负责node_check-point的回滚和任务状态同步rel_check_point=node_check_point
```
[FROM-NODE-TASK]->[TO-NODE-TASK]->[REL-TASK]
```
### 同构图数据构建任务执行逻辑
- 每个任务都需要获取锁然后执行数据构建逻辑，不管构建逻辑是否成功执行TASK结束时必须释放锁
- [NODE-TASK]负责锁的node_check-point更新以及后续任务的rel_check_point同步
- [REL-TASK]负责node_check-point的回滚和任务状态同步rel_check_point=node_check_point
```
[NODE-TASK]->[REL-TASK]
```
```
```
CREATE TABLE `ONGDB_TASK_CHECK_POINT` (
`huid` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '自增主键',
`hcode` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '代码：HGRAPHTASK(FromLabel)-[RelType]->(ToLabel)',
`from` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '名称',
`relationship` varchar(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '关联类型',
`to` varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT 'MSTR_ORG的hcode',
`node_check_point` datetime DEFAULT '1900-01-01 00:00:00' COMMENT '节点可以获取检查点时间可更改，关系TASK可以获取检查点时间【一个完整的图数据DAG-TASK必须包含节点和关系构建TASK】',
`rel_check_point` datetime DEFAULT '1900-01-01 00:00:00' COMMENT '保存更新前node_check_point的值',
`description` varchar(256) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '对该检查点任务的具体描述',
`overall_data_split_cypher` text CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci COMMENT '同步全量数据的CYPHER：数据分块方案脚本',
`overall_data_timezone_cypher` text CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci COMMENT '同步全量数据的CYPHER：不设置时间范围的同步脚本',
`hcreatetime` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
`hupdatetime` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
`create_by` varchar(24) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '创建人',
`update_by` varchar(24) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '更新人',
`hisvalid` int(11) NOT NULL DEFAULT '1' COMMENT '逻辑删除标记：0-无效；1-有效',
`src` varchar(32) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL DEFAULT 'ART' COMMENT '数据来源标记',
PRIMARY KEY (`huid`) USING BTREE,
UNIQUE KEY `unique_key_02` (`hcode`) USING BTREE COMMENT '唯一索引',
UNIQUE KEY `unique_key_01` (`from`,`to`,`relationship`) USING BTREE COMMENT '唯一索引',
KEY `updateTime` (`hupdatetime`) USING BTREE,
KEY `name` (`from`) USING BTREE,
KEY `hisvalid` (`hisvalid`) USING BTREE,
KEY `type` (`relationship`) USING BTREE,
KEY `check_point` (`node_check_point`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=742715632 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci ROW_FORMAT=DYNAMIC COMMENT='ONgDB DAG TASK检查点记录表';
```
- 节点任务
```
// ===========================获取锁并执行TASK===========================
// 获取任务锁并锁定任务
CALL apoc.load.jdbcUpdate('jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC','UPDATE ONGDB_TASK_CHECK_POINT_LOCK updateLock INNER JOIN (SELECT task_lock,hcode FROM ONGDB_TASK_CHECK_POINT_LOCK selectLock WHERE task_lock=0 AND hcode=?) selectLock ON updateLock.hcode=selectLock.hcode SET updateLock.task_lock=1',['HGRAPHTASK(HORGGuaranteeV001)-[担保]->(HORGGuaranteeV001)']) YIELD row AS lock WHERE lock.count>0 WITH lock
// 获取检查点时间【跑全量数据时修改CHECK_POINT的时间点为最早的一个时间即可】【数据量高于堆内存限制则必须使用数据分块方案】
CALL apoc.load.jdbc('jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC','SELECT DATE_FORMAT(node_check_point,\'%Y-%m-%d %H:%i:%s\') AS check_point,DATE_FORMAT(NOW(),\'%Y-%m-%d %H:%i:%s\') AS currentTime FROM ONGDB_TASK_CHECK_POINT WHERE hcode=?',['HGRAPHTASK(HORGGuaranteeV001)-[担保]->(HORGGuaranteeV001)']) YIELD row WITH apoc.text.join(['\'',row.check_point,'\''], '') AS check_point,row.currentTime AS currentTime,row.check_point AS rawCheckPoint
// 数据分块-从数据库获取检查点之后最大最小自增ID
CALL apoc.load.jdbc('jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC','SELECT MIN(huid) AS min,MAX(huid) AS max FROM HORGGuaranteeV001 WHERE hupdatetime>=?',[rawCheckPoint]) YIELD row WITH row.min AS min,row.max AS max,check_point,currentTime,rawCheckPoint
// 数据分块-从检查点开始按照指定数据块大小执行数据分块【设置一个默认分块，保证锁能顺利释放】
WITH apoc.coll.union(olab.ids.batch(min,max,10000),[[0,1]]) AS value,check_point,currentTime,rawCheckPoint
UNWIND value AS bactIdList
WITH bactIdList[0] AS batchMin,bactIdList[1] AS batchMax,check_point,currentTime,rawCheckPoint
// 定义SQL获取数据方式
WITH REPLACE('CALL apoc.load.jdbc(\'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC\', \'SELECT
hcode,name,credit_code,label,CONVERT(DATE_FORMAT(hcreatetime,\\\'%Y%m%d%H%i%S\\\'),UNSIGNED INTEGER) AS hcreatetime,CONVERT(DATE_FORMAT(hupdatetime,\\\'%Y%m%d%H%i%S\\\'),UNSIGNED INTEGER) AS hupdatetime,hisvalid,create_by,update_by FROM HORGGuaranteeV001 WHERE hupdatetime>=? AND huid>=? AND huid<=?\',[check_point,batchMin,batchMax])','check_point,batchMin,batchMax',check_point+','+batchMin+','+batchMax) AS sqlData,currentTime,rawCheckPoint
// 批量迭代执行节点构建
CALL apoc.periodic.iterate(sqlData,'MERGE (n:HORGGuaranteeV001 {hcode:row.hcode}) SET n+=row WITH n,row CALL apoc.create.addLabels(n,apoc.convert.fromJsonList(row.label)) YIELD node RETURN node', {parallel:false,batchSize:10000,iterateList:true}) YIELD batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations
WITH batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations,currentTime,rawCheckPoint
// 当操作失败的数据包数量小于1时【即操作全部执行成功】则更新检查点【更新node_check_point为系统时间】【rel_check_point设置为更新前node_check_point的值】
WITH SUM(batch.failed) AS batchFailedSize,currentTime,rawCheckPoint
CALL apoc.do.case([batchFailedSize<1,'CALL apoc.load.jdbcUpdate(\'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC\',\'UPDATE ONGDB_TASK_CHECK_POINT SET node_check_point=?,rel_check_point=? WHERE hcode=?\',[$currentTime,$rawCheckPoint,\'HGRAPHTASK(HORGGuaranteeV001)-[担保]->(HORGGuaranteeV001)\']) YIELD row RETURN row;'],'',{currentTime:currentTime,rawCheckPoint:rawCheckPoint}) YIELD value WITH value,batchFailedSize,currentTime,rawCheckPoint
// 释放锁【TASK结束运行释放锁操作】【数据分块处设置一个默认分块，保证释放锁操作顺利执行】
CALL apoc.load.jdbcUpdate('jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC','UPDATE ONGDB_TASK_CHECK_POINT_LOCK SET task_lock=0 WHERE hcode=?',['HGRAPHTASK(HORGGuaranteeV001)-[担保]->(HORGGuaranteeV001)']) YIELD row AS releaseLock RETURN releaseLock,value,batchFailedSize,currentTime,rawCheckPoint;
```
- 关系任务
```
// ===========================获取锁并执行TASK===========================
// 获取任务锁并锁定任务
CALL apoc.load.jdbcUpdate('jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC','UPDATE ONGDB_TASK_CHECK_POINT_LOCK updateLock INNER JOIN (SELECT task_lock,hcode FROM ONGDB_TASK_CHECK_POINT_LOCK selectLock WHERE task_lock=0 AND hcode=?) selectLock ON updateLock.hcode=selectLock.hcode SET updateLock.task_lock=1',['HGRAPHTASK(HORGGuaranteeV001)-[担保]->(HORGGuaranteeV001)']) YIELD row AS lock WHERE lock.count>0 WITH lock
// 获取检查点时间【跑全量数据时修改CHECK_POINT的时间点为最早的一个时间即可】【数据量高于堆内存限制则必须使用数据分块方案】
CALL apoc.load.jdbc('jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC','SELECT DATE_FORMAT(rel_check_point,\'%Y-%m-%d %H:%i:%s\') AS check_point FROM ONGDB_TASK_CHECK_POINT WHERE hcode=?',['HGRAPHTASK(HORGGuaranteeV001)-[担保]->(HORGGuaranteeV001)']) YIELD row WITH apoc.text.join(['\'',row.check_point,'\''], '') AS check_point,row.check_point AS rawCheckPoint
// 数据分块-从数据库获取检查点之后最大最小自增ID
CALL apoc.load.jdbc('jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC','SELECT MIN(huid) AS min,MAX(huid) AS max FROM HORGGuarantee_GuarV001 WHERE hupdatetime>=?',[rawCheckPoint]) YIELD row WITH row.min AS min,row.max AS max,check_point,rawCheckPoint
// 数据分块-从检查点开始按照指定数据块大小执行数据分块【设置一个默认分块，保证锁能顺利释放】
WITH apoc.coll.union(olab.ids.batch(min,max,10000),[[0,1]]) AS value,check_point,rawCheckPoint
UNWIND value AS bactIdList
WITH bactIdList[0] AS batchMin,bactIdList[1] AS batchMax,check_point,rawCheckPoint
// 定义SQL获取数据方式
WITH REPLACE('CALL apoc.load.jdbc(\'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC\', \'SELECT `from`,`to`,guarantee_detail,guarantee_detail_size,CONVERT(DATE_FORMAT(hcreatetime,\\\'%Y%m%d%H%i%S\\\'),UNSIGNED INTEGER) AS hcreatetime,CONVERT(DATE_FORMAT(hupdatetime,\\\'%Y%m%d%H%i%S\\\'),UNSIGNED INTEGER) AS hupdatetime,hisvalid,create_by,update_by FROM HORGGuarantee_GuarV001 WHERE hupdatetime>=? AND huid>=? AND huid<=?\',[check_point,batchMin,batchMax])','check_point,batchMin,batchMax',check_point+','+batchMin+','+batchMax) AS sqlData,rawCheckPoint
// 批量迭代执行节点构建
CALL apoc.periodic.iterate(sqlData,'MATCH (from:HORGGuaranteeV001 {hcode:row.from}),(to:HORGGuaranteeV001 {hcode:row.to}) MERGE (from)-[r:担保]->(to) SET r+={guarantee_detail_size:row.guarantee_detail_size,guarantee_detail:row.guarantee_detail,hupdatetime:row.hupdatetime,hcreatetime:row.hcreatetime,hisvalid:row.hisvalid,create_by:row.create_by,update_by:row.update_by}', {parallel:false,batchSize:10000,iterateList:true}) YIELD batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations WITH batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations,rawCheckPoint
// batchFailedSize>0则任务状态回滚【当任意一个批量构建关系的任务失败时回滚任务状态】【回滚：设置node_check_point等于当前的rel_check_point】
// batchFailedSize<=0【成功执行则节点TASK与关系TASK状态同步】正常执行则更新rel_check_point=node_check_point
WITH SUM(batch.failed) AS batchFailedSize,rawCheckPoint
CALL apoc.do.case([batchFailedSize>0,'CALL apoc.load.jdbcUpdate(\'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC\',\'UPDATE ONGDB_TASK_CHECK_POINT SET node_check_point=? WHERE hcode=?\',[$rawCheckPoint,\'HGRAPHTASK(HORGGuaranteeV001)-[担保]->(HORGGuaranteeV001)\']) YIELD row RETURN row'],'CALL apoc.load.jdbcUpdate(\'jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC\',\'UPDATE ONGDB_TASK_CHECK_POINT SET rel_check_point=node_check_point WHERE hcode=?\',[\'HGRAPHTASK(HORGGuaranteeV001)-[担保]->(HORGGuaranteeV001)\']) YIELD row RETURN row',{rawCheckPoint:rawCheckPoint}) YIELD value WITH value,batchFailedSize,rawCheckPoint
// 释放锁【TASK结束运行释放锁操作】【数据分块处设置一个默认分块，保证释放锁操作顺利执行】
CALL apoc.load.jdbcUpdate('jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC','UPDATE ONGDB_TASK_CHECK_POINT_LOCK SET task_lock=0 WHERE hcode=?',['HGRAPHTASK(HORGGuaranteeV001)-[担保]->(HORGGuaranteeV001)']) YIELD row AS releaseLock RETURN releaseLock,value,batchFailedSize,rawCheckPoint;
```

