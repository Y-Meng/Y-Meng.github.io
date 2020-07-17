# 重缓存的聚合

ES可以缓存常用聚合（例如，用于在网站主页上显示）以获得更快的响应。这些缓存的结果与未缓存聚合返回的结果相同 - 永远不会得到过时的结果。

# 仅返回聚合结果

在很多情况下，需要聚合，但搜索结果不是。对于这些情况，可以通过设置size=0忽略命中。例如：

```json
GET /twitter/_search
{
  "size": 0,
  "aggregations": {
    "my_agg": {
      "terms": {
        "field": "text"
      }
    }
  }
}
```

# 聚合元数据

您可以在请求时将一段元数据与各个聚合相关联，这些聚合将在响应时就地返回。

考虑这个例子，我们希望将蓝色与terms聚合关联起来。

```json
GET /twitter/_search
{
  "size": 0,
  "aggs": {
    "titles": {
      "terms": {
        "field": "title"
      },
      "meta": {
        "color": "blue"
      }
    }
  }
}
```

然后，该元数据将返回到我们的标题terms聚合的位置

```js
{
    "aggregations": {
        "titles": {
            "meta": {
                "color" : "blue"
            },
            "doc_count_error_upper_bound" : 0,
            "sum_other_doc_count" : 0,
            "buckets": [
            ]
        }
    },
    ...
}
```

# 返回聚合的类型

有时需要知道聚合的确切类型才能分析其结果。typed_keys参数可用于更改响应中聚合的名称，以便以其内部类型作为前缀。

```json
GET /twitter/_search?typed_keys // typed_keys更改响应中聚合的名称
{
  "aggregations": {
    "tweets_over_time": {
      "date_histogram": {
        "field": "date",
        "calendar_interval": "year"
      },
      "aggregations": {
        "top_users": {
            "top_hits": {
                "size": 1
            }
        }
     
     }
    }
  }
}
```

结果

```js
{
    "aggregations": {
        "date_histogram#tweets_over_time": { 
            "buckets" : [
                {
                    "key_as_string" : "2009-01-01T00:00:00.000Z",
                    "key" : 1230768000000,
                    "doc_count" : 5,
                    "top_hits#top_users" : {  // 名称改变为type#key
                        "hits" : {
                            "total" : {
                                "value": 5,
                                "relation": "eq"
                            },
                            "max_score" : 1.0,
                            "hits" : [
                                {
                                  "_index": "twitter",
                                  "_type": "_doc",
                                  "_id": "0",
                                  "_score": 1.0,
                                  "_source": {
                                    "date": "2009-11-15T14:12:12",
                                    "message": "trying out Elasticsearch",
                                    "user": "kimchy",
                                    "likes": 0
                                  }
                                }
                            ]
                        }
                    }
                }
            ]
        }
    },
    ...
}
```

