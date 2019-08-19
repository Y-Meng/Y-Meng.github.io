# 开始检索

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/getting-started-search.html)

现在让我们从一些简单的搜索开始。运行搜索有两种基本方法：
一种是通过REST请求URI发送搜索参数，另一种是通过REST请求体(request body)发送搜索参数。
请求体方法允许您更具表现力，还可以用更可读的JSON格式定义搜索。
我们将尝试一个请求URI方法的示例，但在本教程的其余部分中，我们将专门使用请求体方法。

用于搜索的REST API可以从搜索端点访问。此示例返回银行索引中的所有文档：
```bash
GET /bank/_search?q=*&sort=account_number:asc&pretty
```

我们先分析一下搜索请求。我们正在银行索引中搜索（**_search endpoint**），
**q=\***参数指示ElasticSearch匹配索引中的所有文档。
**sort=account_number:asc**参数指示使用每个文档的**account_number**字段按升序对结果进行排序。
同样，**pretty**参数只告诉elasticsearch返回漂亮的JSON结果。 

返回的部分数据如下：
```json
{
  "took" : 63,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
        "value": 1000,
        "relation": "eq"
    },
    "max_score" : null,
    "hits" : [ {
      "_index" : "bank",
      "_type" : "_doc",
      "_id" : "0",
      "sort": [0],
      "_score" : null,
      "_source" : {"account_number":0,"balance":16623,"firstname":"Bradshaw","lastname":"Mckenzie","age":29,"gender":"F","address":"244 Columbus Place","employer":"Euron","email":"bradshawmckenzie@euron.com","city":"Hobucken","state":"CO"}
    }, {
      "_index" : "bank",
      "_type" : "_doc",
      "_id" : "1",
      "sort": [1],
      "_score" : null,
      "_source" : {"account_number":1,"balance":39225,"firstname":"Amber","lastname":"Duke","age":32,"gender":"M","address":"880 Holmes Lane","employer":"Pyrami","email":"amberduke@pyrami.com","city":"Brogan","state":"IL"}
    }, ...
    ]
  }
}
```
在反馈结果中我们可以看到如下几个部分：
* took -ES执行检索消耗的时间，以毫秒为单位
* timed_out -告诉我们检索是否超时
* _shards -告诉我们搜索了多少碎片，以及搜索成功/失败的碎片的计数
* hits -检索结果
* hits.total -包含与搜索条件匹配的文档总数信息的对象
    * hits.total.value -总命中数的值（必须在hits.total.relation上下文中解释）
    * hits.total.relation -hits.total.value是确切的命中计数，在这种情况下，它的值为eq；或者是总命中计数的下限（大于或等于），在这种情况下，它的值为gte
* hits.hits -搜索结果的实际数组（默认为前10个文档）
* hits.sort -每个结果的排序键的排序值（如果按分数排序，则缺少该值）
* hits._score 和 max_score -暂时忽略这两个字段（后面再讲）



hits.total的准确性由请求参数track_total_hits控制，当设置为true时，请求将准确跟踪总命中（"relation"："eq"）。
它默认为10000，这意味着总命中数精确跟踪多达10000个文档。
您可以通过将track_total_hits显式设置为true强制进行精确计数。
有关详细信息，请参阅请求体文档。
 
以下是使用可选请求体方法进行的相同精确搜索： 
```bash
GET /bank/_search
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ]
}
```

这里的区别在于，我们没有在URI中传递q=*，而是向SearchAPI提供一个JSON风格的查询请求体。
我们将在下一节中讨论这个JSON查询。

重要的是需要理解，一旦您返回搜索结果，意味着ElasticSearch完全完成请求，并且不会维护任何类型的服务器端资源或在结果中打开游标。
这与许多其他平台（如SQL）形成了鲜明的对比，在这些平台中，最初可能会提前获得查询结果的一部分子集，
然后如果希望使用某种有状态的服务器端获取（或翻页）其余结果，则必须继续返回服务器游标。

# 查询语言介绍

ElasticSearch提供了一种JSON风格的特定于域的语言，您可以使用它来执行查询。
这被称为查询DSL。查询语言非常全面，乍一看可能很吓人，但实际学习它的最好方法是从几个基本示例开始。

返回我们最近的一个示例，我们执行了下面的查询：
```bash
GET /bank/_search
{
  "query": { "match_all": {} }
}
```

仔细分析上面的内容，查询部分告诉我们查询定义是什么，匹配部分只是我们想要运行的查询类型。
match_all查询只是搜索指定索引中的所有文档。

除了查询参数，我们还可以传递其他参数来影响搜索结果。
在上面部分的示例中，我们传入了sort，这里我们传入了size：
```bash
GET /bank/_search
{
  "query": { "match_all": {} },
  "size": 1
}
```

请注意，如果未指定size大小，则默认为10。 
下面的例子执行“match_all”并返回文档10到19：
```bash
GET /bank/_search
{
  "query": { "match_all": {} },
  "from": 10,
  "size": 10
}
```
“from”参数（基于0）指定从哪个文档索引开始，而“size”参数指定从“from”参数开始返回多少文档。
此功能在实现搜索结果的分页时很有用。请注意，如果未指定From，则默认为0。

此示例执行“match_all”，并按帐户余额降序排序结果，并返回前10个（默认大小）文档。

