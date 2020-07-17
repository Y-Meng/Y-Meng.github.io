# Query和Filter上下文

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/query-filter-context.html)

## 相关度评分

默认情况下，Elasticsearch根据相关性得分对匹配的搜索结果进行排序，相关性得分衡量每个文档与查询的匹配程度。

相关性得分是一个正浮点数，返回到搜索API的“_score”元字段中。分数越高，文档就越相关。虽然每种查询类型可以计算不同的相关分数，但分数计算也取决于查询子句是在查询（query）上下文中运行还是在筛选（filter）上下文中运行。

## Query Context

在查询（query）上下文中，查询子句回答“此文档与此查询子句的匹配程度如何？“除了决定文档是否匹配之外，query子句还计算出_score元字段中的相关分数。

查询上下文在查询子句传递给query参数（如搜索API中的query参数）时生效。

## Filter Context

在筛选（filter）上下文中，查询子句回答“此文档是否与此查询子句匹配”的问题？“答案是简单的是或否，不计算分数。过滤上下文主要用于过滤结构化数据，例如。

* 时间戳是否在2015-2016之间
* 状态字段是不是“已发布”

Elasticsearch会自动缓存常用的过滤器，以提高性能。

当查询子句传递给筛选参数（如bool查询中的Filter或must_not参数、常量_score查询中的Filter参数或筛选聚合）时，Filter context生效。

## 例子

下面是在搜索API的查询和筛选上下文中使用的查询子句的示例。此查询将匹配满足以下所有条件的文档：

* title字段包含单词search。

* 内容字段包含单词elasticsearch。

* status字段包含已发布的确切单词。

* 发布日期字段包含自2015年1月1日起的日期。

```json
GET /_search
{
  "query": { // 1
    "bool": { // 2
      "must": [
        { "match": { "title":   "Search"        }},
        { "match": { "content": "Elasticsearch" }}
      ],
      "filter": [ // 3
        { "term":  { "status": "published" }},
        { "range": { "publish_date": { "gte": "2015-01-01" }}}
      ]
    }
  }
}
```

> 1.查询参数指示查询上下文。
>
> 2.bool和两个match子句用于查询上下文，这意味着它们用于计算每个文档匹配的程度。
>
> 3.filter参数指示筛选器上下文。它的term和range子句用于筛选上下文。它们将筛选出不匹配的文档，但不会影响匹配文档的分数。

为查询上下文中的查询计算的分数表示为单精度浮点数；它们只有24位表示有效位的精度。超过有效位精度的分数计算将转换为精度损失的浮点。

在查询上下文中使用查询子句来处理应影响匹配文档分数的条件（即文档匹配程度），并在筛选上下文中使用所有其他查询子句。