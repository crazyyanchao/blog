---
title: Elasticsearch-7.5.1测试向量检索功能
tags: [Elasticsearch,向量数据库]
author: Yc-Ma
show_author_profile: true
key: 2023-06-01-Elasticsearch-7.5.1测试向量检索功能
pageview: true
---

Here's the table of contents:
1. TOC
{:toc}

# Elasticsearch-7.5.1测试向量检索功能

```
# https://github.com/elastic/elasticsearch/blob/e8c382f89553e3a7aaafa88a5934288c1192acdc/docs/reference/vectors/vector-functions.asciidoc

# 在向量函数计算过程中，对所有匹配的文档进行线性扫描。因此，查询预计时间会随着匹配文档的数量线性增长。出于这个原因，建议使用查询参数限制匹配文档的数量。

# cosinessimilarity -计算余弦相似度
# dotProduct -计算点积
# L1 norm -计算L1距离
# l2norm -计算L2距离
# doc[<field>].vectorValue - 以浮点数数组的形式返回向量的值
# doc[<field>].magnitude - 返回向量的大小

# Elasticsearch-7.5.1
# 创建索引设置向量字段
PUT index3
{
  "mappings": {
    "properties": {
      "my_vector": {
        "type": "dense_vector",
        "dims": 3
      },
      "my_text" : {
        "type" : "keyword"
      }
    }
  }
}

# 写入数据
PUT index3/_doc/1
{
  "my_text" : "text1",
  "my_vector" : [0.5, 10, 6]
}

PUT index3/_doc/2
{
  "my_text" : "text2",
  "my_vector" : [-0.5, 10, 10]
}

# 搜索向量字段 - 余弦相似度：cosineSimilarity
POST index3/_search
{
  "query": {
    "script_score": {
      "query": {
        "match_all": {}
      },
      "script": {
        "source": "cosineSimilarity(params.queryVector, doc['my_vector'])+1.0",
        "params": {
          "queryVector": [-0.5, 10, 6]
        }
      }
    }
  }
}

# cosinessimilarity函数计算给定查询向量和文档向量之间的余弦相似性度量
# 搜索向量字段 - 计算点积：dotProduct
POST index3/_search
{
  "query": {
    "script_score": {
      "query": {
        "match_all": {}
      },
      "script": {
        "source": """
        double value = dotProduct(params.queryVector,doc['my_vector']);
        return sigmoid(1, Math.E, -value);
        """,
        "params": {
          "queryVector": [
            -0.5,
            10,
            6
          ]
        }
      }
    }
  }
}

# dotProduct函数计算给定查询向量和文档向量之间的点积度量
# 搜索向量字段 - 计算点积：dotProduct
POST index3/_search
{
  "query": {
    "script_score": {
      "query": {
        "match_all": {}
      },
      "script": {
        "source": """
        double value = dotProduct(params.queryVector,doc['my_vector']);
        return sigmoid(1, Math.E, -value);
        """,
        "params": {
          "queryVector": [
            -0.5,
            10,
            6
          ]
        }
      }
    }
  }
}


# l1norm函数计算给定查询向量和文档向量之间的L1距离(曼哈顿距离)
# 搜索向量字段 - 曼哈顿距离：l1norm
POST index3/_search
{
  "query": {
    "script_score": {
      "query": {
        "match_all": {}
      },
      "script": {
        "source":"1 / (1 + l1norm(params.queryVector, doc['my_vector']))",
        "params": {
          "queryVector": [-0.5, 10, 6]
        }
      }
    }
  }
}

# l2norm函数计算给定查询向量和文档向量之间的L2距离(欧几里德距离)
# 搜索向量字段 - 欧几里得距离：l2norm
POST index3/_search
{
  "query": {
    "script_score": {
      "query": {
        "match_all": {}
      },
      "script": {
        "source": "1 / (1 + l2norm(params.queryVector, doc['my_vector']))",
        "params": {
          "queryVector": [
            -0.5,
            10,
            6
          ]
        }
      }
    }
  }
}



# 使用函数自定义实现向量余弦相似度计算
# 搜索向量字段 - 自定义余弦相似度
# ES 中向量检索 doc[<field>].vectorValue 函数是在 Elasticsearch 7.8.0 版本开始支持的，在ES 7.5.1 或 7.8.0 以下版本会运行失败
POST index3/_search
{
  "query": {
    "script_score": {
      "query": {
        "match_all": {}
      },
      "script": {
        "source": """
          float[] v = doc['my_vector'].vectorValue;
          float vm = doc['my_vector'].magnitude;
          float dotProduct = 0;
          for (int i = 0; i < v.length; i++) {
            dotProduct += v[i] * params.queryVector[i];
          }
          return dotProduct / (vm * (float) params.queryVectorMag);
        """,
        "params": {
          "queryVector": [
            -0.5,
            10,
            6
          ],
          "queryVectorMag": 5.25357
        }
      }
    }
  }
}
```

