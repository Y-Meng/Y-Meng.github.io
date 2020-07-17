[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/search-aggregations-metrics.html)

# 度量聚合

这个系列中的聚合是基于以某种方式从正在聚合的文档中提取的值来计算度量的。这些值通常从文档的字段中提取（使用字段数据），但也可以使用脚本生成。

数值度量聚合是一种特殊的度量聚合类型，它输出数值。一些聚合输出单个数值度量（例如avg），称为单值数值度量聚合（single-value numeric metrics aggregation），其他聚合生成多个度量（例如stats），称为多值数值度量聚合（multi-value numeric metrics aggregation）。当这些聚合充当某些bucket聚合的直接子聚合（某些bucket聚合使您能够根据每个bucket中的数字度量对返回的bucket进行排序）时，单值和多值数值度量聚合之间的区别发挥了作用。



## 1.平均值聚合（avg）

一种单值度量聚合，用于计算从聚合文档中提取的数值的平均值。这些值可以从文档中的特定数字字段中提取，也可以由提供的脚本生成。

假设数据由代表学生考试成绩（0到100分）的文件组成，我们可以通过以下方法对他们的分数进行平均：

```json
POST /exams/_search?size=0
{
    "aggs" : {
        "avg_grade" : { "avg" : { "field" : "grade" } }
    }
}
```

上面的聚合计算所有文档的平均级别。聚合类型是avg，字段设置定义要计算平均值的文档的数字字段。以上将返回以下内容：

```json
{
    ...
    "aggregations": {
        "avg_grade": {
            "value": 75.0
        }
    }
}
```

聚合的名称（上面的avg_grade）也用作从返回的响应中检索聚合结果的键值。

#### 脚本

通过脚本计算平均值：

```json
POST /exams/_search?size=0
{
    "aggs" : {
        "avg_grade" : {
            "avg" : {
                "script" : {
                    "source" : "doc.grade.value"
                }
            }
        }
    }
}
```

这将把脚本参数解释为名为painless的脚本语言的内联脚本，且不使用脚本参数。要使用已存储的脚本，请使用以下语法：

```
POST /exams/_search?size=0
{
    "aggs" : {
        "avg_grade" : {
            "avg" : {
                "script" : {
                    "id": "my_script",
                    "params": {
                        "field": "grade"
                    }
                }
            }
        }
    }
}
```

#### 值脚本

结果发现，这次考试远远超出了学生的水平，需要进行分数修正。我们可以使用值脚本获得新的平均值：

```
POST /exams/_search?size=0
{
    "aggs" : {
        "avg_corrected_grade" : {
            "avg" : {
                "field" : "grade",
                "script" : {
                    "lang": "painless",
                    "source": "_value * params.correction",
                    "params" : {
                        "correction" : 1.2
                    }
                }
            }
        }
    }
}
```

#### 缺失值

missing参数定义了如何处理缺少值的文档。默认情况下，它们将被忽略，但也可以将它们视为具有值。

```json
POST /exams/_search?size=0
{
    "aggs" : {
        "grade_avg" : {
            "avg" : {
                "field" : "grade",
                "missing": 10 
            }
        }
    }
}
```

“grade”字段中没有值的文档将与值为10的文档属于同一个存储桶。

## 2.加权平均值聚合（weighted_avg）

加权平均值聚合是一种单值度量聚合，用于计算从聚合文档中提取的数值的加权平均值。这些值可以从文档中的特定数字字段中提取。

当计算常规平均值时，每个数据点都有一个相等的“权重”…它对最终值的贡献相等。另一方面，加权平均则对每个数据点进行不同的加权。每个数据点对最终值的贡献量从文档中提取，或由脚本提供。

权重平均值的计算公式为：

∑(value * weight) / ∑(weight)

常规平均值也可以被认为是加权平均值，其中每个值的隐式权重为1。

#### **weighted_avg** 参数

| 参数名     | 描述                             | 是否必填 | 默认值 |
| ---------- | -------------------------------- | -------- | ------ |
| value      | 配置计算值的字段或者取值脚本     | 必填     |        |
| weight     | 配置权重的字段或取值脚本         | 必填     |        |
| format     | 数值结果格式                     | 选填     |        |
| value_type | 关于纯脚本或未映射字段的值的提示 | 选填     |        |

* value参数：

| 参数名  | 描述         | 是否必填 | 默认值 |
| ------- | ------------ | -------- | ------ |
| field   | 取值字段     | 必填     |        |
| missing | 缺失值的取值 | 选填     |        |

* weight参数：

| 参数名  | 描述           | 是否必填 | 默认值 |
| ------- | -------------- | -------- | ------ |
| field   | 权重的取值字段 | 必填     |        |
| missing | 缺失值的取值   | 选填     |        |

例子

如果我们的文档中有一个“grade”字段包含0-100个数字分数，而“weight”字段包含任意数字权重，我们可以使用以下方法计算加权平均值：

```json
POST /exams/_search
{
    "size": 0,
    "aggs" : {
        "weighted_grade": {
            "weighted_avg": {
                "value": {
                    "field": "grade"
                },
                "weight": {
                    "field": "weight"
                }
            }
        }
    }
}
```

结果形式如下：

```json
{
    ...
    "aggregations": {
        "weighted_grade": {
            "value": 70.0
        }
    }
}
```

虽然每个字段允许多个值，但只允许一个权重。如果聚合遇到具有多个权重的文档（例如，权重字段是多值字段），它将引发异常。如果出现这种情况，则需要为权重字段指定一个脚本，并使用该脚本将多个值合并为一个要使用的值。

此单一权重将独立应用于从值字段提取的每个值。

此例显示如何使用单个权重平均具有多个值的单个文档：

```json
POST /exams/_doc?refresh
{
    "grade": [1, 2, 3],
    "weight": 2
}

POST /exams/_search
{
    "size": 0,
    "aggs" : {
        "weighted_grade": {
            "weighted_avg": {
                "value": {
                    "field": "grade"
                },
                "weight": {
                    "field": "weight"
                }
            }
        }
    }
}
```

三个值（1、2、3）将作为独立值包括在内，权重均为2。结果是聚合返回2.0，这与手工计算时的期望值相匹配：（（1\*2）+（2\*2）+（3\*2））/（2+2+2）==2 

#### 脚本

值和权重都可以从脚本而不是字段派生。作为一个简单的例子，下面将使用脚本在文档中的等级和权重中添加一个:

```json
POST /exams/_search
{
    "size": 0,
    "aggs" : {
        "weighted_grade": {
            "weighted_avg": {
                "value": {
                    "script": "doc.grade.value + 1"
                },
                "weight": {
                    "script": "doc.weight.value + 1"
                }
            }
        }
    }
}
```



#### 缺失值

missing参数定义了如何处理缺少值的文档。值和权重的默认行为不同：

* 如果缺少值字段，则忽略该文档，并将聚合移到下一个文档。
* 如果缺少权重字段，则假定其权重为1（与正常平均值类似）。

这两个默认值都可以用缺少的参数重写：

```json
POST /exams/_search
{
    "size": 0,
    "aggs" : {
        "weighted_grade": {
            "weighted_avg": {
                "value": {
                    "field": "grade",
                    "missing": 2
                },
                "weight": {
                    "field": "weight",
                    "missing": 3
                }
            }
        }
    }
}
```

## 3.最大值聚合（max）

单一值度量聚合，跟踪并返回从聚合文档中提取的数值中的最大值。这些值可以从文档中的特定数字字段中提取，也可以由提供的脚本生成。

> 最小和最大聚合对数据的双精度表示进行操作。因此，在绝对值大于2^53的long上运行时，结果可能是近似的。

