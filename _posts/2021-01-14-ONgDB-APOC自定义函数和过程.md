---
title: ONgDB-APOC自定义函数和过程
tags: [ONgDB,Apoc]
author: Yc-Ma
show_author_profile: true
key: 2021-01-14-ONgDB-APOC自定义函数和过程
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

## 定义说明
>APOC提供相关过程来创建用户自定义的函数和过程。这些函数和过程实际上是参数化的Cypher语言查询，类似宏（Macro）的概念。

### 查看自定义函数和过程
```
CALL dbms.functions() YIELD name,signature,description,roles WHERE name CONTAINS 'custom' RETURN name,signature,description,roles
UNION
CALL dbms.procedures() YIELD name,signature,description,roles WHERE name CONTAINS 'custom' RETURN name,signature,description,roles
```

### 查看构建自定义函数和过程的存储过程
```
CALL dbms.functions() YIELD name,signature,description,roles WHERE name CONTAINS 'apoc.custom' RETURN name,signature,description,roles
UNION
CALL dbms.procedures() YIELD name,signature,description,roles WHERE name CONTAINS 'apoc.custom' RETURN name,signature,description,roles
```

### 注册一个自定义函数
```
输入输出字段及其类型，格式如下：
[ ['item1','type1'],
  ['item2','type2'],
  …
]
mode查询执行模式，取值'read'或'write'
apoc.custom.asFunction(name, statement, outputs, inputs, forceSingle, description)
apoc.custom.declareFunction(signature, statement, forceSingle, description)
```

### 注册一个自定义过程
```
apoc.custom.asProcedure(name, statement, mode, outputs, inputs, description)
apoc.custom.declareProcedure(signature, statement, mode, description)
```

### 查看函数和过程清单
```
apoc.custom.list()
```

### 删除函数
```
apoc.custom.removeFunction(name, type)
```

### 删除过程
```
apoc.custom.removeProcedure(name)
```

### 使用案例
- 自定义函数
```
CALL apoc.custom.asFunction(
  'double',
  'RETURN $input*2 as answer',
  'long',
  [['input','number']]
);
RETURN custom.double(12) AS value;
```
- 自定义过程
```
CALL apoc.custom.asProcedure('answer','RETURN $input as answer','read',[['answer','number']],[['input','int','42']])
CALL custom.answer() YIELD answer RETURN answer
CALL custom.answer(13) YIELD answer RETURN answer
```

