---
title: ONgDB集成Kafka组件
tags: [ONgDB,Kafka,Neo4j]
author: Yc-Ma
show_author_profile: true
key: 2020-12-30-ONgDB集成Kafka组件
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

### 下载Kafka
```
wget https://mirrors.tuna.tsinghua.edu.cn/apache/kafka/2.7.0/kafka_2.12-2.7.0.tgz
```

### 解压
```
tar -zxvf kafka_2.12-2.7.0.tgz
```

### 启动
```
./zookeeper-server-start.sh ../config/zookeeper.properties
```
```
./kafka-server-start.sh ../config/server.properties
```

### 测试Kafka
- 创建kafka TestTopic
```
./kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic TestTopic
```
- 查看创建了多少个topic
```
./kafka-topics.sh --list --zookeeper localhost:2181
```
- 通过生产者进行发送消息
```
./kafka-console-producer.sh --broker-list localhost:9092 --topic TestTopic
```
- 消费者来进行接收消息
```
./kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic TestTopic --from-beginning
```

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

### 监听TestTopic并运行指定CYPHER操作
- 通过生产者给TestTopic发送消息【该消息会触发streams.sink.topic.cypher.TestTopic监听的CYPHER】
```
{  "id":"111","properties":{"name":"Smith","dob":19800101}}
```

### 通过生产者发送CYPHER操作给CypherTopic【从消息队列获取CYPHER后执行】
- 新增配置
```
streams.sink.topic.cypher.CypherTopic=CALL apoc.cypher.doIt(event.cypher,{}) YIELD value RETURN value
```
- 创建Topic并启动生产者和消费者
```
./kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic CypherTopic
./kafka-console-producer.sh --broker-list localhost:9092 --topic CypherTopic
./kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic CypherTopic --from-beginning
```
- 通过生产者给CypherTopic发送消息
```
{  "cypher":"MERGE (n:Person {id:'TEST-CYPHER-TOPIC-doit-1'})"}
{  "cypher":"MERGE (n:Person {id:'TEST-CYPHER-TOPIC2-doit-1'})"}
{  "cypher":"MERGE (n:Person {id:'TEST-CYPHER-TOPIC2-doit-2'})"}
```
- 查询通过kafka发送过来的CYPHER创建的数据
```
MATCH (n:Person {id:'TEST-CYPHER-TOPIC2-doit-2'}) RETURN n
```
- 生产消息的存储过程
```
CALL streams.publish('CypherTopic', 'SEND CypherTopic DATA!')
```
- 消费消息的存储过程
```
CALL streams.consume('CypherTopic', {}) YIELD event RETURN event
```