例子：计算所有商品的最高价格

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "max_price" : { "max" : { "field" : "price" } }
    }
}
```

结果：

```json
{
    ...
    "aggregations": {
        "max_price": {
            "value": 200.0
        }
    }
}
```

可以看到，聚合的名称（上面的max_price）也是从返回的响应中检索聚合结果的键。

#### 脚本

max aggregation还可以计算脚本的最大值。以下示例计算最高价格：

```json
POST /sales/_search
{
    "aggs" : {
        "max_price" : {
            "max" : {
                "script" : {
                    "source" : "doc.price.value"
                }
            }
        }
    }
}
```

这将使用painless脚本语言，并且没有脚本参数。要使用存储的脚本，请使用以下语法：

```json
POST /sales/_search
{
    "aggs" : {
        "max_price" : {
            "max" : {
                "script" : {
                    "id": "my_script",
                    "params": {
                        "field": "price"
                    }
                }
            }
        }
    }
}
```

#### 值脚本

假设我们索引中文档的价格是以美元为单位的，但是我们想计算以欧元为单位的最大值（为了这个例子，假设转换率是1.2）。我们可以使用值脚本将转换率应用于聚合前的每个值：

```json
POST /sales/_search
{
    "aggs" : {
        "max_price_in_euros" : {
            "max" : {
                "field" : "price",
                "script" : {
                    "source" : "_value * params.conversion_rate",
                    "params" : {
                        "conversion_rate" : 1.2
                    }
                }
            }
        }
    }
}
```

#### 缺失值

缺少的参数定义了如何处理缺少值的文档。默认情况下，它们将被忽略，但也可以将它们视为具有值。

```json
POST /sales/_search
{
    "aggs" : {
        "grade_max" : {
            "max" : {
                "field" : "grade",
                "missing": 10 
            }
        }
    }
}
```

“grade”字段中没有值的文档将与值为10的文档属于同一个存储桶。

## 4.最小值聚合（min）

单一值度量聚合，跟踪并返回从聚合文档中提取的数值中的最小值。这些值可以从文档中的特定数字字段中提取，也可以由提供的脚本生成。

例子

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "min_price" : { "min" : { "field" : "price" } }
    }
}
```

结果

```json
{
    ...

    "aggregations": {
        "min_price": {
            "value": 10.0
        }
    }
}
```

#### 脚本

最小聚合还可以计算脚本的最小值。以下示例计算最低价格：

```json
POST /sales/_search
{
    "aggs" : {
        "min_price" : {
            "min" : {
                "script" : {
                    "source" : "doc.price.value"
                }
            }
        }
    }
}
```

这将使用painless脚本语言，并且没有脚本参数。要使用存储的脚本，请使用以下语法：

```json
POST /sales/_search
{
    "aggs" : {
        "min_price" : {
            "min" : {
                "script" : {
                    "id": "my_script",
                    "params": {
                        "field": "price"
                    }
                }
            }
        }
    }
}
```

#### 值脚本

假设我们索引中文档的价格是以美元为单位的，但是我们想计算以欧元为单位的最小值（为了这个例子，假设转换率是1.2）。我们可以使用值脚本将转换率应用于聚合前的每个值：

```json
POST /sales/_search
{
    "aggs" : {
        "min_price_in_euros" : {
            "min" : {
                "field" : "price",
                "script" : {
                    "source" : "_value * params.conversion_rate",
                    "params" : {
                        "conversion_rate" : 1.2
                    }
                }
            }
        }
    }
}
```

#### 缺失值

缺少的参数定义了如何处理缺少值的文档。默认情况下，它们将被忽略，但也可以将它们视为具有值。

```json
POST /sales/_search
{
    "aggs" : {
        "grade_max" : {
            "min" : {
                "field" : "grade",
                "missing": 10 
            }
        }
    }
}
```

“grade”字段中没有值的文档将与值为10的文档属于同一个存储桶。



## 5.求和聚合（sum）

一个单值度量聚合，它汇总从聚合文档中提取的数值。这些值可以从文档中的特定数字字段中提取，也可以由提供的脚本生成。

假设数据由代表销售记录的文档组成，我们可以将所有帽子的销售价格加起来：

```json
POST /sales/_search?size=0
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "match" : { "type" : "hat" }
            }
        }
    },
    "aggs" : {
        "hat_prices" : { "sum" : { "field" : "price" } }
    }
}
```

结果

```json
{
    ...
    "aggregations": {
        "hat_prices": {
           "value": 450.0
        }
    }
}
```

聚合的名称（上面的hat_prices）也是从返回的响应中检索聚合结果的键。

#### 脚本

我们可以通过脚本获取销售价格：

```json
POST /sales/_search?size=0
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "match" : { "type" : "hat" }
            }
        }
    },
    "aggs" : {
        "hat_prices" : {
            "sum" : {
                "script" : {
                   "source": "doc.price.value"
                }
            }
        }
    }
}
```

这将使用painless脚本语言，并且没有脚本参数。要使用存储的脚本，请使用以下语法：

```json
POST /sales/_search?size=0
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "match" : { "type" : "hat" }
            }
        }
    },
    "aggs" : {
        "hat_prices" : {
            "sum" : {
                "script" : {
                    "id": "my_script",
                    "params" : {
                        "field" : "price"
                    }
                }
            }
        }
    }
}
```

#### 值脚本

也可以使用_value从脚本访问字段值。例如，这将对所有帽子的价格平方求和：

```json
POST /sales/_search?size=0
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "match" : { "type" : "hat" }
            }
        }
    },
    "aggs" : {
        "square_hats" : {
            "sum" : {
                "field" : "price",
                "script" : {
                    "source": "_value * _value"
                }
            }
        }
    }
}
```

#### 缺失值

missing参数定义了如何处理缺少值的文档。默认情况下，将忽略缺少该值的文档，但也可以将其视为具有值。例如，这将所有没有价格的帽子销售视为100。

```json
POST /sales/_search?size=0
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "match" : { "type" : "hat" }
            }
        }
    },
    "aggs" : {
        "hat_prices" : {
            "sum" : {
                "field" : "price",
                "missing": 100 
            }
        }
    }
}
```

## 6.计数聚合（value_count）

一个单一值度量聚合，它计算从聚合文档中提取的值的数量。这些值可以从文档中的特定字段提取，也可以由提供的脚本生成。通常，此聚合器将与其他单值聚合一起使用。例如，当计算平均值时，一个人可能对计算平均值所依据的值的数量感兴趣。

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "types_count" : { "value_count" : { "field" : "type" } }
    }
}
```

结果

```json
{
    ...
    "aggregations": {
        "types_count": {
            "value": 7
        }
    }
}
```

#### 脚本

统计脚本生成的值：

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "type_count" : {
            "value_count" : {
                "script" : {
                    "source" : "doc['type'].value"
                }
            }
        }
    }
}
```

