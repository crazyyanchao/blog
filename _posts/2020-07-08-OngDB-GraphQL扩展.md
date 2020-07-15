---
title: OngDB-GraphQL扩展
tags: [GraphQL,ONgDB]
author: Yc-Ma
show_author_profile: true
key: 2020-07-08-OngDB-GraphQL扩展
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

## 组件安装
### 下载插件
- [GraphiQL](https://www.electronjs.org/apps/graphiql) A GUI for editing and testing GraphQL queries and mutations

### 修改neo4j.conf配置
```
dbms.unmanaged_extension_classes=org.neo4j.graphql=/graphql
graphql.admin.procedures.read=db.*,dbms.components,dbms.queryJ*
graphql.admin.procedures.write=db.create*,dbIndexExplicitFor*
```

## GraphQL使用
### 设置一个GraphQL Schema
```
http://localhost:7474/graphql/idl/
type Movie  {
  title: String!
  released: Int
  actors: [Person] @relation(name:"ACTED_IN",direction:IN)
}
type Person {
  name: String!
  born: Int
  movies: [Movie] @relation(name:"ACTED_IN")
}
```

```
CALL graphql.idl('
type Movie  {
  title: String!
  released: Int
  tagline: String
  actors: [Person] @relation(name:"ACTED_IN",direction:IN)
}
type Person {
  name: String!
  born: Int
  movies: [Movie] @relation(name:"ACTED_IN")
}
')
```

### 查看GraphQL Schema
```
CALL graphql.schema()
```

### 运行查询
- 使用CYPHER运行GraphQL查询数据

```
CALL graphql.query('{ Person(name:"test") {name born movies { title released tagline } }}')
CALL graphql.query('{ Person(born: 1961) { born } }')
```

```
WITH 'query ($year:Long,$limit:Int) { Movie(released: $year, first:$limit) { title, actors {name} } }' as query
CALL graphql.query(query,{year:1995,limit:5}) YIELD result
UNWIND result.Movie as movie
RETURN movie.title, [a IN movie.actors | a.name] as actors
```

- 使用HTTP接口运行GraphQL查询数据

```
http://localhost:7474/graphql/experimental/
{
  person(name: "test", born: 1961) {
    name
    born
  }
}
OR
query {person(name:"test", born:1961) {name, born}}
相当于：MATCH (person:Person) WHERE person.name = 'test' AND person.born = 1961 RETURN person.name,person.born AS person
```

- 使用CYPHER运行GraphQL修改数据

```
// 执行修改操作
CALL graphql.execute('mutation { createMovie(title:"The Shape of Water", released:2018)}')
// 相当于下面这个CYPHER
CREATE (n:Movie) SET n.title='The Shape of Water',n.released=2018
```

- 使用HTTP接口运行GraphQL修改数据

```
http://localhost:7474/graphql/
mutation {
  createMovie(title: "The Shape of Water", released: 2018)
}
```

### 重置GraphQL Schema
```
CALL graphql.reset()
```

### 查看已有GraphQL Schema的字符串表示
```
RETURN graphql.getIdl()
```

### 查看远程的GraphQL Schema
```
CALL graphql.introspect("https://api.github.com/graphql",{Authorization:"bearer d8xxxxxxxxxxxxxxxxxxxxxxx"})
```

## 参考
- [Neo4j-GraphQL Extension](https://github.com/neo4j-graphql/neo4j-graphql)
- [JVM Library to translate GraphQL queries and mutations to Neo4j’s Cypher](https://github.com/neo4j-graphql/neo4j-graphql-java)

```
CALL graphql.idl('
type Column  {
  name: String!
}
')
query {column(name:"ods.test_table.c1") {name}}

mutation {
  createColumn(name: "The Shape of Water")
}
```

## OTHER
- GraphQL Schema
```
type Movie {
  title: String!
  released: Int
  actors: [Person] @relation(name: "ACTED_IN", direction: IN)
}
type Person {
  name: String!
  born: Int
  movies: [Movie] @relation(name: "ACTED_IN")
}
```

- mutation

```
mutation {
  createMovie(title: "The Shape of Water", released: 2018)
}
```

- query

```
query {
  person(name: "Kelly McGillis", born: 1957) {
    name
    born
  }
}
```

## HORGShareHold
- GraphQL Schema
```
type HORGShareHold {
  name: String!
  hcode: String!
  hORGShareHoldCount: Int @cypher(statement: "MATCH (:HORGShareHold) RETURN count(*)")
  hOrgShareHoldRelCount: String! @cypher(statement: "MATCH p=(n:HORGShareHold)-[r]->(m:HORGShareHold) RETURN COUNT(p)")
  hold: [HORGShareHold] @relation(name: "HOLD")
}
type HORGGuarantee {
  name: String!
  hcode: String!
  hORGGuaranteeCount: Int @cypher(statement: "MATCH (:HORGGuarantee) RETURN count(*)")
  hORGGuaranteeRelCount: String! @cypher(statement: "MATCH p=(n:HORGGuarantee)-[r]->(m:HORGGuarantee) RETURN COUNT(p)")
}
```

- mutation
```
mutation {
  createMovie(title: "The Shape of Water", released: 2018)
}
```

- [1]query
```
query($name1:String,$name2:String)
{
  # =================查询持股网络的节点和统计值=================
  # 查询名称匹配的公司
  hORGShareHold(name: $name1) {
    # 返回节点名称
    name
    # 返回节点的HCODE
    hcode
    # 返回这个标签下的节点统计值
    hORGShareHoldCount
    # 返回HORGShareHold标签关联的持股关系数量
    hOrgShareHoldRelCount
  }
  # 查询名称匹配的公司
  hORGShareHold(name: $name2) {
    # 返回节点名称
    name
    # 返回节点的HCODE
    hcode
    # 返回这个标签下的节点统计值
    hORGShareHoldCount
    # 返回HORGShareHold标签关联的持股关系数量
    hOrgShareHoldRelCount
  }
   # =================查询担保网络的节点和统计值=================
  # 查询名称匹配的公司
  hORGGuarantee(name: "长春万润房地产有限公司") {
    # 返回节点名称
    name
    # 返回节点的HCODE
    hcode
    # 返回这个标签下的节点统计值
    hORGGuaranteeCount
    # 返回HORGGuarantee标签关联的持股关系数量
    hORGGuaranteeRelCount
  }
}
{
  "name1":"深圳市金田房地产开发公司汕头公司",
  "name2":"华美加国际(集团)有限公司",
  "username":"ongdb",
  "password":"ongdb%dev"
}
```

- [2]query
```
query($name1:String)
{
  # =================查询持股网络是关联关系=================
  # 查询名称匹配的公司
  hORGShareHold(name: $name1) {
    # 返回节点名称
    name
    # 返回节点的HCODE
    hcode
    hold{
     # 返回节点名称
     name
     # 返回节点的HCODE
     hcode
    }
  }
}
{
  "name1":"深圳市金田房地产开发公司汕头公司",
  "username":"ongdb",
  "password":"ongdb%dev"
}
```

