---
title: Elasticsearch常用查询
tags: [Elasticsearch]
---

Here's the table of contents:
1. TOC
{:toc}

## 支持不区分大小写的mapping
```
{
  "settings": {
    "number_of_replicas": 1,
    "number_of_shards": 6,
    "refresh_interval": "1s",
    "translog": {
      "flush_threshold_size": "1.6gb"
    },
    "merge": {
      "scheduler": {
        "max_thread_count": "1"
      }
    },
    "index": {
      "routing": {
        "allocation": {
          "total_shards_per_node": "3"
        }
      }
    },
    "analysis": {
      "normalizer": {
        "my_normalizer": {
          "type": "custom",
          "filter": [
            "lowercase",
            "asciifolding"
          ]
        }
      }
    }
  },
  "mappings": {
    "PRE公司中文名称": {
      "dynamic": "false",
      "_source": {
        "enabled": true
      },
      "properties": {
        "name": {
          "index": "not_analyzed",
          "store": true,
          "type": "keyword",
          "normalizer": "my_normalizer"
        },
        "hcode": {
          "index": "not_analyzed",
          "store": true,
          "type": "keyword",
          "normalizer": "my_normalizer"
        },
        "pcode": {
          "index": "not_analyzed",
          "store": true,
          "type": "keyword",
          "normalizer": "my_normalizer"
        },
        "hupdatetime": {
          "index": "not_analyzed",
          "store": true,
          "type": "long"
        },
        "cluster_id": {
          "index": "not_analyzed",
          "store": true,
          "type": "integer"
        }
      }
    }
  }
}
```

## keyword字段完全匹配
```
{
  "_source": [
    "pcode",
    "name"
  ],
  "query": {
    "bool": {
      "must": {
        "term": {
          "pcode": "windL8PR59Hvvs"
        }
      }
    }
  }
}
```

## text字段完全匹配【必须包含】
```
{
  "_source": [
    "content",
    "hcode",
    "pcode"
  ],
  "query": {
    "bool": {
      "must": {
        "query_string": {
          "query": "+content:(\"北京晋轩科技咨询有限公司\")"
        }
      }
    }
  }
}
```

## 精确匹配【从执行分析的字段】
```
{
  "_source": [
    "name",
    "hcode",
    "pcode"
  ],
  "query": {
    "bool": {
      "must": {
        "query_string": {
          "query": "+name:(\"北京晋轩科技咨询有限公司\")"
        }
      }
    }
  }
}

```
```
{
  "_source": [
    "content",
    "crawler_time"
  ],
  "query": {
    "bool": {
      "must": [
        {
          "query_string": {
            "query": "+id:(237501394733502460 OR 237501394733502461)"
          }
        }
      ],
      "filter": {
        "bool": {
          "must": [
            {
              "range": {
                "crawler_time": {
                  "gte": "2019-03-01 09:56:04",
                  "lte": "2019-05-23 09:56:04"
                }
              }
            }
          ]
        }
      }
    }
  }
}
```
```
{
  "_source": [
    "content",
    "crawler_time"
  ],
  "query": {
    "bool": {
      "must": {
        "terms": {
          "id": [237501393831731200,237501396138594300]
        }
      },
      "filter": {
        "bool": {
          "must": [
            {
              "range": {
                "crawler_time": {
                  "gte": "2019-03-01 09:56:04",
                  "lte": "2019-05-23 09:56:04"
                }
              }
            }
          ]
        }
      }
    }
  }
}

```
```
{
  "_source": [
    "area_list"
  ],
  "query": {
    "bool": {
      "must": [
        {
          "query_string": {
            "query": "+area_list:(长春 OR 北京)"
          }
        }
      ],
      "filter": {
        "bool": {
          "must": [
            {
              "range": {
                "pubtimeAll": {
                  "gte": "2019-05-18 09:56:04",
                  "lte": "2019-05-28 09:56:04"
                }
              }
            }
          ]
        }
      }
    }
  }
}
```

