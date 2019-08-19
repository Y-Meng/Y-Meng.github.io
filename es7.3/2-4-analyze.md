# 聚合分析

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/getting-started-aggregations.html)

聚合(Aggregations)提供了从数据中分组和提取统计信息的能力。
考虑聚合最简单的方法是大致将其等同于SQL Group By和SQL聚合函数。
在ElasticSearch中，您可以执行返回点击数的搜索，同时返回与点击数分离的聚合结果。
从某种意义上说，这是非常强大和高效的，您可以运行查询和多个聚合，并一次性获得这两个（或任何一个）操作的结果，
使用简洁和简化的API避免进行过多网络交互。

首先，下面示例按状态对所有帐户进行分组，然后返回按计数降序排序（默认）的前10的州（也是默认值）：
```bash
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      }
    }
  }
}
```
使用SQL上面的聚合类似于：
```sql
SELECT state, COUNT(*) FROM bank GROUP BY state ORDER BY COUNT(*) DESC LIMIT 10;
```

返回结果（部分）如下：
```json
{
  "took": 29,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped" : 0,
    "failed": 0
  },
  "hits" : {
     "total" : {
        "value": 1000,
        "relation": "eq"
     },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "group_by_state" : {
      "doc_count_error_upper_bound": 20,
      "sum_other_doc_count": 770,
      "buckets" : [ {
        "key" : "ID",
        "doc_count" : 27
      }, {
        "key" : "TX",
        "doc_count" : 27
      }, {
        "key" : "AL",
        "doc_count" : 25
      }, {
        "key" : "MD",
        "doc_count" : 25
      }, {
        "key" : "TN",
        "doc_count" : 23
      }, {
        "key" : "MA",
        "doc_count" : 21
      }, {
        "key" : "NC",
        "doc_count" : 21
      }, {
        "key" : "ND",
        "doc_count" : 21
      }, {
        "key" : "ME",
        "doc_count" : 20
      }, {
        "key" : "MO",
        "doc_count" : 20
      } ]
    }
  }
}
```

我们可以看到，ID（爱达荷州）有27个账户，德克萨斯州有27个账户，阿拉巴马州有25个账户，以此类推。

请注意，我们将size=0设置为不显示搜索结果，因为我们只希望在响应中看到聚合结果。

基于前面的聚合，此示例按状态计算平均帐户余额（同样，仅针对按计数降序排序的前10的州）：
```bash
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
```
请注意，我们如何将平均余额average_balance 聚合嵌套在group_by_state 组内。这是所有聚合的通用模式。
可以在聚合中任意嵌套聚合，以从数据中提取所需的透视汇总。

基于前面的聚合，现在让我们按降序对平均余额进行排序：
```bash
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword",
        "order": {
          "average_balance": "desc"
        }
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
```

下面例子演示了我们如何按年龄段（20-29岁、30-39岁和40-49岁）分组，然后按性别分组，最后得到每个年龄段每个性别的平均帐户余额：
```bash
GET /bank/_search
{
  "size": 0,
  "aggs": {
    "group_by_age": {
      "range": {
        "field": "age",
        "ranges": [
          {
            "from": 20,
            "to": 30
          },
          {
            "from": 30,
            "to": 40
          },
          {
            "from": 40,
            "to": 50
          }
        ]
      },
      "aggs": {
        "group_by_gender": {
          "terms": {
            "field": "gender.keyword"
          },
          "aggs": {
            "average_balance": {
              "avg": {
                "field": "balance"
              }
            }
          }
        }
      }
    }
  }
}
```

还有许多其他的聚合功能，我们在这里不详细介绍。如果您想做进一步的实验，
[聚合参考指南文档](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/search-aggregations.html)是一个很好的起点。