---
title: ONgDB-APOC自定义函数和过程
tags: [ONgDB,Apoc,Neo4j]
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
# 输入输出字段及其类型，格式如下：
[ ['item1','type1'],
  ['item2','type2'],
  …
]
# String::output
# List<List<String>>::inputs
# String::description
```
```
apoc.custom.asFunction(name, statement, outputs, inputs, forceSingle, description)
apoc.custom.declareFunction(signature, statement, forceSingle, description)
```

### 注册一个自定义过程
- 支持返回更复杂的数据类型
```
# 输入输出字段及其类型，格式如下：
[ ['item1','type1'],
  ['item2','type2'],
  …
]
# List<List<String>>::output
# List<List<String>>::inputs
# String::description
# mode过程支持的模式:
    /** This procedure will only perform read operations against the graph */
    READ,
    /** This procedure may perform both read and write operations against the graph */
    WRITE,
    /** This procedure will perform operations against the schema */
    SCHEMA,
    /** This procedure will perform system operations - i.e. not against the graph */
    DBMS,
    /** This procedure will only perform read operations against the graph */
    DEFAULT
```
```
apoc.custom.asProcedure(name, statement, mode, outputs, inputs, description)
apoc.custom.declareProcedure(signature, statement, mode, description)
```

### 输入输出参数支持的数据类型
```
case "ANY": return NTAny;
case "MAP": return NTMap;
case "NODE": return NTNode;
case "REL": return NTRelationship;
case "RELATIONSHIP": return NTRelationship;
case "EDGE": return NTRelationship;
case "PATH": return NTPath;
case "NUMBER": return NTNumber;
case "LONG": return NTInteger;
case "INT": return NTInteger;
case "INTEGER": return NTInteger;
case "FLOAT": return NTFloat;
case "DOUBLE": return NTFloat;
case "BOOL": return NTBoolean;
case "BOOLEAN": return NTBoolean;
case "DATE": return NTDate;
case "TIME": return NTTime;
case "LOCALTIME": return NTLocalTime;
case "DATETIME": return NTDateTime;
case "LOCALDATETIME": return NTLocalDateTime;
case "DURATION": return NTDuration;
case "POINT": return NTPoint;
case "GEO": return NTGeometry;
case "GEOMETRY": return NTGeometry;
case "STRING": return NTString;
case "TEXT": return NTString;
default: return NTString;
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

### 使用案例一
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

### 使用案例二
- 函数与过程发布给其它用户需要使用admin构建
- 基于时间距离长度加总有效持股数
```
WITH 20160630000000 AS endTime,'深圳市投资控股有限公司' AS name
// 过滤出有效持股数的边并且基于时间距离长度找最近时间的持股关系
// 先过滤出有效持股数holdAmontCalc>0的边，然后基于时间距离长度从边中选举出持股数detail
MATCH (n:HORGShareHoldV002 {name:name})-[r:持股]->(m:HORGShareHoldV002) WHERE ANY(detail IN apoc.convert.fromJsonList(r.shareholding_detail) WHERE detail.holdAmontCalc>0) WITH apoc.convert.fromJsonMap(olab.samplingByDate.dis.jsonArray(olab.convert.json(FILTER(detail IN apoc.convert.fromJsonList(r.shareholding_detail) WHERE detail.holdAmontCalc>0)),'defineDate',endTime)) AS detail
// 加总持股数
RETURN SUM(detail.holdAmontCalc) AS holdTotal
```
- 基于时间距离长度加总有效持股数-定义函数
```
CALL apoc.custom.asFunction(
  'sum.hold',
  'WITH $endTime AS endTime,$name AS name MATCH (n:HORGShareHoldV002 {name:name})-[r:持股]->(m:HORGShareHoldV002) WHERE ANY(detail IN apoc.convert.fromJsonList(r.shareholding_detail) WHERE detail.holdAmontCalc>0) WITH apoc.convert.fromJsonMap(olab.samplingByDate.dis.jsonArray(olab.convert.json(FILTER(detail IN apoc.convert.fromJsonList(r.shareholding_detail) WHERE detail.holdAmontCalc>0)),\'defineDate\',endTime)) AS detail RETURN SUM(detail.holdAmontCalc) AS holdTotal',
  'LONG',
  [['endTime','LONG'],['name','STRING']],
   false,
  '有效持股总数'
);
```
- 查看定义的函数
```
CALL dbms.functions() YIELD name,signature,description,roles WHERE name CONTAINS 'sum.hold' RETURN name,signature,description,roles
```
- 使用函数查询有效持股总数
```
RETURN custom.sum.hold(20160630000000,'深圳市投资控股有限公司')
```