```bash
GET /bank/_search
{
  "query": { "match_all": {} },
  "sort": { "balance": { "order": "desc" } }
}
```

既然我们已经看到了一些基本的搜索参数，那么让我们深入研究一下查询DSL。
让我们先看看返回的文档字段。默认情况下，作为所有搜索的一部分返回完整的JSON文档。
这被称为“源(source)”（搜索命中中的_source字段）。
如果我们不希望返回整个源文档，我们可以只请求从源中返回几个字段。
 
下面的例子展示了如何只返回两个字段，account_number 和 balance：
```bash
GET /bank/_search
{
  "query": { "match_all": {} },
  "_source": ["account_number", "balance"]
}
```

请注意，上面的示例只是减少了_source字段。
它仍然只返回一个名为_source的字段，但在其中，只包括account_number和balance字段。 
如果你有使用SQL的经验，那么上面的内容在概念上与从字段列表中select SQL语法有些相似。

现在让我们转到查询部分。之前我们已经看到了match_all查询如何用于匹配所有文档。
现在让我们介绍一个名为match的新查询，它可以被视为基本的字段化搜索查询（即针对特定字段或一组字段进行的搜索）。

下面的例子返回account_number为20的文档：
```bash
GET /bank/_search
{
  "query": { "match": { "account_number": 20 } }
}
```

下面的例子返回所有address字段中包含词mill的结果：
```bash
GET /bank/_search
{
  "query": { "match": { "address": "mill" } }
}
```

下面的例子返回address中包含mill或者lane的结果：
```bash
GET /bank/_search
{
  "query": { "match": { "address": "mill lane" } }
}
```

下面的例字是match的变体（match_phrase），返回地址中包含短语“mill lane”的所有结果：
```bash
GET /bank/_search
{
  "query": { "match_phrase": { "address": "mill lane" } }
}
```

现在我们来介绍布尔查询（bool query）。布尔查询允许我们使用布尔逻辑将较小的查询组合成较大的查询。
此示例包含两个匹配查询（match），并返回address中包含“mill”和“lane”的所有结果：
```bash
GET /bank/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "address": "mill" } },
        { "match": { "address": "lane" } }
      ]
    }
  }
}
```

在上面的示例中，bool must子句指定文档被视为匹配项时必须为true的所有查询。
> bool 查询中 must 语义近似 and

相反，下面的例子包含两个匹配查询，返回地址中包含“mill”或“lane”的所有结果：
```bash
GET /bank/_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "address": "mill" } },
        { "match": { "address": "lane" } }
      ]
    }
  }
}
```

在上面的示例中，bool should子句指定一个查询列表，其中任何一个查询必须为true，文档才能被视为匹配。 
> bool 查询中 should 语义近似 or

下面示例包含两个匹配查询，并返回地址中既不包含“mill”也不包含“lane”的所有帐户：
```bash
GET /bank/_search
{
  "query": {
    "bool": {
      "must_not": [
        { "match": { "address": "mill" } },
        { "match": { "address": "lane" } }
      ]
    }
  }
}
```
在上面的示例中，bool must_not子句指定一个查询列表，其中任何一个查询都不能为真，文档才能被视为匹配。

我们可以在bool查询中同时组合must、should和must-not子句。
此外，我们可以在这些bool子句中组合bool查询，以模拟任何复杂的多级布尔逻辑。


下面示例返回40岁但不在ID（AHO）州居住的任何人的所有帐户： 
```bash
GET /bank/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "age": "40" } }
      ],
      "must_not": [
        { "match": { "state": "ID" } }
      ]
    }
  }
}
```

# 执行过滤器（filters）

在上一节中，我们跳过了一个称为文档得分（搜索结果中的_score字段）的小细节。
分数是一个数值，它是文档与我们指定的搜索查询匹配程度的相对度量。
得分越高，文件越相关，得分越低，文件越不相关。

但是查询并不总是需要生成分数，特别是当它们只用于“过滤”文档集时。
ElasticSearch检测到这些情况，并自动优化查询执行，以避免计算无用的分数。

我们在前一节中介绍的bool查询还支持筛选(filter)子句，允许我们使用查询来限制将由其他子句匹配的文档，而不更改计算分数的方式。
我们用一个例子介绍范围查询(range query)，它允许我们按一系列值筛选文档。这通常用于数字或日期筛选。

下面示例使用bool查询返回余额在20000和30000之间（包括20000和30000）的所有帐户。
换言之，我们希望找到余额大于或等于20000且小于或等于30000的账户。 
```bash
GET /bank/_search
{
  "query": {
    "bool": {
      "must": { "match_all": {} },
      "filter": {
        "range": {
          "balance": {
            "gte": 20000,
            "lte": 30000
          }
        }
      }
    }
  }
}
```

仔细分析以上内容，bool查询包含一个match-all查询（query部分）和一个range查询（filter部分）。
我们可以将任何其他查询替换为查询和筛选部分。
在上述情况下，范围查询是完全有意义的，因为属于范围的文档都是完全匹配的，即没有文档比其他文档更相关。



除了match ou all、match、bool和range查询之外，还有许多其他可用的查询类型，我们在这里不讨论它们。
因为我们已经对它们的工作方式有了基本的了解，所以在学习和试验其他查询类型时应用这些知识应该不会太困难。 