# 复合查询

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/compound-queries.html)

复合查询可以包装其他复合查询或叶查询，可以组合其结果和分数，更改其行为，或者从查询切换到筛选上下文。

复合查询主要包含以下几种：

* bool query

  用于组合多个叶或复合查询子句的默认查询，例如：must、should、must_not 或 filter子句。must子句和should子句的分数组合为 ： 匹配的子句越多，越好，而must_not子句和filter子句是在filter上下文中执行的。

* boosting query

  返回与positive查询匹配的文档，但减少与negative查询匹配的文档的分数。

* constant_score query

  包装另一个查询，但在筛选上下文中执行的查询。所有匹配的文档都给出相同的“常量”_score。

* dis_max query

  接受多个查询并返回与任何查询子句匹配的任何文档的查询。当bool查询合并所有匹配查询的分数时，dis_max查询使用单个最佳匹配查询子句的分数。

* function_score query

  使用函数修改主查询返回的分数，以考虑流行度、最近度、距离或使用脚本实现的自定义算法等因素。

## Boolean

匹配与其他查询的布尔组合匹配的文档的查询。bool查询映射到Lucene BooleanQuery。它使用一个或多个布尔子句构建，每个子句都有一个类型化的引用。出现类型为：

| 关键字   | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| must     | 子句必须出现在匹配的文档中，并将有助于得分。                 |
| filter   | 子句必须出现在匹配的文档中。但是，与必须忽略查询的分数不同。Filter子句在**Filter**上下文中执行，这意味着将忽略评分，并考虑使用子句进行缓存。 |
| should   | 子句应出现在匹配的文档中。                                   |
| must_not | 子句不能出现在匹配的文档中。子句在**Filter**上下文中执行，这意味着将忽略评分，并考虑使用子句进行缓存。由于忽略评分，因此返回所有文档的0分 |

bool查询采用更多匹配是更好的方法，因此来自每个匹配must或should子句的分数将被添加到一起，以提供每个文档的最终分数。

```json
POST _search
{
  "query": {
    "bool" : {
      "must" : {
        "term" : { "user" : "kimchy" }
      },
      "filter": {
        "term" : { "tag" : "tech" }
      },
      "must_not" : {
        "range" : {
          "age" : { "gte" : 10, "lte" : 20 }
        }
      },
      "should" : [
        { "term" : { "tag" : "wow" } },
        { "term" : { "tag" : "elasticsearch" } }
      ],
      "minimum_should_match" : 1,
      "boost" : 1.0
    }
  }
}
```

### bool.filter的评分

在筛选元素下指定的查询对评分没有影响 - 评分返回为0。分数只受指定查询的影响。例如，以下三个查询都返回status字段包含active一词的所有文档。

由于未指定评分查询，第一个查询为所有文档分配0分：

```json
GET _search
{
  "query": {
    "bool": {
      "filter": {
        "term": {
          "status": "active"
        }
      }
    }
  }
}
```

此bool查询有一个match_all查询，该查询为所有文档分配1.0的分数。

```json
GET _search
{
  "query": {
    "bool": {
      "must": {
        "match_all": {}
      },
      "filter": {
        "term": {
          "status": "active"
        }
      }
    }
  }
}
```

这个常量得分查询的行为方式与上面的第二个示例完全相同。constant_score查询为筛选器匹配的所有文档分配1.0的分数。

```json
GET _search
{
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "status": "active"
        }
      }
    }
  }
}
```

### 使用命名查询来查看子句的匹配情况

如果需要知道bool查询中的哪些子句与查询返回的文档匹配，可以使用命名查询为每个子句指定一个名称。

## Boosting

返回与positive查询匹配的文档，同时减少也与negative查询匹配的文档的相关性分数。

可以使用boosting查询降级某些文档，而不将其从搜索结果中排除。

例如：

