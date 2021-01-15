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

## 已有的TASK命名规范
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

====================================待定补充====================================

## 定义的TASK过程命名规范
### 同构图
- 节点过程
```
CALL custom.task.node.node-HORGGuaranteeV003.HORGGuaranteeV003-F-担保-T-HORGGuaranteeV003()
```
- 关系过程
```
CALL custom.task.node.rel-担保.HORGGuaranteeV003-F-担保-T-HORGGuaranteeV003()
```
### 异构图
- 节点过程
```
CALL custom.task.node.node-F-HBondOrg.HBondOrg-F-发行证券-T-HEventBond()
CALL custom.task.node.node-T-HEventBond.HBondOrg-F-发行证券-T-HEventBond()
```
- 关系过程
```
CALL custom.task.node.rel-发行证券.HBondOrg-F-发行证券-T-HEventBond()
```

## 异构图-TASK脚本生成
- 以一个异构图为例：(HBondOrg)-[:发行证券]->(HEventBond)
- 使用CALL custom.task.build.graph.*()生成下面三个过程
```
CALL custom.task.build.graph.from() -> CALL custom.task.node.node-F-HBondOrg.HBondOrg-F-发行证券-T-HEventBond()
CALL custom.task.build.graph.to() -> CALL custom.task.node.node-T-HEventBond.HBondOrg-F-发行证券-T-HEventBond()
CALL custom.task.build.graph.rel() -> CALL custom.task.node.rel-发行证券.HBondOrg-F-发行证券-T-HEventBond()
```
### 生成custom.task.build.graph.*过程
- CALL apoc.custom.asProcedure生成CALL custom.task.build.graph.*
```
CALL apoc.custom.asProcedure(
  'task.build.graph.from',
  '',
  'WRITE',
  [['label','STRING'],['hcode','STRING']],
  [['label','STRING'],['jsonarray-labels','STRING'],['merge-field','STRING'],['properties','MAP'],['jdbc-etl-url','STRING'],['task-hcode','STRING'],['jdbc-task-url','STRING'],['TASK_CHECK_POINT_LOCK_TABLE','STRING'],['TASK_CHECK_POINT_TABLE','STRING']],
  '构建FROM-NODE-TASK'
);
```
```
CALL apoc.custom.asProcedure(
  'task.build.graph.to',
  '',
  'WRITE',
  [['scope','STRING'],['hcode','STRING']],
  [['name','STRING']],
  '构建FROM-NODE-TASK'
);
```
```
CALL apoc.custom.asProcedure(
  'task.build.graph.rel',
  '',
  'WRITE',
  [['scope','STRING'],['hcode','STRING']],
  [['name','STRING']],
  '构建FROM-NODE-TASK'
);
```
### 节点CYPHER-TASK自动生成
```

```
### 关系CYPHER-TASK自动生成
```

```

## 同构图-TASK脚本生成



