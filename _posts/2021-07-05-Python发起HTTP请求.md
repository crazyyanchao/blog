---
title: Python发起HTTP请求
tags: [Python,Shell,ONgDB,Neo4j,Cypher,图数据库]
author: Yc-Ma
show_author_profile: true
key: 2021-07-05-Python发起HTTP请求
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

### Python - Requests
```
import requests

url = "http://10.20.13.200/db/data/transaction/commit"

payload="{\r\n    \"statements\": [\r\n        {\r\n            \"statement\": \"CALL apoc.load.jdbc('jdbc:oracle:thin:nfdp/testlabgogo@nfdpdb-sync-prod.crkldnwly6ki.rds.cn-north-1.alibaba.com.cn:1521/ORCL', 'SELECT FUND_HCODE AS \\\"hcode\\\",CNAME AS \\\"cname\\\",CSNAME AS \\\"csname\\\",HISVALID AS \\\"hisvalid\\\" FROM INFO.FUND_BASICINFO WHERE HUPDATETIME>=(SELECT sysdate-2 FROM DUAL) ORDER BY HUPDATETIME ASC') YIELD row WITH row CALL apoc.es.put('10.20.13.130:9200','gh_ind_node_keyword_stck_keyword','_doc',row.hcode,'refresh=false',row) YIELD value RETURN value.result AS result\",\r\n            \"resultDataContents\": [\r\n                \"row\"\r\n            ]\r\n        }\r\n    ]\r\n}"
headers = {
  'Authorization': 'Basic b25nZGI6ZGF0YWxhYiVwcm8=',
  'Content-Type': 'application/json'
}

response = requests.request("POST", url, headers=headers, data=payload)

print(response.text)
```