这将把脚本参数解释为使用painless脚本语言的内联脚本，而不使用脚本参数。要使用存储的脚本，请使用以下语法：

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "types_count" : {
            "value_count" : {
                "script" : {
                    "id": "my_script",
                    "params" : {
                        "field" : "type"
                    }
                }
            }
        }
    }
}
```

## 7.统计聚合（stats）

一种多值度量聚合，用于计算从聚合文档中提取的数值的统计信息。这些值可以从文档中的特定数字字段中提取，也可以由提供的脚本生成。

返回的结果包含统计值：`min`, `max`, `sum`, `count` 和`avg`.

假设数据由代表学生考试成绩（0到100分）的文件组成：

```json
POST /exams/_search?size=0
{
    "aggs" : {
        "grades_stats" : { "stats" : { "field" : "grade" } }
    }
}
```

以上汇总计算所有文档的成绩统计信息。聚合类型是stats，字段设置定义要计算统计信息的文档的数值字段。以上将返回以下内容：

```json
{
    ...

    "aggregations": {
        "grades_stats": {
            "count": 2,
            "min": 50.0,
            "max": 100.0,
            "avg": 75.0,
            "sum": 150.0
        }
    }
}
```

聚合的名称（上面的grades_stats）也是从返回的响应中检索聚合结果的键。

#### 脚本

基于脚本计算成绩统计：

```json
POST /exams/_search?size=0
{
    "aggs" : {
        "grades_stats" : {
            "stats" : {
                "script" : {
                    "id": "my_script",
                    "params" : {
                        "field" : "grade"
                    }
                }
            }
        }
    }
}
```

#### 值脚本

结果发现，这次考试远远超出了学生的水平，需要进行成绩修正。我们可以使用值脚本获取新的统计信息：

```json
POST /exams/_search?size=0
{
    "aggs" : {
        "grades_stats" : {
            "stats" : {
                "field" : "grade",
                "script" : {
                    "lang": "painless",
                    "source": "_value * params.correction",
                    "params" : {
                        "correction" : 1.2
                    }
                }
            }
        }
    }
}
```

#### 缺失值

缺少的参数定义了如何处理缺少值的文档。默认情况下，它们将被忽略，但也可以将它们视为具有值。

```json
POST /exams/_search?size=0
{
    "aggs" : {
        "grades_stats" : {
            "stats" : {
                "field" : "grade",
                "missing": 0 
            }
        }
    }
}
```



## 8.扩展统计聚合（extended-stats）

一种多值（multi-value）度量聚合，用于计算从聚合文档中提取字段的数值的统计信息。这些值可以从文档中的特定数字字段中提取，也可以由提供的脚本生成。

扩展的统计数据（extended_stats）聚合是统计数据（stats）聚合的扩展版本，其中添加了额外的度量，如平方和（sum_of_squares）、方差（variance）、标准差（std_deviation）和标准差界限（std_deviation_bounds）。

假设数据由代表学生考试成绩（0到100分）的文档组成。

```json
GET /exams/_search
{
    "size": 0,
    "aggs" : {
        "grades_stats" : { "extended_stats" : { "field" : "grade" } }
    }
}
```

以上汇总计算所有文档的分数统计信息。聚合类型是extended_stats，字段设置定义要计算统计信息的文档的数值字段。以上将返回以下内容：

```json
{
    ...

    "aggregations": {
        "grades_stats": {
           "count": 2,
           "min": 50.0,
           "max": 100.0,
           "avg": 75.0,
           "sum": 150.0,
           "sum_of_squares": 12500.0,
           "variance": 625.0,
           "std_deviation": 25.0,
           "std_deviation_bounds": {
            "upper": 125.0,
            "lower": 25.0
           }
        }
    }
}
```

聚合的名称（上面的grades_stats）也是从返回的响应中检索聚合结果的键。

#### 标准差界限（std_deviation_bounds）

默认情况下，edtended_stats度量将返回一个名为std_deviation_bounds的对象，该对象提供与平均值的正负两个标准偏差的间隔。这是可视化数据变化的有用方法。如果需要不同的边界，例如三个标准差，可以在请求中设置sigma：

```json
GET /exams/_search
{
    "size": 0,
    "aggs" : {
        "grades_stats" : {
            "extended_stats" : {
                "field" : "grade",
                "sigma" : 3 
            }
        }
    }
}
```

> sigma控制应显示的与平均值的标准差的数量

sigma可以是任何非负的双精度数，这意味着您可以请求非整数值，例如1.5。值0是有效的，但只返回上下限的平均值。

**标准偏差和界限要求正态性**

默认情况下会显示标准偏差及其界限，但它们并不总是适用于所有数据集。数据必须是正态分布的，才能使度量有意义。标准差背后的统计数据假定数据是正态分布的，因此如果数据严重左偏或右偏，返回的值将具有误导性。

#### 脚本

基于脚本计算成绩统计：

```json
GET /exams/_search
{
    "size": 0,
    "aggs" : {
        "grades_stats" : {
            "extended_stats" : {
                "script" : {
                    "source" : "doc['grade'].value",
                    "lang" : "painless"
                 }
             }
         }
    }
}
```

这将把脚本参数解释为使用painless脚本语言的内联脚本，而不使用脚本参数。要使用存储的脚本，请使用以下语法：

```json
GET /exams/_search
{
    "size": 0,
    "aggs" : {
        "grades_stats" : {
            "extended_stats" : {
                "script" : {
                    "id": "my_script",
                    "params": {
                        "field": "grade"
                    }
                }
            }
        }
    }
}
```

#### 值脚本

结果发现，这次考试远远超出了学生的水平，需要进行分数修正。我们可以使用值脚本获取新的统计信息：

```json
GET /exams/_search
{
    "size": 0,
    "aggs" : {
        "grades_stats" : {
            "extended_stats" : {
                "field" : "grade",
                "script" : {
                    "lang" : "painless",
                    "source": "_value * params.correction",
                    "params" : {
                        "correction" : 1.2
                    }
                }
            }
        }
    }
}
```

#### 缺失值

missing参数定义了如何处理缺少值的文档。默认情况下，它们将被忽略，但也可以将它们视为具有值。

```json
GET /exams/_search
{
    "size": 0,
    "aggs" : {
        "grades_stats" : {
            "extended_stats" : {
                "field" : "grade",
                "missing": 0 
            }
        }
    }
}
```

“grade”字段中没有值的文档将与值为0的文档属于同一个存储桶。

## 9.基数聚合(cardinality)

计算不同值的近似计数的单值度量聚合。值可以从文档中的特定字段提取，也可以由脚本生成。

假设您正在统计商店销售记录索引，并希望计算与查询匹配的已销售产品种类的唯一值数量：

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "type_count" : {
            "cardinality" : {
                "field" : "type"
            }
        }
    }
}
```

返回结果：

```json
{
    ...
    "aggregations" : {
        "type_count" : {
            "value" : 3
        }
    }
}
```

此聚合还支持精度阈值（precision_threshold）选项：

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "type_count" : {
            "cardinality" : {
                "field" : "type",
                "precision_threshold": 100 
            }
        }
    }
}
```

>  precision_threshold选项允许以内存换取精度，并定义一个唯一的计数，低于该计数时，计数将接近精度。高于此值时，计数可能会变得更模糊。支持的最大值为40000，高于此数字的阈值将与40000的阈值具有相同的效果。默认值为3000。 

#### 数值是近似的

计算精确计数需要将值加载到哈希集中并返回其大小。这在处理高基数集和/或大值时无法扩展，因为所需的内存使用量和在节点之间通信每个碎片集的需要将占用集群的太多资源。

这种基数聚合基于HyperLogLog++算法，该算法基于具有一些有趣属性的值的散列进行计数：

* 可配置精度，决定了如何用内存换取精度
* 在低基数集上有很好的精度
* 固定内存使用率：无论有数百或数十亿个唯一值，内存使用率只取决于配置的精度。

对于精度阈值c，我们使用的实现需要大约c*8字节。

 下表显示了阈值前后误差的变化情况： 

![cardinality error](..\_images\cardinality_error.png)

对于所有3个阈值，计数都精确到配置的阈值。虽然不能保证，但很可能是这样。实际的准确性取决于所讨论的数据集。一般来说，大多数数据集都显示出一贯良好的准确性。还要注意，即使阈值低至100，即使在计算数百万个项目时，错误仍然非常低（如上图所示，为1-6%）。

HyperLogLog++算法依赖于hash值的高位零数，散列在数据集中的精确分布会影响基数的准确性。

#### 预计算的hash值

在具有高基数的字符串字段上，将字段值的哈希存储在索引中，然后在此字段上运行基数聚合可能更快。这可以通过从客户端提供哈希值来实现，也可以通过使用mapper-murdur3插件让Elasticsearch为您计算散列值来实现。

> 预计算hash通常只对非常大和/或高基数字段有用，因为它节省CPU和内存。但是，在数值字段上，哈希运算非常快，存储原始值所需的内存与存储哈希运算所需的内存相同或更少。对于低基数字符串字段也是如此，特别是考虑到这些字段具有优化功能，以确保每个段的每个唯一值最多计算一次哈希值。

#### 脚本

基数度量支持脚本编写，但是由于hash需要动态计算，因此性能受到明显影响。

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "type_promoted_count" : {
            "cardinality" : {
                "script": {
                    "lang": "painless",
                    "source": "doc['type'].value + ' ' + doc['promoted'].value"
                }
            }
        }
    }
}
```

