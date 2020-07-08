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
- [GraphiQL](https://www.electronjs.org/apps/graphiql)A GUI for editing and testing GraphQL queries and mutations

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
```
http://localhost:7474/graphql/experimental/
{
  person(name: "test", born: 1961) {
    name
    born
  }
}
相当于：MATCH (person:Person) WHERE person.name = 'test' AND person.born = 1961 RETURN person { .name } AS person
```
```
// 执行修改操作
CALL graphql.execute('mutation { createMovie(title:"The Shape of Water", released:2018)}')
// 相当于下面这个CYPHER
CREATE (n:Movie) SET n.title='The Shape of Water',n.released=2018
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

