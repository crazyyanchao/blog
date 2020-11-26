---
title: Python调用ONgDB-HTTP接口
tags: [Python,ONgDB]
author: Yc-Ma
show_author_profile: true
key: 2020-11-26-Python调用ONgDB-HTTP接口
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

### 安装包
```
pip install ipython-cypher
```

### ipython-cypher
```
# -*- coding: utf-8 -*-
"""
Created on Tue Oct 27 15:13:46 2020
@author: XXX
"""
import cypher
con = "http://ongdb:datalab%pro@datalab.ongdb.http.server/db/data"
#con = "http://ongdb:datalab%pro@10.20.13.200:7474/db/data"
query = """
    MATCH p=()-->()-->() RETURN p LIMIT 100
"""
comp_list = cypher.run(query, conn=con)
```


