---
title: ONgDB集成Kafka组件
tags: [ONgDB,Kafka]
author: Yc-Ma
show_author_profile: true
key: 2020-12-30-ONgDB集成Kafka组件
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

### 下载neo4j-streams组件包
```
https://github.com/neo4j-contrib/neo4j-streams/releases/tag/3.5.12
```

### 配置neo4j.conf
```
# kafka zookeeper服务地址和端口
kafka.zookeeper.connect=localhost:2181
# kafka队列服务地址和端口
kafka.bootstrap.servers=localhost:9092
# 开启Sink执行模式
streams.sink.enabled=true
# 定义从TestTopic上读取的消息的处理方法
streams.sink.topic.cypher.TestTopic=MERGE (n:Person {id: event.id}) ON CREATE SET n += event.properties
# 当收到错误消息内容时选择忽略。如果不设置，读取到格式不合法的消息时会暂停。
streams.sink.errors.tolerance=all
# 开启错误日志
streams.sink.errors.log.enable=true
# 在日志中包含消息内容
streams.sink.errors.log.include.messages=true
```
- TestTopic中的消息
```
{  "id":"1",
   "properties":{
      "name":"Smith",
      "dob":19800101
   }
}
```