这将把脚本参数解释为使用painless脚本语言的内联脚本，而不使用脚本参数。要使用存储的脚本，请使用以下语法：

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "type_promoted_count" : {
            "cardinality" : {
                "script" : {
                    "id": "my_script",
                    "params": {
                        "type_field": "type",
                        "promoted_field": "promoted"
                    }
                }
            }
        }
    }
}
```

#### 缺失值

missing参数定义了如何处理缺少值的文档。默认情况下，它们将被忽略，但也可以将它们视为具有值。

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "tag_cardinality" : {
            "cardinality" : {
                "field" : "tag",
                "missing": "N/A" 
            }
        }
    }
}
```



## 10.百分比聚合（percentiles）

一种多值度量聚合，计算从聚合文档中提取的数值上的一个或多个百分比。这些值可以从文档中的特定数字字段中提取，也可以由提供的脚本生成。

百分位数表示观察值的某个百分比出现的点。例如，95%是大于观察值95%的值。

百分位数通常用于查找异常值。在正态分布中，0.13%和99.87%代表三个与平均值的标准差。任何超出三个标准差的数据通常被视为异常。

当一系列的百分位数被检索到时，它们可以用来估计数据分布，并确定数据是否是倾斜的、双峰的等。

假设您的数据由网站加载时间组成。对于管理员来说，平均加载时间和中间加载时间不是很有用。最大值可能很有趣，但它很容易被一个缓慢的响应所影响。

让我们看看代表加载时间的百分比范围：

```json
GET latency/_search
{
    "size": 0,
    "aggs" : {
        "load_time_outlier" : {
            "percentiles" : {
                "field" : "load_time" // 字段必须是数值类型
            }
        }
    }
}
```

默认情况下，百分比度量将生成百分比范围：[1、5、25、50、75、95、99]。回答如下：

```json
{
    ...

   "aggregations": {
      "load_time_outlier": {
         "values" : {
            "1.0": 5.0,
            "5.0": 25.0,
            "25.0": 165.0,
            "50.0": 445.0,
            "75.0": 725.0,
            "95.0": 945.0,
            "99.0": 985.0
         }
      }
   }
}
```

如您所见，聚合将返回默认范围内每个百分比的计算值。如果我们假设响应时间以毫秒为单位，那么很明显，网页通常加载时间为10-725ms，但偶尔会飙升到945-985ms。

通常，管理员只对异常值（极端百分位数）感兴趣。我们可以只指定感兴趣的百分比（请求的百分比必须是介于0到100之间的值，包括0到100）：

```json
GET latency/_search
{
    "size": 0,
    "aggs" : {
        "load_time_outlier" : {
            "percentiles" : {
                "field" : "load_time",
                "percents" : [95, 99, 99.9] 
            }
        }
    }
}
```

> 使用percents参数指定要计算的特定百分比

#### 响应key值

默认情况下，keyed标志设置为true，它将一个唯一的字符串键与每个bucket相关联，并将范围作为散列而不是数组返回。将keyed标志设置为false将禁用此行为：

```json
GET latency/_search
{
    "size": 0,
    "aggs": {
        "load_time_outlier": {
            "percentiles": {
                "field": "load_time",
                "keyed": false
            }
        }
    }
}
```

```json
{
    ...

    "aggregations": {
        "load_time_outlier": {
            "values": [
                {
                    "key": 1.0,
                    "value": 5.0
                },
                {
                    "key": 5.0,
                    "value": 25.0
                },
                {
                    "key": 25.0,
                    "value": 165.0
                },
                {
                    "key": 50.0,
                    "value": 445.0
                },
                {
                    "key": 75.0,
                    "value": 725.0
                },
                {
                    "key": 95.0,
                    "value": 945.0
                },
                {
                    "key": 99.0,
                    "value": 985.0
                }
            ]
        }
    }
}
```

#### 脚本

百分位数度量支持脚本。例如，如果加载时间以毫秒为单位，但我们希望以秒为单位计算百分位数，则可以使用脚本动态转换它们：

```json
GET latency/_search
{
    "size": 0,
    "aggs" : {
        "load_time_outlier" : {
            "percentiles" : {
                "script" : {
                    "lang": "painless",
                    "source": "doc['load_time'].value / params.timeUnit", 
                    "params" : {
                        "timeUnit" : 1000   
                    }
                }
            }
        }
    }
}
```

这将把脚本参数解释为使用painless脚本语言的内联脚本，而不使用脚本参数。要使用存储的脚本，请使用以下语法：

```json
GET latency/_search
{
    "size": 0,
    "aggs" : {
        "load_time_outlier" : {
            "percentiles" : {
                "script" : {
                    "id": "my_script",
                    "params": {
                        "field": "load_time"
                    }
                }
            }
        }
    }
}
```

#### 百分比通常是近似的

有许多不同的算法来计算百分位数。naive实现只是将所有值存储在排序数组中。

为了找到位于50%的值，可以简单的计算my_array[count(my_array) * 0.5]

显然，这种简单的实现不会缩放排序数组，它会随着数据集中值的数量线性增长。要计算Elasticsearch集群中可能有数十亿个值的百分位数，需要计算近似百分位数。

百分位数度量使用的算法称为TDigest（由Ted Dunning在使用T-Digest计算精确分位数时引入）。

使用此指标时，需要记住以下几条准则：

* 精度与q（1-q）成正比。这意味着极端百分位数（如99%）比不太极端的百分位数（如中值）更准确
* 对于较小的值集，百分位数是非常精确的（如果数据足够小，则可能100%精确）。
* 随着bucket中的值数量的增加，算法开始接近百分位数。它有效地用精确性换取内存节省。不准确的确切程度很难概括，因为它取决于数据分布和被聚合的数据量
* 下表显示了均匀分布上的相对误差，具体取决于收集值的数量和请求的百分比：

![](C:\Z_WORK\develop\1_GitHub\Y-Meng.github.io\_images\percentiles_error.png)

它显示了极端百分位数的精度如何更好。大量值误差减小的原因是大数定律使得值的分布越来越均匀，TDigest树能够更好地进行总结。在更偏斜的分布上就不是这样了。

#### 压缩

近似算法必须平衡内存利用率和估计精度。可以使用压缩参数控制此平衡：

