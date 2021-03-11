---
title: Cypher实现通过GraphQL更新MySQL中数据
tags: [ONgDB,Cypher,SQL,MySQL,GraphQL]
author: Yc-Ma
show_author_profile: true
key: 2021-03-10-Cypher实现通过GraphQL更新MySQL中数据
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

### 对MySQL数据进行分块
-- 使用场景：MySQL单表数据量过大，例如千万级或者上亿级
-- 数据分块的实现逻辑：根据数据库自增ID对数据进行分块
```
// 获取数据库自增ID的最大最小值
CALL apoc.load.jdbc('jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC','SELECT MIN(huid) AS min,MAX(huid) AS max FROM UIDSRefDataSet') YIELD row WITH row.min AS min,row.max AS max
// 按照指定数据块大小执行数据分块【设置一个默认分块：保证语句执行逻辑正常运行到结束】
WITH apoc.coll.union(olab.ids.batch(min,max,10000),[[0,1]]) AS value
UNWIND value AS bactIdList
WITH bactIdList[0] AS batchMin,bactIdList[1] AS batchMax
RETURN *
```

### 通过自增ID获取源表数据【从源表获取证券代码列表】
```
CALL apoc.load.jdbc('jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC', 'SELECT security_code FROM UIDSRefDataSet WHERE security_code IS NOT NULL AND huid>=? AND huid<=? GROUP BY security_code',[506381471,506382436]) YIELD row RETURN *
```

### 调用GraphQL生成新字段数据
-- 使用Cypher调用GraphQL接口实现新字段数据的生成【这里需要通过证券代码获取公司名称、地域标签、性质标签、统一社会信用代码信息】
```
WITH 'SEC000204503' AS SEC_HCODE
WITH REPLACE('{\"query\": \"{ horgByName(SEC_HCODE: \\\"SEC_HCODE-id\\\",isBond: false, isListed: true, row:1) { name hcode Tag Location MSTR_REGION_ALL { hcode name unit } HOrgInfo { credit_code }  }}\",\"variables\": null}','SEC_HCODE-id',SEC_HCODE) AS query
WITH apoc.convert.fromJsonMap(olab.http.post('http://10.20.13.130/ongdb/graphql',query)) AS result
WITH result.data.horgByName[0].name AS name,result.data.horgByName[0].hcode AS hcode,result.data.horgByName[0].Tag AS Tag,result.data.horgByName[0].Location AS Location,result.data.horgByName[0].MSTR_REGION_ALL AS MSTR_REGION_ALL,result.data.horgByName[0].HOrgInfo[0].credit_code AS credit_code
// 合并三组标签列表
WITH apoc.coll.union(Tag,Location) AS labelTL,REDUCE(mstr_region=[],obj IN MSTR_REGION_ALL | mstr_region+obj.hcode) AS regionHcodes,name,hcode,credit_code
// 排除数据模型的标签，例如['全球']
// WITH apoc.coll.disjunction(apoc.coll.union(labelTL,regionHcodes),['全球']) AS labels,name,hcode,credit_code
WITH FILTER(label IN apoc.coll.union(labelTL,regionHcodes) WHERE label<>'全球') AS labels,name,hcode,credit_code
RETURN name,hcode,credit_code,labels
```
-- 将上面的Cypher封装为一个存储过程
```
CALL apoc.custom.asProcedure(
    'sec.info',
    'WITH $SEC_HCODE AS SEC_HCODE WITH REPLACE(\'{\\\"query\\\": \\\"{ horgByName(SEC_HCODE: \\\\\\"SEC_HCODE-id\\\\\\",isBond: false, isListed: true, row:1) { name hcode Tag Location MSTR_REGION_ALL { hcode name unit } HOrgInfo { credit_code }  }}\\\",\\\"variables\\\": null}\',\'SEC_HCODE-id\',SEC_HCODE) AS query WITH apoc.convert.fromJsonMap(olab.http.post(\'http://10.20.13.130/ongdb/graphql\',query)) AS result WITH result.data.horgByName[0].name AS name,result.data.horgByName[0].hcode AS hcode,result.data.horgByName[0].Tag AS Tag,result.data.horgByName[0].Location AS Location,result.data.horgByName[0].MSTR_REGION_ALL AS MSTR_REGION_ALL,result.data.horgByName[0].HOrgInfo[0].credit_code AS credit_code WITH apoc.coll.union(Tag,Location) AS labelTL,REDUCE(mstr_region=[],obj IN MSTR_REGION_ALL | mstr_region+obj.hcode) AS regionHcodes,name,hcode,credit_code WITH FILTER(label IN apoc.coll.union(labelTL,regionHcodes) WHERE label<>\'全球\') AS labels,name,hcode,credit_code RETURN name,hcode,credit_code,olab.convert.json(labels) AS labels',
    'READ',
    [['name','STRING'],['hcode','STRING'],['credit_code','STRING'],['labels','STRING']],
    [['SEC_HCODE','STRING']],
    '通过证券代码获取公司名称、地域标签、性质标签、统一社会信用代码信息'
);
```
-- 使用存储过程：通过证券代码获取公司名称、地域标签、性质标签、统一社会信用代码信息
```
CALL custom.sec.info('SEC062127226') YIELD name,hcode,credit_code,labels RETURN name,hcode,credit_code,apoc.convert.fromJsonList(labels) AS labels
```