```json
GET /_search
{
    "query": {
        "boosting" : {
            "positive" : {
                "term" : {
                    "text" : "apple"
                }
            },
            "negative" : {
                 "term" : {
                     "text" : "pie tart fruit crumble tree"
                }
            },
            "negative_boost" : 0.5
        }
    }
}
```

### boosting顶级参数

* positive

  必填，要运行的查询。任何返回的文档都必须与此查询匹配。

* negative

  必填，用于降低匹配文档的相关性得分的查询。

  如果返回的文档与正查询和此查询匹配，则boosting查询将按如下方式计算文档的最终相关性分数：

  1. 从正查询中获取原始相关性分数。
  2. 将分数乘以negative_boost值。

* negative_boost

  必填，介于0和1.0之间的浮点数，用于降低与负查询匹配的文档的相关性分数。

## Constant Score

包装筛选器查询并返回每个匹配文档，相关度得分等于boost参数值。

```json
GET /_search
{
    "query": {
        "constant_score" : {
            "filter" : {
                "term" : { "user" : "kimchy"}
            },
            "boost" : 1.2
        }
    }
}
```

### constant_score顶级参数

* filter

  必填，要运行的筛选查询。任何返回的文档都必须与此查询匹配。

  筛选查询不计算相关性分数。为了提高性能，Elasticsearch会自动缓存常用的筛选查询。

* boost

  选填，浮点数，用作与筛选查询匹配的每个文档的恒定相关性分数。默认为1.0。

## Disjunction Max Query

回与一个或多个包装查询（称为查询子句或子句）匹配的文档。

如果返回的文档与多个查询子句匹配，dis_max查询将从任何匹配子句中为文档分配最高的相关性分数，并为任何其他匹配的子查询分配断路的增量。

您可以使用dis_max在映射了不同提升因子的字段中搜索term。

例子

```json
GET /_search
{
    "query": {
        "dis_max" : {
            "queries" : [
                { "term" : { "title" : "Quick pets" }},
                { "term" : { "body" : "Quick pets" }}
            ],
            "tie_breaker" : 0.7
        }
    }
}
```

### dis_max顶级参数

* querys

  必填，包含一个或多个查询子句。返回的文档必须与这些查询中的一个或多个匹配。如果一个文档匹配多个查询，Elasticsearch将使用最高的相关性得分。

* tie_breaker

  选填，介于0和1.0之间的浮点数，用于增加匹配多个查询子句的文档的相关性分数。默认为0.0。

  可以使用tie_breaker值为在多个字段中包含同一term的文档分配比仅在这些多个字段中的最佳字段中包含此term的文档更高的相关性分数，而不会将此与多个字段中两个不同term的更好情况混淆。

  如果一个文档匹配多个子句，dis_max查询将按如下方式计算文档的相关性分数：

  1. 从得分最高的匹配子句中获取相关性得分。
  2. 将任何其他匹配子句的得分乘以tie_breaker值。
  3. 把最高的分数加到乘以的分数上。

  如果tie_breaker值大于0.0，则所有匹配的子句都计数，但得分最高的子句计数最多。

## Function Score

function_score允许您修改通过查询检索的文档的分数。例如，如果score函数在计算上成本根号，并且足以计算过滤后的文档集上的分数，则这一点非常有用。

要使用function_score，用户必须定义一个查询和一个或多个函数，这些函数为查询返回的每个文档计算新的分数。

unction_score只能与以下一个函数一起使用：

```json
GET /_search
{
    "query": {
        "function_score": {
            "query": { "match_all": {} },
            "boost": "5",
            "random_score": {}, 
            "boost_mode":"multiply"
        }
    }
}
```

此外，可以组合多个功能。在这种情况下，只有当文档与给定的筛选查询匹配时，才可以选择应用该函数