```json
GET latency/_search
{
    "size": 0,
    "aggs" : {
        "load_time_outlier" : {
            "percentiles" : {
                "field" : "load_time",
                "tdigest": {
                  "compression" : 200 //压缩控制内存使用和近似错误
                }
            }
        }
    }
}
```

TDigest算法使用许多“节点”来近似百分位数 - 可用的节点越多，与数据量成比例的精度（和较大的内存占用）就越高。compression参数限制节点的最大数量为 20 * compression。

因此，通过增加压缩值，您可以以消耗更多内存为代价提高百分位数的精度。较大的压缩值也会使算法变慢，因为底层树数据结构的大小会增加，从而导致更昂贵的操作。默认压缩值为100。

一个“node”使用大约32字节的内存，所以在最坏的情况下（大量数据按顺序到达）默认设置将产生大约64KB大小的TDigest。实际上，数据往往更随机，而TDigest将使用更少的内存。



#### HDR直方图

> 此设置解释HDR直方图的内部实现，并且将来语法可能会更改。

HDR直方图（High Dynamic Range Histogram）是一种替代实现，在计算延迟测量的百分位数时非常有用，因为它可以比TDigest实现更快，同时兼顾更大的内存占用。

此实现维护一个固定的更坏情况百分比错误（指定为有效数字的数目）。这意味着，如果记录的数据在直方图中的值从1微秒到1小时（3600000000微秒）设置为3位有效数字，则对于最大跟踪值（1小时），它将保持1微秒的值分辨率和3.6秒（或更好）的值分辨率。

HDR直方图可以通过在请求中指定方法参数来使用：

```json
GET latency/_search
{
    "size": 0,
    "aggs" : {
        "load_time_outlier" : {
            "percentiles" : {
                "field" : "load_time",
                "percents" : [95, 99, 99.9],
                "hdr": { // 1
                  "number_of_significant_value_digits" : 3 // 2
                }
            }
        }
    }
}
```

> 1.hdr对象表示应该使用hdr直方图来计算百分位数，并且可以在对象内部指定此算法的特定设置
>
> 2.number_of_significant_value_digits有效值位数指定直方图值的分辨率（有效位数）

HDRHistogram只支持正值，如果传递负值，它将出错。如果值的范围未知，那么使用HDRHistogram也不是一个好主意，因为这可能会导致高内存使用率。

#### 缺失值

缺少的参数定义了如何处理缺少值的文档。默认情况下，它们将被忽略，但也可以将它们视为具有值。

```json
GET latency/_search
{
    "size": 0,
    "aggs" : {
        "grade_percentiles" : {
            "percentiles" : {
                "field" : "grade",
                "missing": 10 
            }
        }
    }
}
```



## 11.百分比秩聚合（percentile_ranks）

一种多值度量聚合，它计算一个或多个百分位数，高于从聚合文档中提取的数值。这些值可以从文档中的特定数字字段中提取，也可以由提供的脚本生成。

百分位秩表示观察值低于某个值的百分比。例如，如果某个值大于或等于观察值的95%，则称其为第95个百分位。

假设您的数据由网站加载时间组成。您可能有一个服务协议，即95%的页面加载要在500毫秒内完成，99%的页面加载要在600毫秒内完成。

让我们看看代表加载时间的百分比范围：

```json
GET latency/_search
{
    "size": 0,
    "aggs" : {
        "load_time_ranks" : {
            "percentile_ranks" : {
                "field" : "load_time", // 必须为数值类型
                "values" : [500, 600]
            }
        }
    }
}
```

结果

```json
{
    ...

   "aggregations": {
      "load_time_ranks": {
         "values" : {
            "500.0": 55.00000000000001,
            "600.0": 64.0
         }
      }
   }
}
```

根据这些信息，您可以确定您达到了99%的加载时间目标，但没有完全达到95%的加载时间目标。

#### 响应key值

默认情况下，keyed标志设置为true，将唯一的字符串键与每个bucket相关联，并将范围作为散列而不是数组返回。将keyed标志设置为false将禁用此行为：

```json
GET latency/_search
{
    "size": 0,
    "aggs": {
        "load_time_ranks": {
            "percentile_ranks": {
                "field": "load_time",
                "values": [500, 600],
                "keyed": false
            }
        }
    }
}
```

响应

```json
{
    ...

    "aggregations": {
        "load_time_ranks": {
            "values": [
                {
                    "key": 500.0,
                    "value": 55.00000000000001
                },
                {
                    "key": 600.0,
                    "value": 64.0
                }
            ]
        }
    }
}
```

#### 脚本

百分位秩度量支持脚本。例如，如果加载时间以毫秒为单位，但我们希望以秒为单位指定值，则可以使用脚本动态转换它们：

```json
GET latency/_search
{
    "size": 0,
    "aggs" : {
        "load_time_ranks" : {
            "percentile_ranks" : {
                "values" : [500, 600],
                "script" : {
                    "lang": "painless",
                    "source": "doc['load_time'].value / params.timeUnit", 
                    "params" : {
                        "timeUnit" : 1000   
                    }
                }
            }
        }
    }
}
```

> 1.field参数替换为script参数，该参数使用脚本生成值，计算百分比等级
>
> 2.脚本支持参数化输入，就像其他脚本一样

这将把脚本参数解释为使用painless脚本语言的内联脚本，而不使用脚本参数。要使用存储的脚本，请使用以下语法：

```json
GET latency/_search
{
    "size": 0,
    "aggs" : {
        "load_time_ranks" : {
            "percentile_ranks" : {
                "values" : [500, 600],
                "script" : {
                    "id": "my_script",
                    "params": {
                        "field": "load_time"
                    }
                }
            }
        }
    }
}
```

#### HDR直方图

> 此设置解释HDR直方图的内部实现，并且将来语法可能会更改。

HDR直方图（High Dynamic Range Histogram）是一种替代实现，在计算延迟测量的百分位数时非常有用，因为它可以比TDigest实现更快，同时兼顾更大的内存占用。

此实现维护一个固定的更坏情况百分比错误（指定为有效数字的数目）。这意味着，如果记录的数据在直方图中的值从1微秒到1小时（3600000000微秒）设置为3位有效数字，则对于最大跟踪值（1小时），它将保持1微秒的值分辨率和3.6秒（或更好）的值分辨率。

HDR直方图可以通过在请求中指定方法参数来使用：

```json
GET latency/_search
{
    "size": 0,
    "aggs" : {
        "load_time_outlier" : {
            "percentile_ranks" : {
                "field" : "load_time",
                "values" : [500,600],
                "hdr": { // 1
                  "number_of_significant_value_digits" : 3 // 2
                }
            }
        }
    }
}
```

> 1.hdr对象表示应该使用hdr直方图来计算百分位数，并且可以在对象内部指定此算法的特定设置
>
> 2.number_of_significant_value_digits有效值位数指定直方图值的分辨率（有效位数）

HDRHistogram只支持正值，如果传递负值，它将出错。如果值的范围未知，那么使用HDRHistogram也不是一个好主意，因为这可能会导致高内存使用率。

#### 缺失值

缺少的参数定义了如何处理缺少值的文档。默认情况下，它们将被忽略，但也可以将它们视为具有值

```
GET latency/_search
{
    "size": 0,
    "aggs" : {
        "load_time_ranks" : {
            "percentile_ranks" : {
                "field" : "load_time",
                "values" : [500, 600],
                "missing": 10 
            }
        }
    }
}
```



## 12.排行聚合（top_hits）

top_hits度量聚合器跟踪被聚合的最相关文档。此聚合器将用作子聚合器，以便每个bucket可以聚合顶部匹配的文档。