### 更新MySQL
-- 通过Cypher更新MySQL中的数据
```
CALL apoc.load.jdbcUpdate('jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC','UPDATE UIDSRefDataSet SET hcode=?,name=?,credit_code=?,label=? WHERE security_code=?',['亚美能源控股有限公司','HORG749c2c5ca5713fc04baaf83b1d1223a7',null,'["上市公司","山西","香港","沁水县","晋城","HREG00000000257","HREG00000003218","HREG00000003329","开曼群岛","HREG00000000215","HREG00000000259","民营企业","晋城市","民企","HREG00000003358","沁水","山西省"]','SEC002185086']) YIELD row
```

### 案例一【从GraphQL获取数据更新MySQL】：完整实现
```
// 对MySQL数据进行分块
// 获取数据库自增ID的最大最小值
CALL apoc.load.jdbc('jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC','SELECT MIN(huid) AS min,MAX(huid) AS max FROM UIDSRefDataSet') YIELD row WITH row.min AS min,row.max AS max
// 按照指定数据块大小执行数据分块【设置一个默认分块：保证语句执行逻辑正常运行到结束】
WITH apoc.coll.union(olab.ids.batch(min,max,10000),[[0,1]]) AS value
UNWIND value AS bactIdList
WITH bactIdList[0] AS batchMin,bactIdList[1] AS batchMax
// 获取证券代码列表
CALL apoc.load.jdbc('jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC', 'SELECT security_code FROM UIDSRefDataSet WHERE security_code IS NOT NULL AND huid>=? AND huid<=? GROUP BY security_code',[batchMin,batchMax]) YIELD row WITH row.security_code AS security_code
// 调用GraphQL生成新字段数据
CALL custom.sec.info(security_code) YIELD name,hcode,credit_code,labels WITH security_code,name,hcode,credit_code,labels
// 通过Cypher更新MySQL中的数据
CALL apoc.load.jdbcUpdate('jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC','UPDATE UIDSRefDataSet SET hcode=?,name=?,credit_code=?,label=? WHERE security_code=?',[name,hcode,credit_code,labels,security_code]) YIELD row RETURN row;
```

### 案例二【从MySQL获取数据更新到MySQL指定表】：完整实现
```
// 对MySQL数据进行分块
// 获取数据库自增ID的最大最小值
CALL apoc.load.jdbc('jdbc:mysql://contentdb.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/schema_uids?user=datalab_dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC','SELECT MIN(huid) AS min,MAX(huid) AS max FROM DSL_INDICATOR_DEF') YIELD row WITH row.min AS min,row.max AS max
// 按照指定数据块大小执行数据分块【设置一个默认分块：保证语句执行逻辑正常运行到结束】
WITH apoc.coll.union(olab.ids.batch(min,max,10000),[[0,1]]) AS value
UNWIND value AS bactIdList
WITH bactIdList[0] AS batchMin,bactIdList[1] AS batchMax
// 获取原子指标名称与代码列表
CALL apoc.load.jdbc('jdbc:mysql://contentdb.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/schema_uids?user=datalab_dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC', 'SELECT IND_HCODE,IND_CNAME FROM DSL_INDICATOR_DEF WHERE huid>=? AND huid<=?',[batchMin,batchMax]) YIELD row WITH row.IND_HCODE AS IND_HCODE,row.IND_CNAME AS IND_CNAME
// 通过Cypher更新MySQL中的数据
CALL apoc.load.jdbcUpdate('jdbc:mysql://datalab-contentdb-dev.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:3306/analytics_graph_data?user=dev&password=datalabgogo&useUnicode=true&characterEncoding=utf8&serverTimezone=UTC','UPDATE UIDSRefDataSet SET ind_name=? WHERE ind_code=?',[IND_CNAME,IND_HCODE]) YIELD row RETURN row;
```