```json
GET /_search
{
    "query": {
        "function_score": {
          "query": { "match_all": {} },
          "boost": "5", 
          "functions": [
              {
                  "filter": { "match": { "test": "bar" } },
                  "random_score": {}, 
                  "weight": 23
              },
              {
                  "filter": { "match": { "test": "cat" } },
                  "weight": 42
              }
          ],
          "max_boost": 42,
          "score_mode": "max",
          "boost_mode": "multiply",
          "min_score" : 42
        }
    }
}
```

每个函数的过滤查询产生的分数无关紧要。如果没有为函数提供筛选器，这相当于指定“match_all”：{}

首先，每个文档由定义的函数进行评分。参数score_mode指定如何组合计算的分数：

| score_mode | 说明                       |
| ---------- | -------------------------- |
| multiply   | 分数相乘                   |
| sum        | 分数相加                   |
| avg        | 分数平均值                 |
| first      | 具有匹配筛选器的第一个函数 |
| max        | 分数最大值                 |
| min        | 分数最小值                 |

由于分数可以在不同的尺度上（例如，衰减函数在0到1之间，但场值因子是任意的），而且有时函数对分数的影响不同也是可取的，因此可以使用用户定义的权重调整每个函数的分数。权重可以在函数数组中为每个函数定义（如上图所示），并与相应函数计算的分数相乘。如果权重是在没有任何其他函数声明的情况下给定的，则权重充当一个函数，只返回权重。

假如score_mode设置为avg个人得分将通过加权平均值进行组合。例如，如果两个函数返回分数1和2，且它们各自的权重为3和4，则它们的分数将合并为（1*3+2*4）/（3+4）而不是（1*3+2*4）/2。

通过设置max_boost参数，可以限制新分数不超过某个限制。max_boost的默认值是FLT_max。

新计算的分数与查询的分数合并。参数boost_mode定义如下：

| boost_mode | 说明                       |
| ---------- | -------------------------- |
| multiply   | 查询得分和函数分数相乘     |
| sum        | 查询得分和函数分数相加     |
| avg        | 查询得分和函数分数平均值   |
| replace    | 仅使用函数分，舍弃查询分数 |
| max        | 查询得分和函数分数最大值   |
| min        | 查询得分和函数分数最小值   |

默认情况下，修改分数不会更改匹配的文档。要排除不符合特定分数阈值的文档，可以将min_score参数设置为所需的分数阈值。

function_score query提供了几种类型的打分函数。

- [`script_score`](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/query-dsl-function-score-query.html#function-script-score)
- [`weight`](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/query-dsl-function-score-query.html#function-weight)
- [`random_score`](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/query-dsl-function-score-query.html#function-random)
- [`field_value_factor`](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/query-dsl-function-score-query.html#function-field-value-factor)
- [decay functions](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/query-dsl-function-score-query.html#function-decay): `gauss`, `linear`, `exp`

### script_score

script_score函数允许您包装另一个查询，并通过使用脚本表达式从文档中的其他数值字段值派生的计算自定义该查询的评分。下面是一个简单的示例：

```json
GET /_search
{
    "query": {
        "function_score": {
            "query": {
                "match": { "message": "elasticsearch" }
            },
            "script_score" : {
                "script" : {
                  "source": "Math.log(2 + doc['likes'].value)" // 结果不能为负
                }
            }
        }
    }
}
```

在不同的脚本字段值和表达式之上，_score 脚本参数可用于基于包装查询检索分数。

脚本编译被缓存以加快执行速度。如果脚本包含需要考虑的参数，则最好重用同一脚本，并为其提供参数：

```json
GET /_search
{
    "query": {
        "function_score": {
            "query": {
                "match": { "message": "elasticsearch" }
            },
            "script_score" : {
                "script" : {
                    "params": {
                        "a": 5,
                        "b": 1.2
                    },
                    "source": "params.a / Math.pow(params.b, doc['likes'].value)"
                }
            }
        }
    }
}
```

注意，与自定义评分查询不同，查询的评分与脚本评分的结果相乘。如果要禁止此操作，请设置“boost_mode”：“replace”