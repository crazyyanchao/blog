---
title: Shell发起HTTP请求
tags: [Shell,ONgDB,Neo4j,Cypher,图数据库]
author: Yc-Ma
show_author_profile: true
key: 2021-07-02-Shell发起HTTP请求
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

### Shell - Httppie
```
printf '{
    "statements": [
        {
            "statement": "CALL apoc.load.jdbc('\''jdbc:oracle:thin:ngdp/datalabgogo@ngdpdb-sync-prod.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:1521/ORCL'\'', '\''SELECT FUND_HCODE AS \\"hcode\\",CNAME AS \\"cname\\",CSNAME AS \\"csname\\",HISVALID AS \\"hisvalid\\" FROM INFO.FUND_BASICINFO WHERE HUPDATETIME>=(SELECT sysdate-2 FROM DUAL) ORDER BY HUPDATETIME ASC'\'') YIELD row WITH row CALL apoc.es.put('\''10.20.13.130:9200'\'','\''gh_ind_node_keyword_stck_keyword'\'','\''_doc'\'',row.hcode,'\''refresh=false'\'',row) YIELD value RETURN value.result AS result",
            "resultDataContents": [
                "row"
            ]
        }
    ]
}'| http  --follow --timeout 3600 POST 'http://10.20.13.200/db/data/transaction/commit' \
 Authorization:'Basic b25nZGI6ZGF0YWxhYiVwcm8=' \
 Content-Type:'application/json'
```

### Shell - wget
```
wget --no-check-certificate --quiet \
  --method POST \
  --timeout=0 \
  --header 'Authorization: Basic b25nZGI6ZGF0YWxhYiVwcm8=' \
  --header 'Content-Type: application/json' \
  --body-data '{
    "statements": [
        {
            "statement": "CALL apoc.load.jdbc('\''jdbc:oracle:thin:ngdp/datalabgogo@ngdpdb-sync-prod.crkldnwly6ki.rds.cn-north-1.amazonaws.com.cn:1521/ORCL'\'', '\''SELECT FUND_HCODE AS \"hcode\",CNAME AS \"cname\",CSNAME AS \"csname\",HISVALID AS \"hisvalid\" FROM INFO.FUND_BASICINFO WHERE HUPDATETIME>=(SELECT sysdate-2 FROM DUAL) ORDER BY HUPDATETIME ASC'\'') YIELD row WITH row CALL apoc.es.put('\''10.20.13.130:9200'\'','\''gh_ind_node_keyword_stck_keyword'\'','\''_doc'\'',row.hcode,'\''refresh=false'\'',row) YIELD value RETURN value.result AS result",
            "resultDataContents": [
                "row"
            ]
        }
    ]
}' \
   'http://10.20.13.200/db/data/transaction/commit'
```

### Shell - curl
```
curl -u ongdb:datalab%pro -d '{"statements": [{"statement": "RETURN 2 AS num","resultDataContents": ["row","graph"]}]}' -H 'Content-Type: application/json' -X POST http://10.20.13.200/db/data/transaction/commit
```