### 使用案例三
- 函数与过程发布给其它用户需要使用admin构建
- 查询公司基本信息和实控人
```
// 1、定义公司名称
WITH ['能科股份','科大讯飞','中粮集团有限公司','嘉实基金','武汉当代科技产业集团股份有限公司'] AS nameList
UNWIND nameList AS companyName
// 2、定义GraphQL查询
// # 公司的标准名称name
// # 公司HCODE hcode
// # 标签信息 Tag
// # 来源处代码SrcCompanyCode source # 来源处主表代码-code # 主表名称-table
// legal_person_repr # 法人代表 reg_capital # 注册资本(元) establishment_date # 成立日期 biz_scope # 经营范围 business_major # 主营业务
WITH REPLACE('{\"query\":\"query myConcernedCompany($name: String) { horgByName(name: $name) {name hcode Tag SrcCompanyCode { source code table Hold_Controller { name ratio } } HOrgInfo {legal_person_repr reg_capital establishment_date biz_scope business_major } }}\","variables":{\"name\":\"company-name\"},\"operationName\":\"myConcernedCompany\"}','company-name',companyName) AS query
WITH apoc.convert.fromJsonMap(olab.http.post('http://10.20.13.130/ongdb/graphql',query)) AS result
// 3、获取公司法人、注册资本、成立日期、经营范围信息 并且 获取万得代码和财汇代码
WITH result.data.horgByName[0].hcode AS hcode,result.data.horgByName[0].name AS name,result.data.horgByName[0].Tag AS Tag,result.data.horgByName[0].SrcCompanyCode AS SrcCompanyCode,result.data.horgByName[0].HOrgInfo AS HOrgInfo
// 解析万得和财汇代码 以及 主营业务信息 解析实控人对象
WITH hcode,name,Tag,FILTER(source IN SrcCompanyCode WHERE source.table='TCR0001')[0].code AS ciahuiCode,FILTER(source IN SrcCompanyCode WHERE source.table='CompIntroduction')[0].code AS windCode,HOrgInfo,FILTER(holdCont IN SrcCompanyCode WHERE holdCont.Hold_Controller<>[])[0].Hold_Controller[0] AS holdController
WITH hcode,name,Tag,REPLACE(ciahuiCode,'api','') AS caihuiCode,windCode,HOrgInfo[0].reg_capital AS reg_capital,HOrgInfo[0].legal_person_repr AS legal_person_repr,HOrgInfo[0].business_major AS business_major,HOrgInfo[0].biz_scope AS biz_scope,HOrgInfo[0].establishment_date AS establishment_date,holdController.name AS holdShareController,holdController.ratio AS holdShareControllerRatio
CALL apoc.case([business_major IS NOT NULL,'RETURN $business_major AS scope',biz_scope IS NOT NULL,'RETURN $biz_scope AS scope'],'',{business_major:business_major,biz_scope:biz_scope}) YIELD value
// 经营范围 主体代码 标准名称 标签数组 财汇代码 WIND代码 注册资本 法人 成立日期 实控人 实控人持股比例
RETURN value.scope AS scope ,hcode,name,Tag,caihuiCode,windCode,reg_capital,legal_person_repr,establishment_date,holdShareController,holdShareControllerRatio
```
- 使用过程查询公司基本信息和实控人-构建过程
```
CALL apoc.custom.asProcedure(
  'org.basicinfo',
  'WITH $name AS companyName WITH REPLACE(\'{\\\"query\\\":\\\"query myConcernedCompany($name: String) { horgByName(name: $name) {name hcode Tag SrcCompanyCode { source code table Hold_Controller { name ratio } } HOrgInfo {legal_person_repr reg_capital establishment_date biz_scope business_major } }}\\\","variables":{\\\"name\\\":\\\"company-name\\\"},\\\"operationName\\\":\\\"myConcernedCompany\\\"}\',\'company-name\',companyName) AS query WITH apoc.convert.fromJsonMap(olab.http.post(\'http://10.20.13.130/ongdb/graphql\',query)) AS result WITH result.data.horgByName[0].hcode AS hcode,result.data.horgByName[0].name AS name,result.data.horgByName[0].Tag AS Tag,result.data.horgByName[0].SrcCompanyCode AS SrcCompanyCode,result.data.horgByName[0].HOrgInfo AS HOrgInfo WITH hcode,name,Tag,FILTER(source IN SrcCompanyCode WHERE source.table=\'TCR0001\')[0].code AS ciahuiCode,FILTER(source IN SrcCompanyCode WHERE source.table=\'CompIntroduction\')[0].code AS windCode,HOrgInfo,FILTER(holdCont IN SrcCompanyCode WHERE holdCont.Hold_Controller<>[])[0].Hold_Controller[0] AS holdController WITH hcode,name,Tag,REPLACE(ciahuiCode,\'api\',\'\') AS caihuiCode,windCode,HOrgInfo[0].reg_capital AS reg_capital,HOrgInfo[0].legal_person_repr AS legal_person_repr,HOrgInfo[0].business_major AS business_major,HOrgInfo[0].biz_scope AS biz_scope,HOrgInfo[0].establishment_date AS establishment_date,holdController.name AS holdShareController,holdController.ratio AS holdShareControllerRatio CALL apoc.case([business_major IS NOT NULL,\'RETURN $business_major AS scope\',biz_scope IS NOT NULL,\'RETURN $biz_scope AS scope\'],\'\',{business_major:business_major,biz_scope:biz_scope}) YIELD value RETURN value.scope AS scope ,hcode,name,olab.convert.json(Tag) as Tag,caihuiCode,windCode,reg_capital,legal_person_repr,establishment_date,holdShareController,holdShareControllerRatio',
  'READ',
  [['scope','STRING'],['hcode','STRING'],['name','STRING'],['Tag','STRING'],['caihuiCode','STRING'],['windCode','STRING'],['reg_capital','STRING'],['legal_person_repr','STRING'],['establishment_date','STRING'],['holdShareController','STRING'],['holdShareControllerRatio','STRING']],
  [['name','STRING']],
  '使用过程查询公司基本信息和实控人返回【经营范围 主体代码 标准名称 标签数组 财汇代码 WIND代码 注册资本 法人 成立日期 实控人 实控人持股比例】'
);
```
- 查看定义的过程
```
CALL dbms.procedures() YIELD name,signature,description,roles WHERE name CONTAINS 'org.basicinfo' RETURN name,signature,description,roles
```
- 使用过程查询公司基本信息和实控人
```
WITH ['能科股份','科大讯飞','中粮集团有限公司','嘉实基金','武汉当代科技产业集团股份有限公司'] AS nameList
UNWIND nameList AS companyName
CALL custom.org.basicinfo(companyName) YIELD scope,hcode,name,Tag,caihuiCode,windCode,reg_capital,legal_person_repr,establishment_date,holdShareController,holdShareControllerRatio RETURN scope,hcode,name,apoc.convert.fromJsonList(Tag) AS tag,caihuiCode,windCode,reg_capital,legal_person_repr,establishment_date,holdShareController,holdShareControllerRatio
```

### 使用案例四
- 中文函数名
```
CALL apoc.custom.asFunction(
  '数字打印函数',
  'RETURN $input*2 as answer',
  'long',
  [['input','number']]
);
RETURN custom.数字打印函数(12) AS value;
```
- 中文过程名
```
CALL apoc.custom.asProcedure('数字打印过程','RETURN $input as answer','read',[['answer','number']],[['input','int','42']]);
CALL custom.数字打印过程(12) YIELD answer RETURN answer;
```
### 自定义函数与过程存储位置
- 新增属性
```
apoc.custom
apoc.custom.update
```

