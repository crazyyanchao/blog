---
title: ONgDB图数据库集成Elasticsearch
tags: [ONgDB,Elasticsearch,Neo4j]
author: Yc-Ma
show_author_profile: true
key: 2020-06-14-ONgDB图数据库集成Elasticsearch
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

>在ONgDB中主要有模式索引和全文索引，可以支持一些基本的查询，但是在大量数据的时候都会有性能瓶颈。此外全文索引功能还不可以支持数值类型的检索。使用插件集成es之后，可以让图库支持更加复杂的检索并保证高性能。图数据库事务的CRUD操作都会同步到es，保持数据的一致。

## 插件下载
>此插件支持索引中文标签，下载之后按照说明在neo4j.conf中配置对应选项。【创建好mapping之后再启动图库】

- [ongdb-lab-elasticsearch](https://github.com/ongdb-contrib/ongdb-lab-elasticsearch/releases)
- [插件集成使用说用-EN](https://github.com/ongdb-contrib/ongdb-lab-elasticsearch)
- [插件集成使用说用-CN](https://github.com/ongdb-contrib/ongdb-lab-elasticsearch/blob/3.5.17/README-CN.md)

## 索引的示例配置【在neo4j.conf中配置】
```
#********************************************************************
## ONgDB Elasticsearch Integration
##********************************************************************
## elasticsearch.discovery=true
elasticsearch.host_name=https://localhost:9200
elasticsearch.index_spec=pre_org_cn_node:PRE公司中文名称(name,hcode,pcode,hupdatetime,cluster_id)
```

## 创建索引mapping样例
> 索引mapping不建议使用自动生成的字段类，需要自定义设置，用es的put接口生成。图库中的标签与es的索引类型相对应，索引名使用英文定义并且不要和索引集群的其它索引重名。

- 【Mapping支持不区分大小写查询】put https://localhost:9200/pre_org_cn_node

```json
{
  "settings": {
    "number_of_replicas": 1,
    "number_of_shards": 6,
    "refresh_interval": "1s",
    "translog": {
      "flush_threshold_size": "1.6gb"
    },
    "merge": {
      "scheduler": {
        "max_thread_count": "1"
      }
    },
    "index": {
      "routing": {
        "allocation": {
          "total_shards_per_node": "3"
        }
      }
    },
    "analysis": {
      "normalizer": {
        "my_normalizer": {
          "type": "custom",
          "filter": [
            "lowercase",
            "asciifolding"
          ]
        }
      }
    }
  },
  "mappings": {
    "PRE公司中文名称": {
      "dynamic": "false",
      "_source": {
        "enabled": true
      },
      "properties": {
        "name": {
          "index": "not_analyzed",
          "store": true,
          "type": "keyword",
          "normalizer": "my_normalizer"
        },
        "hcode": {
          "index": "not_analyzed",
          "store": true,
          "type": "keyword",
          "normalizer": "my_normalizer"
        },
        "pcode": {
          "index": "not_analyzed",
          "store": true,
          "type": "keyword",
          "normalizer": "my_normalizer"
        },
        "hupdatetime": {
          "index": "not_analyzed",
          "store": true,
          "type": "long"
        },
        "cluster_id": {
          "index": "not_analyzed",
          "store": true,
          "type": "integer"
        }
      }
    }
  }
}
```

## 初始化数据导入
如果图库中已经有数据则按照下列说明对应操作。无数据则不用再手动同步，插件的事务同步机制会自动同步数据。
>创建好mapping之后重启图库。例如初始化导入‘PRE公司中文名称’标签下节点的数据，可以用以下过程：【进行索引数据强制事务提交：n.is_indices=1】【设置一个属性的原因就是为了单独触发事务更新机制，让数据同步到elasticsearch集群】

```
CALL apoc.periodic.iterate('MATCH (n:PRE公司中文名称) RETURN n','WITH {n} AS n SET n.is_indices=1', {parallel:false,batchSize:1000}) YIELD  batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations RETURN batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations
```

>如果索引被删除了需要重新初始化导入，可以先删除‘is_indices’属性再执行一遍上述过程。【REMOVE【‘is_indices’】之后，重新强制提交事务】

```
CALL apoc.periodic.iterate('MATCH (n:PRE公司中文名称) RETURN n','WITH {n} AS n REMOVE n.is_indices', {parallel:false,batchSize:1000}) YIELD  batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations RETURN batches,total,timeTaken,committedOperations,failedOperations,failedBatches,retries,errorMessages,batch,operations
```

## 使用cypher存储过程在es中检索

例如下面这个查询，看起来并不复杂数据量小并发不大的情况下响应速度也非常快。但是对应标签下数据量上千万之后性能就比较差了。【次查询实现返回前一百个公司聚簇】
```
MATCH (n:PREClusterHeart公司) WITH n.cluster_id AS clusterId MATCH (m:PRE公司中文名称) WHERE m.cluster_id=clusterId RETURN clusterId AS master,COUNT(m) AS slaveCount,COLLECT(id(m)+'-'+m.name) AS slaves ORDER BY slaveCount DESC LIMIT 100
```
使用es优化上述查询：
```
"CALL apoc.es.query('https://localhost:9200','pre_org_cn_node','',null,{size:0,query:{bool:{}},aggs:{cluster_id:{terms:{field:'cluster_id',size:10,shard_size:100000,order:{_count:'DESC'}},aggs:{topHitsData:{top_hits:{size:100,_source:{includes:['name']}}}}},field_count:{cardinality:{precision_threshold:100000,field:'cluster_id'}}}}) yield value WITH value.aggregations.cluster_id.buckets AS buckets UNWIND buckets AS topHitsData RETURN topHitsData.key AS master,topHitsData.doc_count AS slaveCount,topHitsData.topHitsData.hits.hits AS slaves"
}
```
## 其它常用查询
>如果使用了中文标签，下列查询中GET请求与POST请求都不可以使用中文索引类型去查询【创建mapping时是可以支持中文的】，所以在使用GET请求时设置TYPE为_all，POST时TYPE为空即可。【所以上述提到不要有重名的索引】

- 查看索引统计信息
```
CALL apoc.es.stats('localhost:9200') YIELD value RETURN value
CALL apoc.es.stats('localhost') YIELD value RETURN value
```
- 通过节点ID查询公司名称
```
CALL apoc.es.get('localhost','pre_org_cn_node','_all','15097731',null,null) yield value
```
- 通过节点ID查询公司名称【设置需要返回的字段和不需要返回的字段】
```
CALL apoc.es.get('localhost','pre_org_cn_node','_all','15097731',{_source_include:'name,pcode',_source_exclude:'description'},null) yield value
CALL apoc.es.get('localhost','pre_org_cn_node','_all','15097731','_source_include=name&_source_exclude=description',null) yield value
CALL apoc.es.get('localhost','pre_org_cn_node','_all','15097731',{_source_include:'name'},null) yield value
CALL apoc.es.get('localhost:9200','pre_org_cn_node','_all','15097731',{_source_include:'name'},null) yield value
```
- 使用_search接口搜索文档【随机返回一些doc】
```
CALL apoc.es.query('localhost','pre_org_cn_node','',null,null) yield value
```
- 通过name参数搜索/_search?q=name:?
```
CALL apoc.es.query('localhost','pre_org_cn_node','','q=name:吉林白山航空发展股份有限公司',null) yield value
```
- 通过name参数搜索【设置分页参数】/_search?q=name:get
```
CALL apoc.es.query('localhost','pre_org_cn_node','','size=1&scroll=1m&_source=true&q=name:吉林白山航空发展股份有限公司',null) yield value
```
- 使用elasticsearch-dsl查询数据
```
CALL apoc.es.query('localhost','pre_org_cn_node','',null,{query: {match: {name: '吉林白山航空发展股份有限公司'}}}) yield value
```
- 解析查询返回doc【返回hists中的节点数据】
```
CALL apoc.es.query('localhost','pre_org_cn_node','',null,{query: {match: {name: '吉林白山航空发展股份有限公司'}}}) yield value with value.hits.hits AS hits WITH hits
UNWIND hits AS hit
RETURN hit._source AS nodeData
```
- 解析查询返回doc【返回name】
```
CALL apoc.es.get('localhost','pre_org_cn_node','_all','15097731',{_source_include:'name'},null) yield value with value._source.name AS name return name
```
- 通过节点ID查询查询与当前节点名称相同的节点
```
// 这个查询看起来复杂度不高，但是当数据库资源占用较多时也非常耗时基本都在秒级以上
MATCH (n:PRE公司中文名称) WHERE ID(n)=" + idNNodeId + " WITH n
MATCH (m:PRE公司中文名称) WHERE n<>m AND m.name=n.name RETURN ID(m) AS idM
```
```
// 优化上述查询，在es中实现
WITH 'https://localhost:9200' AS esUrl
WITH 'pre_org_cn_node' AS indexName,esUrl
CALL apoc.es.get(esUrl,indexName,'_all','10232180',null,null) YIELD value WITH value._source.name AS name,indexName,esUrl
CALL apoc.es.query(esUrl,indexName,'',null,{query: {term: {name: name}}}) yield value with value.hits.hits AS hits
UNWIND hits AS hit
WITH hit._source.id AS nodeId
MATCH (n:PRE公司中文名称) WHERE ID(n)<>10232180 AND ID(n)=TOINT(nodeId) RETURN n
```
- 统计name字段并返回前10个结果
```
CALL apoc.es.query('localhost','pre_org_cn_node','',null,{aggs: {field: {terms: {field: 'name',size: 10}}}}) yield value WITH value.aggregations.field.buckets AS aggregations RETURN aggregations
```
- 统计name字段并返回前1个结果【并在图中展示这些节点】【使用节点ID到图中检索节点不要用名称，因为涉及到检索的大小写转换问题】
```
WITH 'localhost' AS esUrl
WITH 'pre_org_cn_node' AS indexName,esUrl
CALL apoc.es.query(esUrl,indexName,'',null,{aggs: {field: {terms: {field: 'name',size: 1}}}}) yield value
WITH value.aggregations.field.buckets AS aggregations,esUrl,indexName
UNWIND aggregations AS keyCount
WITH keyCount.key AS name,esUrl,indexName
CALL apoc.es.query(esUrl,indexName,'','q=name:'+name,null) yield value WITH value.hits.hits AS hits
UNWIND hits AS hit
WITH hit._source.id AS nodeId
MATCH (n:`PRE公司中文名称`) WHERE ID(n)=TOINT(nodeId) RETURN n
```
- 使用with设置索引配置
```
WITH 'https://localhost:9200' AS esUrl
WITH 'pre_org_cn_node' AS indexName,esUrl
CALL apoc.es.query(esUrl,indexName,'',null,{aggs: {field: {terms: {field: 'name',size: 1}}}}) yield value
WITH value.aggregations.field.buckets AS aggregations,esUrl,indexName
UNWIND aggregations AS keyCount
WITH keyCount.key AS name,esUrl,indexName
CALL apoc.es.query(esUrl,indexName,'','q=name:'+name,null) yield value WITH value.hits.hits AS hits
UNWIND hits AS hit
WITH hit._source.id AS nodeId
MATCH (n:`PRE公司中文名称`) WHERE ID(n)=TOINT(nodeId) RETURN n
```
- 使用unwind设置索引配置
```
UNWIND [{esUrl:'https://localhost:9200',indexName:'pre_org_cn_node'}] AS configuration
WITH configuration.esUrl AS esUrl,configuration.indexName AS indexName
CALL apoc.es.query(esUrl,indexName,'',null,{aggs: {field: {terms: {field: 'name',size: 1}}}}) yield value
WITH value.aggregations.field.buckets AS aggregations,esUrl,indexName
UNWIND aggregations AS keyCount
WITH keyCount.key AS name,esUrl,indexName
CALL apoc.es.query(esUrl,indexName,'','q=name:'+name,null) yield value WITH value.hits.hits AS hits
UNWIND hits AS hit
WITH hit._source.id AS nodeId
MATCH (n:`PRE公司中文名称`) WHERE ID(n)=TOINT(nodeId) RETURN n
```
## 参考资料
[Neo4j and Elasticsearch](https://neo4j.com/developer/elastic-search/)