top_hits聚合器可以有效地用于通过bucket聚合器按特定字段对结果集进行分组。一个或多个bucket聚合器确定结果集被切片到的属性。

#### 选项

* **from** - 要获取的第一个结果的偏移量。
* **size** - 每个存储桶要返回的最大匹配命中数。默认情况下，会返回前三个匹配的点击。
* **sort** - 如何对命中文档进行排序。默认情况下，命中率按主查询的分数排序。

#### 每个命中功能

top_hits聚合返回常规搜索结果，因为可以支持以下每次命中的功能：

* 高亮（Highlighting）
* 解释（Explain）
* 命名过滤或查询（Named filters and queries）
* 源过滤（source filtering）
* 排序字段（sorted fields）
* 脚本字段（script fields）
* 文档值字段（doc value fields）
* 包含版本（include version）
* 包含序列号和基本术语（include sequence numbers and primary terms）

例子

在下面的示例中，我们按类型对销售进行分组，并按类型显示上次销售。对于每次销售，只有日期和价格字段包含在源中。

```json
POST /sales/_search?size=0
{
    "aggs": {
        "top_tags": {
            "terms": {
                "field": "type",
                "size": 3
            },
            "aggs": {
                "top_sales_hits": {
                    "top_hits": {
                        "sort": [
                            {
                                "date": {
                                    "order": "desc"
                                }
                            }
                        ],
                        "_source": {
                            "includes": [ "date", "price" ]
                        },
                        "size" : 1
                    }
                }
            }
        }
    }
}
```

结果：

```json
{
  ...
  "aggregations": {
    "top_tags": {
       "doc_count_error_upper_bound": 0,
       "sum_other_doc_count": 0,
       "buckets": [
          {
             "key": "hat",
             "doc_count": 3,
             "top_sales_hits": {
                "hits": {
                   "total" : {
                       "value": 3,
                       "relation": "eq"
                   },
                   "max_score": null,
                   "hits": [
                      {
                         "_index": "sales",
                         "_type": "_doc",
                         "_id": "AVnNBmauCQpcRyxw6ChK",
                         "_source": {
                            "date": "2015/03/01 00:00:00",
                            "price": 200
                         },
                         "sort": [
                            1425168000000
                         ],
                         "_score": null
                      }
                   ]
                }
             }
          },
          {
             "key": "t-shirt",
             "doc_count": 3,
             "top_sales_hits": {
                "hits": {
                   "total" : {
                       "value": 3,
                       "relation": "eq"
                   },
                   "max_score": null,
                   "hits": [
                      {
                         "_index": "sales",
                         "_type": "_doc",
                         "_id": "AVnNBmauCQpcRyxw6ChL",
                         "_source": {
                            "date": "2015/03/01 00:00:00",
                            "price": 175
                         },
                         "sort": [
                            1425168000000
                         ],
                         "_score": null
                      }
                   ]
                }
             }
          },
          {
             "key": "bag",
             "doc_count": 1,
             "top_sales_hits": {
                "hits": {
                   "total" : {
                       "value": 1,
                       "relation": "eq"
                   },
                   "max_score": null,
                   "hits": [
                      {
                         "_index": "sales",
                         "_type": "_doc",
                         "_id": "AVnNBmatCQpcRyxw6ChH",
                         "_source": {
                            "date": "2015/01/01 00:00:00",
                            "price": 150
                         },
                         "sort": [
                            1420070400000
                         ],
                         "_score": null
                      }
                   ]
                }
             }
          }
       ]
    }
  }
}
```

#### 字段折叠示例

字段折叠或结果分组是一种功能，它可以将结果集逻辑分组，并按组返回顶级文档。组的顺序由组中第一个文档的相关性决定。在Elasticsearch中，这可以通过bucket聚合器实现，bucket聚合器将top_hits聚合器包装为子聚合器。

在下面的例子中，我们搜索爬网网页。对于每个网页，我们都存储了网页所属的正文和域名。通过在域名字段上定义term聚合器，我们可以按域名对网页的结果集进行分组。然后将top_hits聚合器定义为子聚合器，以便每个bucket收集最匹配的hits。

还定义了一个max aggregator，terms aggregator的order特性使用它按bucket中最相关文档的关联顺序返回bucket。

```json
POST /sales/_search
{
  "query": {
    "match": {
      "body": "elections"
    }
  },
  "aggs": {
    "top_sites": {
      "terms": {
        "field": "domain",
        "order": {
          "top_hit": "desc"
        }
      },
      "aggs": {
        "top_tags_hits": {
          "top_hits": {}
        },
        "top_hit" : {
          "max": {
            "script": {
              "source": "_score"
            }
          }
        }
      }
    }
  }
}
```

目前，需要使用max（或min）聚合器来确保根据每个域中最相关网页的得分对术语聚合器中的bucket进行排序。不幸的是，top_hits aggregator还不能用于terms aggregator的order选项。

#### top_hits支持嵌套或反向嵌套聚合器

如果top_hits聚合器包装在嵌套或反向嵌套聚合器中，则返回嵌套的hits。嵌套命中在某种意义上是隐藏的迷你文档，这些文档是常规文档的一部分，在映射中已配置了嵌套字段类型。

如果顶级hits聚合器包装在嵌套或反向嵌套聚合器中，则它可以取消隐藏这些文档。阅读有关嵌套[类型映射中嵌套](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/nested.html)的详细信息。

如果已配置嵌套类型，则单个文档实际上会被索引为多个Lucene文档，并且它们共享同一id。为了确定嵌套命中的标识，需要的不仅仅是id，因此嵌套命中也包括其嵌套标识。嵌套标识保留在搜索命中中的嵌套字段下，并包括嵌套命中所属的数组字段中的数组字段和偏移量。偏移量是基于零的。

我们通过一个示例来看它是如何工作的：

```json
PUT /sales
{
    "mappings": {
        "properties" : {
            "tags" : { "type" : "keyword" },
            "comments" : { // 1
                "type" : "nested",
                "properties" : {
                    "username" : { "type" : "keyword" },
                    "comment" : { "type" : "text" }
                }
            }
        }
    }
}
```

> 1. comments是一个数组，用于保存产品对象下的嵌套文档。

现在可以执行以下top_hits聚合（包装在嵌套聚合中）：

```json
POST /sales/_search
{
    "query": {
        "term": { "tags": "car" }
    },
    "aggs": {
        "by_sale": {
            "nested" : {
                "path" : "comments"
            },
            "aggs": {
                "by_user": {
                    "terms": {
                        "field": "comments.username",
                        "size": 1
                    },
                    "aggs": {
                        "by_nested": {
                            "top_hits":{}
                        }
                    }
                }
            }
        }
    }
}
```

使用嵌套命中的响应片段，位于数组字段注释的第一个槽中：

```json
{
  ...
  "aggregations": {
    "by_sale": {
      "by_user": {
        "buckets": [
          {
            "key": "baddriver007",
            "doc_count": 1,
            "by_nested": {
              "hits": {
                "total" : {
                   "value": 1,
                   "relation": "eq"
                },
                "max_score": 0.3616575,
                "hits": [
                  {
                    "_index": "sales",
                    "_type" : "_doc",
                    "_id": "1",
                    "_nested": {
                      "field": "comments",  // 1
                      "offset": 0 // 2
                    },
                    "_score": 0.3616575,
                    "_source": {
                      "comment": "This car could have better brakes", // 3
                      "username": "baddriver007"
                    }
                  }
                ]
              }
            }
          }
          ...
        ]
      }
    }
  }
}
```

> 1.包含嵌套命中的数组字段名称
>
> 2.嵌套命中在数组中的位置
>
> 3.嵌套命中的数据源

如果请求了_source，则只返回嵌套对象源的一部分，而不是文档的整个源。嵌套内部对象级别上的存储字段也可以通过位于嵌套或反向嵌套聚合器中的top_hits聚合器访问。

只有嵌套命中在命中中有一个嵌套（nested）字段，非嵌套（常规）命中没有一个嵌套字段。

如果未启用_source，嵌套中的信息也可以用于解析其他地方的原始源。

如果在映射中定义了多层嵌套对象类型，则嵌套信息也可以是分层的，以便表示两层或两层以上的嵌套命中的标识。

在下面的示例中，嵌套的命中位于字段嵌套的子字段的第一个槽中，该槽位于嵌套的子字段的第二个槽中：

```json
...
"hits": {
 "total" : {
     "value": 2565,
     "relation": "eq"
 },
 "max_score": 1,
 "hits": [
   {
     "_index": "a",
     "_type": "b",
     "_id": "1",
     "_score": 1,
     "_nested" : {
       "field" : "nested_child_field",
       "offset" : 1,
       "_nested" : {
         "field" : "nested_grand_child_field",
         "offset" : 0
       }
     }
     "_source": ...
   },
   ...
 ]
}
...
```



## 13.中值标准差聚合（median_absolute_deviation）

这种单值聚合近似于其搜索结果的中位数绝对值标准差。

中位数绝对偏差是可变性的度量。它是一个稳健的统计，这意味着它对于描述可能有异常值或可能不是正态分布的数据是有用的。对于这些数据，它可以比标准差更具描述性。

它被计算为每个数据点偏离整个样本中值的中值。也就是说，对于随机变量X，中值标准差是

median（|median（X）-Xi |）。

例子

假设我们的数据代表一到五星级的产品评论。这样的评论通常被总结为一种手段，这是容易理解的，但没有描述评论的可变性。估计中位数标准差可以洞察评论之间的差异。

在这个例子中，我们有一个平均评级为3星的产品。让我们看看它的评级的中位数标准差，以确定它们之间的差异有多大。

```json
GET reviews/_search
{
  "size": 0,
  "aggs": {
    "review_average": {
      "avg": {
        "field": "rating"
      }
    },
    "review_variability": {
      "median_absolute_deviation": {
        "field": "rating" // 必须为数值字段
      }
    }
  }
}
```

由此产生的中位数绝对偏差2告诉我们，在评级中有相当多的可变性。评论者对这个产品必须有不同的意见。

```json
{
  ...
  "aggregations": {
    "review_average": {
      "value": 3.0
    },
    "review_variability": {
      "value": 2.0
    }
  }
}
```

#### 近似值（approximation）

计算中值绝对偏差的原始实现将整个样本存储在内存中，因此此聚合将计算近似值。

它使用tdiest数据结构来近似样本中值和偏离样本中值的中值。有关tdigest的近似特性的更多信息，请参见百分位数（通用）近似值。

TDigest分位数近似的资源使用和精度之间的折衷，以及因此该聚合的中值绝对偏差近似的精度，由压缩参数控制。较高的压缩设置以较高的内存使用率为代价提供了更精确的近似值。有关tdiest压缩参数特性的更多信息，请参见[压缩](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/search-aggregations-metrics-percentile-aggregation.html#search-aggregations-metrics-percentile-aggregation-compression)。

```json
GET reviews/_search
{
  "size": 0,
  "aggs": {
    "review_variability": {
      "median_absolute_deviation": {
        "field": "rating",
        "compression": 100
      }
    }
  }
}
```

此聚合的默认compression值为1000。在这个压缩级别上，这种聚合通常在精确结果的5%以内，但是观察到的性能将取决于样本数据。

#### 脚本

此度量聚合支持脚本。在上面的例子中，产品评论的范围是1到5。如果我们想将它们修改为1到10的比例，我们可以使用脚本。

使用内联脚本

```json
GET reviews/_search
{
  "size": 0,
  "aggs": {
    "review_variability": {
      "median_absolute_deviation": {
        "script": {
          "lang": "painless",
          "source": "doc['rating'].value * params.scaleFactor",
          "params": {
            "scaleFactor": 2
          }
        }
      }
    }
  }
}
```

使用预定义脚本

```json
GET reviews/_search
{
  "size": 0,
  "aggs": {
    "review_variability": {
      "median_absolute_deviation": {
        "script": {
          "id": "my_script",
          "params": {
            "field": "rating"
          }
        }
      }
    }
  }
}
```

#### 缺失值

missing参数定义了如何处理缺少值的文档。默认情况下，它们将被忽略，但也可以将它们视为具有值。

让我们乐观一点，假设一些评论员非常喜欢这个产品，以至于忘记给它打分。我们给他们分配五颗星：

```json
GET reviews/_search
{
  "size": 0,
  "aggs": {
    "review_variability": {
      "median_absolute_deviation": {
        "field": "rating",
        "missing": 5
      }
    }
  }
}
```

## 14.脚本定义度量聚合（scripted_metric）

可以使用脚本执行以提供度量输出的度量聚合。

```json
POST ledger/_search?size=0
{
    "query" : {
        "match_all" : {}
    },
    "aggs": {
        "profit": {
            "scripted_metric": {
                "init_script" : "state.transactions = []", 
                "map_script" : "state.transactions.add(doc.type.value == 'sale' ? doc.amount.value : -1 * doc.amount.value)",
                "combine_script" : "double profit = 0; for (t in state.transactions) { profit += t } return profit",
                "reduce_script" : "double profit = 0; for (a in states) { profit += a } return profit"
            }
        }
    }
}
```

> init_sript是可选参数，其他参数都是必填参数

上面的聚合演示了如何使用脚本聚合计算销售和成本交易的总利润。

结果如下：

```json
{
    "took": 218,
    ...
    "aggregations": {
        "profit": {
            "value": 240.0
        }
   }
}
```

上面的示例也可以使用预定义存储的脚本指定，如下所示：

```json
POST ledger/_search?size=0
{
    "aggs": {
        "profit": {
            "scripted_metric": {
                "init_script" : {
                    "id": "my_init_script"
                },
                "map_script" : {
                    "id": "my_map_script"
                },
                "combine_script" : {
                    "id": "my_combine_script"
                },
                "params": {
                    "field": "amount" // 1
                },
                "reduce_script" : {
                    "id": "my_reduce_script"
                }
            }
        }
    }
}
```

> 1.必须在全局params对象中指定init、map和combine脚本的脚本参数，以便在脚本之间共享。

更多信息请查看[脚本文档](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/modules-scripting.html)

#### 允许的返回值

虽然任何有效的脚本对象都可以在单个脚本中使用，但脚本必须仅返回或存储以下类型：

* 基本类型
* String
* Map(key和value的类型必须是这里列出的类型)
* Array(元素的类型必须是这里列出的类型)

#### 脚本范围

脚本化度量聚合在其执行的4个阶段使用脚本：

* **init_script**

在任何文件收集之前执行。允许聚合设置任何初始状态。

在上面的示例中init_script脚本在state对象中创建一个transactions数组。

* **map_script**

每收集一份文档执行一次。这是必需的脚本。如果未指定任何组合脚本，则结果状态需要存储在state对象中。

在上面的示例中，map_script检查type字段的值。如果值是sale，那么amount字段的值将添加到transactions数组中。如果type字段的值不是sale，那么amount字段的反值将添加到事务处理中。

* **combine_script**

在文档收集完成后对每个分片执行一次。这是必需的脚本。允许聚合合并从每个碎片返回的状态。

在上面的例子中，combine_script遍历所有存储的事务，求和利润变量中的值，最后返回利润。

* **reduce_script**

在所有分片返回结果后在协调节点上执行一次。这是必需的脚本。该脚本提供对变量状态的访问，变量状态是每个碎片上的combine_script结果的数组。

在上面的例子中，reduce_script遍历每个shard返回的利润，并将这些值相加，然后返回在聚合响应中返回的最终合并利润。

#### 工作过程示例

想象一下这样一种情况：您将以下文档索引为包含两个碎片的索引：

```json
PUT /transactions/_bulk?refresh
{"index":{"_id":1}}
{"type": "sale","amount": 80}
{"index":{"_id":2}}
{"type": "cost","amount": 10}
{"index":{"_id":3}}
{"type": "cost","amount": 30}
{"index":{"_id":4}}
{"type": "sale","amount": 130}
```

假设文档1和3位于shard A，文档2和4位于shard B。下面是在上面示例的每个阶段的聚合结果明细。

1. init_script前

初始化一个新的stat对象

```json
"state" : {}
```

2. init_script后

在执行任何文档收集之前，这将在每个shard上运行一次，因此我们将在每个分片上有一个副本：

```json
// 分片A
"state" : {
    "transactions" : []
}
// 分片B
"state" : {
    "transactions" : []
}
```

3. map_script后

每个分片收集其文档，并在收集的每个文档上运行map_script：

```json
// 分片B
"state" : {
    "transactions" : [ 80, -30 ]
}
// 分片B
"state" : {
    "transactions" : [ -10, 130 ]
}
```

4. combine_script后

文档收集完成后，在每个碎片上执行combine_script，并将所有事务减少到每个分片的单个利润数字（通过对事务数组中的值求和），然后将其传递回协调节点：

```json
// Shard A
50
// Shard B
120
```

5. reduce_script后

reduce_script接收一个states数组，其中包含每个shard的合并脚本的结果：

```json
"states" : [
    50,
    120
]
```

它将shard的响应减少到最终的总利润数字（通过对值求和），并将其作为聚合的结果返回以生成响应：

```json
{
    ...
    "aggregations": {
        "profit": {
            "value": 170
        }
   }
}
```

#### 其他参数

* params

params定义一个可选参数对象，其内容将作为变量传递给init\_script、map\_script和combine_script。这对于允许用户控制聚合行为和存储脚本之间的状态非常有用。如果未指定，则默认值等于提供：

```json
"params" : {}
```

#### 空桶

如果脚本化度量聚合的父bucket不收集任何文档，则将从shard返回空聚合响应，返回空值。在这种情况下，reduce_script的states变量将包含null作为来自该shard的响应。因此reduce_script需要可以处理来自碎片的空响应。

## 15.地理边界聚合（geo_bounds）

计算包含字段所有地理点值的边界框的度量聚合。

例子i

```json
PUT /museums
{
    "mappings": {
        "properties": {
            "location": {
                "type": "geo_point"
            }
        }
    }
}

POST /museums/_bulk?refresh
{"index":{"_id":1}}
{"location": "52.374081,4.912350", "name": "NEMO Science Museum"}
{"index":{"_id":2}}
{"location": "52.369219,4.901618", "name": "Museum Het Rembrandthuis"}
{"index":{"_id":3}}
{"location": "52.371667,4.914722", "name": "Nederlands Scheepvaartmuseum"}
{"index":{"_id":4}}
{"location": "51.222900,4.405200", "name": "Letterenhuis"}
{"index":{"_id":5}}
{"location": "48.861111,2.336389", "name": "Musée du Louvre"}
{"index":{"_id":6}}
{"location": "48.860000,2.327000", "name": "Musée d'Orsay"}

POST /museums/_search?size=0
{
    "query" : {
        "match" : { "name" : "musée" }
    },
    "aggs" : {
        "viewport" : {
            "geo_bounds" : {
                "field" : "location", // 1
                "wrap_longitude" : true  // 2
            }
        }
    }
}
```

> 1.指定聚合分析字段
>
> 2.wrap_longitude是一个可选参数，指定是否允许边界框重叠国际日期行。默认值为true

上面的聚合演示了如何计算所有文档的location字段的边界框。

聚合的返回值如下：

```json
{
    ...
    "aggregations": {
        "viewport": {
            "bounds": {
                "top_left": {
                    "lat": 48.86111099738628,
                    "lon": 2.3269999679178
                },
                "bottom_right": {
                    "lat": 48.85999997612089,
                    "lon": 2.3363889567553997
                }
            }
        }
    }
}
```

## 16.地理质心聚合（geo_centroid）

从地理点字段的所有坐标值计算加权质心的度量聚合。

例子：

```json
PUT /museums
{
    "mappings": {
        "properties": {
            "location": {
                "type": "geo_point"
            }
        }
    }
}

POST /museums/_bulk?refresh
{"index":{"_id":1}}
{"location": "52.374081,4.912350", "city": "Amsterdam", "name": "NEMO Science Museum"}
{"index":{"_id":2}}
{"location": "52.369219,4.901618", "city": "Amsterdam", "name": "Museum Het Rembrandthuis"}
{"index":{"_id":3}}
{"location": "52.371667,4.914722", "city": "Amsterdam", "name": "Nederlands Scheepvaartmuseum"}
{"index":{"_id":4}}
{"location": "51.222900,4.405200", "city": "Antwerp", "name": "Letterenhuis"}
{"index":{"_id":5}}
{"location": "48.861111,2.336389", "city": "Paris", "name": "Musée du Louvre"}
{"index":{"_id":6}}
{"location": "48.860000,2.327000", "city": "Paris", "name": "Musée d'Orsay"}

POST /museums/_search?size=0
{
    "aggs" : {
        "centroid" : {
            "geo_centroid" : {
                "field" : "location" // 1
            }
        }
    }
}
```

> 1.geo_centroid聚集指定用于计算质心的字段。（注意：字段必须是地理点geo-point类型）

上面的聚合演示了如何计算所有文档的位置字段的质心。

聚合结果如下：

```json
{
    ...
    "aggregations": {
        "centroid": {
            "location": {
                "lat": 51.00982965203002,
                "lon": 3.9662131341174245
            },
            "count": 6
        }
    }
}
```

当组合为其他bucket聚合的子聚合时，geo_centroid聚合更有趣。

```json
POST /museums/_search?size=0
{
    "aggs" : {
        "cities" : {
            "terms" : { "field" : "city.keyword" },
            "aggs" : {
                "centroid" : {
                    "geo_centroid" : { "field" : "location" }
                }
            }
        }
    }
}
```

上面的示例使用geo_centroid作为bucket聚合的子聚合，用于查找每个城市中博物馆的中心位置。

结果如下：

```json
{
    ...
    "aggregations": {
        "cities": {
            "sum_other_doc_count": 0,
            "doc_count_error_upper_bound": 0,
            "buckets": [
               {
                   "key": "Amsterdam",
                   "doc_count": 3,
                   "centroid": {
                      "location": {
                         "lat": 52.371655656024814,
                         "lon": 4.909563297405839
                      },
                      "count": 3
                   }
               },
               {
                   "key": "Paris",
                   "doc_count": 2,
                   "centroid": {
                      "location": {
                         "lat": 48.86055548675358,
                         "lon": 2.3316944623366
                      },
                      "count": 2
                   }
                },
                {
                    "key": "Antwerp",
                    "doc_count": 1,
                    "centroid": {
                       "location": {
                          "lat": 51.22289997059852,
                          "lon": 4.40519998781383
                       },
                       "count": 1
                    }
                 }
            ]
        }
    }
}
```