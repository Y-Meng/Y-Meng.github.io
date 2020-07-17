# 管道聚合

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/search-aggregations-pipeline.html)



管道聚合处理从其他聚合（而不是从文档集）生成的输出，将信息添加到输出树中。有许多不同类型的管道聚合，每种聚合计算的信息与其他聚合不同，但这些类型可以分为两个系列：

* Parent：一个系列管道聚合，随其父聚合的输出一起提供，能够计算新的存储桶或新的聚合以添加到现有存储桶。
* Sibling：管道聚合，随同级聚合的输出一起提供，并且能够计算与同级聚合处于同一级别的新聚合。

管道聚合可以引用它们执行计算所需的聚合，方法是使用buckets_path参数指示指向所需度量的路径。定义这些路径的语法可以在下面的bucket路径语法部分找到。

管道聚合不能有子聚合，但取决于它可以引用bucket路径中允许链接管道聚合的另一个管道的类型。例如，可以将两个导数连在一起以计算二阶导数（即一个导数的导数）。

>  由于管道聚合仅添加到输出中，因此在链接管道聚合时，每个管道聚合的输出将包含在最终输出中。

### buckets_path语法

大多数管道聚合需要另一个聚合作为其输入。输入聚合通过bucket_path参数定义，该参数遵循特定格式：

```properties
AGG_SEPARATOR       =  '>' ;
METRIC_SEPARATOR    =  '.' ;
AGG_NAME            =  <the name of the aggregation> ;
METRIC              =  <the name of the metric (in case of multi-value metrics aggregation)> ;
PATH                =  <AGG_NAME> [ <AGG_SEPARATOR>, <AGG_NAME> ]* [ <METRIC_SEPARATOR>, <METRIC> ] ;
```

例如，路径“my_bucket>my_stats.avg”将指向“my_stats”度量中的avg值，该度量包含在“my_bucket”bucket聚合中。

路径与管道聚合的位置是相对的；它们不是绝对路径，路径不能返回聚合树。例如，此移动平均值嵌入在日期直方图中，并引用“sibling”度量“the_sum”：

```json
POST /_search
{
    "aggs": {
        "my_date_histo":{
            "date_histogram":{
                "field":"timestamp",
                "calendar_interval":"day"
            },
            "aggs":{
                "the_sum":{
                    "sum":{ "field": "lemmings" } 
                },
                "the_movavg":{
                    "moving_avg":{ "buckets_path": "the_sum" } // buckets_path通过相对路径指向the_sum
                }
            }
        }
    }
}
```

buckets_path还用于同级管道聚合，其中聚合是一系列buckets的“下一个”，而不是嵌入它们的“内部”。例如，max_bucket聚合使用bucket_路径指定嵌入在同级聚合中的度量：

```json
POST /_search
{
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                }
            }
        },
        "max_monthly_sales": {
            "max_bucket": { // max_bucket为最大桶聚合（管道聚合的一种）
                "buckets_path": "sales_per_month>sales"
            }
        }
    }
}
```

### 特殊路径

bucket路径可以使用特殊的**“\_count”**路径，而不是指向度量。这指示管道聚合使用文档计数作为其输入。例如，可以根据每个存储桶的文档计数来计算移动平均值，而不是特定的度量：

```json
POST /_search
{
    "aggs": {
        "my_date_histo": {
            "date_histogram": {
                "field":"timestamp",
                "calendar_interval":"day"
            },
            "aggs": {
                "the_movavg": {
                    "moving_avg": { "buckets_path": "_count" } 
                }
            }
        }
    }
}
```

bucket_path还可以使用**“\_bucket_count”**和多bucket聚合的路径，以使用该聚合在管道聚合中返回的bucket数，而不是度量。例如，这里可以使用**bucket_selector**筛选出不包含内部术语聚合的bucket：

```json
POST /sales/_search
{
  "size": 0,
  "aggs": {
    "histo": {
      "date_histogram": {
        "field": "date",
        "calendar_interval": "day"
      },
      "aggs": {
        "categories": {
          "terms": {
            "field": "category"
          }
        },
        "min_bucket_selector": {
          "bucket_selector": {
            "buckets_path": {
              "count": "categories._bucket_count"  // 1
            },
            "script": {
              "source": "params.count != 0"
            }
          }
        }
      }
    }
  }
}
```

> 通过使用\_bucket_count而不是metric名称，我们可以筛选出histo bucket，其中它们不包含类别聚合的bucket

### 处理聚合名称中的点

支持另一种语法来处理名称中有点的聚合或度量，例如99.9%的值。该指标可称为：

```json
"buckets_path": "my_percentile[99.9]"
```

### 处理数据中的空隙

现实世界中的数据常常是嘈杂的，有时会包含数据根本不存在的间隙。这可能有多种原因，最常见的是：

* 落入桶中的文档不包含一个需要的字段
* 存在一个或多个桶未匹配到文档
* 正在计算的度量无法生成值，这可能是因为另一个依赖bucket缺少值。某些管道聚合具有必须满足的特定要求（例如，由于没有以前的值，导数无法计算第一个值的度量，Holtwiners移动平均需要“预热”数据才能开始计算，等等）

当遇到“gappy”或丢失的数据时，Gap策略是一种通知管道聚合所需行为的机制。所有管道聚合都接受gap_policy参数。目前有两种差距政策可供选择：

* skip：此选项将丢失的数据视为bucket不存在。它将跳过bucket并使用下一个可用值继续计算。
* insert_zeros：此选项将用零（0）替换缺少的值，管道聚合计算将正常进行。

# 管道聚合类型

## 1. Parent

### 1.1. Derivative导数聚合

父管道聚合，它计算父直方图（或日期直方图）聚合中指定度量的导数。指定的度量必须是数字，并且封闭的直方图必须将“最小文档计数”设置为0（直方图聚合的默认值）。

```json
{
    "derivative": {
 	 	"buckets_path": "the_sum"
	}
}
```

参数

| 参数名      | 说明                         | 必填 | 默认值 |
| ----------- | ---------------------------- | ---- | ------ |
| bucket_path | 我们希望计算导数的桶的路径   | 是   |        |
| gap_policy  | 在数据中发现空隙时应用的策略 | 否   | skip   |
| format      | 应用于此聚合的输出值的格式   | 否   | null   |

#### 一阶导数

以下代码片段计算每月总销售额的增长率（导数）：

```json
POST /sales/_search
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                },
                "sales_deriv": {
                    "derivative": {
                        "buckets_path": "sales" // 使用sales的输出作为输入求导
                    }
                }
            }
        }
    }
}
```

结果

```json
{
   "took": 11,
   "timed_out": false,
   "_shards": [],
   "hits": []
   "aggregations": {
      "sales_per_month": {
         "buckets": [
            {
               "key_as_string": "2015/01/01 00:00:00",
               "key": 1420070400000,
               "doc_count": 3,
               "sales": {
                  "value": 550.0
               } 
            },
            {
               "key_as_string": "2015/02/01 00:00:00",
               "key": 1422748800000,
               "doc_count": 2,
               "sales": {
                  "value": 60.0
               },
               "sales_deriv": {
                  "value": -490.0 
               }
            },
            {
               "key_as_string": "2015/03/01 00:00:00",
               "key": 1425168000000,
               "doc_count": 2, 
               "sales": {
                  "value": 375.0
               },
               "sales_deriv": {
                  "value": 315.0
               }
            }
         ]
      }
   }
}
```

#### 二阶导数



二阶导数可以通过将导数管道聚合链接到另一个导数管道聚合的结果来计算，如下例所示，该示例将计算每月总销售额的一阶和二阶导数：

```json
POST /sales/_search
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                },
                "sales_deriv": {
                    "derivative": {
                        "buckets_path": "sales"
                    }
                },
                "sales_2nd_deriv": {
                    "derivative": {
                        "buckets_path": "sales_deriv" 
                    }
                }
            }
        }
    }
}

```

结果

```json
{
   "took": 50,
   "timed_out": false,
   "_shards": ...,
   "hits": ...,
   "aggregations": {
      "sales_per_month": {
         "buckets": [
            {
               "key_as_string": "2015/01/01 00:00:00",
               "key": 1420070400000,
               "doc_count": 3,
               "sales": {
                  "value": 550.0
               } 
            },
            {
               "key_as_string": "2015/02/01 00:00:00",
               "key": 1422748800000,
               "doc_count": 2,
               "sales": {
                  "value": 60.0
               },
               "sales_deriv": {
                  "value": -490.0
               } 
            },
            {
               "key_as_string": "2015/03/01 00:00:00",
               "key": 1425168000000,
               "doc_count": 2,
               "sales": {
                  "value": 375.0
               },
               "sales_deriv": {
                  "value": 315.0
               },
               "sales_2nd_deriv": {
                  "value": 805.0
               }
            }
         ]
      }
   }
}
```

#### 单位

导数聚合允许指定导数值的单位。这将在响应normalized_value中返回一个额外的字段，该字段以所需的x轴单位报告导数值。在下面的示例中，我们计算每月总销售额的导数，但要求以每天销售额为单位计算销售额的导数：

```json
POST /sales/_search
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                },
                "sales_deriv": {
                    "derivative": {
                        "buckets_path": "sales",
                        "unit": "day" // 指定单位为天
                    }
                }
            }
        }
    }
}
```

结果

```json
{
   "took": 50,
   "timed_out": false,
   "_shards": ...,
   "hits": ...,
   "aggregations": {
      "sales_per_month": {
         "buckets": [
            {
               "key_as_string": "2015/01/01 00:00:00",
               "key": 1420070400000,
               "doc_count": 3,
               "sales": {
                  "value": 550.0
               } 
            },
            {
               "key_as_string": "2015/02/01 00:00:00",
               "key": 1422748800000,
               "doc_count": 2,
               "sales": {
                  "value": 60.0
               },
               "sales_deriv": {
                  "value": -490.0, 
                  "normalized_value": -15.806451612903226 
               }
            },
            {
               "key_as_string": "2015/03/01 00:00:00",
               "key": 1425168000000,
               "doc_count": 2,
               "sales": {
                  "value": 375.0
               },
               "sales_deriv": {
                  "value": 315.0,
                  "normalized_value": 11.25
               }
            }
         ]
      }
   }
}
```

### 1.2. Moving Average聚合

（6.4.0以后废弃，使用Moving Function聚合替代）

### 1.3. Moving Function聚合

给定一系列有序的数据，移动函数聚合将在数据上滑动一个窗口，并允许用户指定在每个数据窗口上执行的自定义脚本。为了方便起见，预定义了许多常用函数，如min/max、移动平均值等。

这在概念上与移动平均管道聚合非常相似，只是它提供了更多的功能。

```json
{
    "moving_fn": {
        "buckets_path": "the_sum",
        "window": 10,
        "script": "MovingFunctions.min(values)"
    }
}
```

参数

| 参数名      | 说明                           | 必填 | 默认值 |
| ----------- | ------------------------------ | ---- | ------ |
| bucket_path | 我们希望计算的桶的路径         | 是   |        |
| window      | 要在直方图上“滑动”的窗口大小   | 是   |        |
| script      | 应该在每个数据窗口上执行的脚本 | 是   |        |

移动的聚集必须嵌入**直方图**或**日期直方图**聚集中。它们可以像任何其他度量聚合一样嵌入：

```json
POST /_search
{
    "size": 0,
    "aggs": {
        "my_date_histo":{                
            "date_histogram":{
                "field":"date",
                "calendar_interval":"1M"
            },
            "aggs":{
                "the_sum":{
                    "sum":{ "field": "price" } 
                },
                "the_movfn": {
                    "moving_fn": {
                        "buckets_path": "the_sum", 
                        "window": 10,
                        "script": "MovingFunctions.unweightedAvg(values)"
                    }
                }
            }
        }
    }
}
```

移动平均值是通过首先在字段上指定histogram或date_histogram来构建的。然后，您可以选择在该直方图内添加数字度量，例如总和。最后，在直方图中嵌入moving_fn。然后，buckets_path参数用于“指向”直方图内的一个同级度量（有关buckets_path语法的说明，请参阅buckets_path语法）。

例如上面的结果可能如下：

```js
{
   "took": 11,
   "timed_out": false,
   "_shards": ...,
   "hits": ...,
   "aggregations": {
      "my_date_histo": {
         "buckets": [
             {
                 "key_as_string": "2015/01/01 00:00:00",
                 "key": 1420070400000,
                 "doc_count": 3,
                 "the_sum": {
                    "value": 550.0
                 },
                 "the_movfn": {
                    "value": null
                 }
             },
             {
                 "key_as_string": "2015/02/01 00:00:00",
                 "key": 1422748800000,
                 "doc_count": 2,
                 "the_sum": {
                    "value": 60.0
                 },
                 "the_movfn": {
                    "value": 550.0
                 }
             },
             {
                 "key_as_string": "2015/03/01 00:00:00",
                 "key": 1425168000000,
                 "doc_count": 2,
                 "the_sum": {
                    "value": 375.0
                 },
                 "the_movfn": {
                    "value": 305.0
                 }
             }
         ]
      }
   }
}
```

#### 自定义计算脚本

移动函数聚合允许用户指定任意脚本来定义自定义逻辑。每次收集新的数据窗口时都会调用该脚本。这些值在values变量中提供给脚本。然后，脚本应该执行某种计算并发出一个double作为结果。不允许返回空值，但允许NaN和+/-Inf。

例如，此脚本只返回窗口中的第一个值，如果没有可用的值，则返回NaN：

```json
POST /_search
{
    "size": 0,
    "aggs": {
        "my_date_histo":{
            "date_histogram":{
                "field":"date",
                "calendar_interval":"1M"
            },
            "aggs":{
                "the_sum":{
                    "sum":{ "field": "price" }
                },
                "the_movavg": {
                    "moving_fn": {
                        "buckets_path": "the_sum",
                        "window": 10,
                        "script": "return values.length > 0 ? values[0] : Double.NaN"
                    }
                }
            }
        }
    }
}
```

#### 内建脚本

为了方便起见，已预先构建了许多函数，并可在移动脚本上下文中使用，这些函数可从MovingFunctions命名空间中获得。例如，MovingFunctions.max(values)

- `max(values)` 最大值
- `min(values)` 最小值
- `sum(values)` 求和
- `unweightedAvg(values)` 平均值
- `stdDev(values, avg)` 标准差 
- `linearWeightedAvg(values)`  线性权重均值，越老的数据对均值的贡献越小
- `ewma(values, alpha)` 单指数权重均值
- `holt(values, alpha, beta)` 双指数权重均值
- `holtWinters(values, alpha, beta, gamma, period, multiplicative)` 三指数权重均值

### 1.4. Cumulative Sum 聚合

累计求和：父管道聚合，计算父直方图（或日期直方图）聚合中指定度量的累积和。指定的度量必须是数字，并且封闭的直方图必须将“最小文档计数”设置为0（直方图聚合的默认值）。

```js
{
    "cumulative_sum": {
        "buckets_path": "the_sum"
    }
}
```

参数

| 参数名      | 说明                       | 必填 | 默认值 |
| ----------- | -------------------------- | ---- | ------ |
| bucket_path | 我们希望计算桶的路径       | 是   |        |
| format      | 应用于此聚合的输出值的格式 | 否   | null   |

以下代码段计算每月总销售额的累积和：

```json
POST /sales/_search
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                },
                "cumulative_sales": {
                    "cumulative_sum": {
                        "buckets_path": "sales" 
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
   "took": 11,
   "timed_out": false,
   "_shards": ...,
   "hits": ...,
   "aggregations": {
      "sales_per_month": {
         "buckets": [
            {
               "key_as_string": "2015/01/01 00:00:00",
               "key": 1420070400000,
               "doc_count": 3,
               "sales": {
                  "value": 550.0
               },
               "cumulative_sales": {
                  "value": 550.0
               }
            },
            {
               "key_as_string": "2015/02/01 00:00:00",
               "key": 1422748800000,
               "doc_count": 2,
               "sales": {
                  "value": 60.0
               },
               "cumulative_sales": {
                  "value": 610.0
               }
            },
            {
               "key_as_string": "2015/03/01 00:00:00",
               "key": 1425168000000,
               "doc_count": 2,
               "sales": {
                  "value": 375.0
               },
               "cumulative_sales": {
                  "value": 985.0
               }
            }
         ]
      }
   }
}
```

### 1.5. Bucket Script 聚合

父管道聚合，它执行一个脚本，该脚本可以对父多桶聚合中指定的度量执行每个桶的计算。指定的度量值必须是数字，脚本必须返回数字值。

```js
{
    "bucket_script": {
        "buckets_path": {
            "my_var1": "the_sum", 
            "my_var2": "the_value_count"
        },
        "script": "params.my_var1 / params.my_var2"
    }
}
```

参数

| 参数名      | 说明                       | 必填 | 默认值 |
| ----------- | -------------------------- | ---- | ------ |
| script      | 计算脚本                   | 是   |        |
| bucket_path | 我们希望计算桶的路径的map  | 是   |        |
| gap_policy  | 空隙数据处理策略           | 否   | skip   |
| format      | 应用于此聚合的输出值的格式 | 否   | null   |

以下代码片段计算了t恤衫销售额与每月总销售额的比率：

```json
POST /sales/_search
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "total_sales": {
                    "sum": {
                        "field": "price"
                    }
                },
                "t-shirts": {
                  "filter": {
                    "term": {
                      "type": "t-shirt"
                    }
                  },
                  "aggs": {
                    "sales": {
                      "sum": {
                        "field": "price"
                      }
                    }
                  }
                },
                "t-shirt-percentage": {
                    "bucket_script": {
                        "buckets_path": {
                          "tShirtSales": "t-shirts>sales",
                          "totalSales": "total_sales"
                        },
                        "script": "params.tShirtSales / params.totalSales * 100"
                    }
                }
            }
        }
    }
}
```

结果

```json
{
   "took": 11,
   "timed_out": false,
   "_shards": ...,
   "hits": ...,
   "aggregations": {
      "sales_per_month": {
         "buckets": [
            {
               "key_as_string": "2015/01/01 00:00:00",
               "key": 1420070400000,
               "doc_count": 3,
               "total_sales": {
                   "value": 550.0
               },
               "t-shirts": {
                   "doc_count": 1,
                   "sales": {
                       "value": 200.0
                   }
               },
               "t-shirt-percentage": {
                   "value": 36.36363636363637
               }
            },
            {
               "key_as_string": "2015/02/01 00:00:00",
               "key": 1422748800000,
               "doc_count": 2,
               "total_sales": {
                   "value": 60.0
               },
               "t-shirts": {
                   "doc_count": 1,
                   "sales": {
                       "value": 10.0
                   }
               },
               "t-shirt-percentage": {
                   "value": 16.666666666666664
               }
            },
            {
               "key_as_string": "2015/03/01 00:00:00",
               "key": 1425168000000,
               "doc_count": 2,
               "total_sales": {
                   "value": 375.0
               },
               "t-shirts": {
                   "doc_count": 1,
                   "sales": {
                       "value": 175.0
                   }
               },
               "t-shirt-percentage": {
                   "value": 46.666666666666664
               }
            }
         ]
      }
   }
}
```

### 1.6. Bucket Selector聚合

父管道聚合，它执行一个脚本，该脚本确定当前bucket是否将保留在父多bucket聚合中。指定的度量必须是数字，脚本必须返回布尔值。如果脚本语言是表达式，则允许数值返回值。在这种情况下，0.0将被计算为false和所有其他值。

> bucket\_selector聚合与所有管道聚合一样，在所有其他同级聚合之后执行。这意味着使用bucket\_selector聚合来筛选响应中返回的bucket不会节省运行聚合的执行时间。

```js
{
    "bucket_selector": {
        "buckets_path": {
            "my_var1": "the_sum", 
            "my_var2": "the_value_count"
        },
        "script": "params.my_var1 > params.my_var2"
    }
}
```

参数

| 参数名      | 说明                      | 必填 | 默认值 |
| ----------- | ------------------------- | ---- | ------ |
| script      | 计算脚本                  | 是   |        |
| bucket_path | 我们希望计算桶的路径的map | 是   |        |
| gap_policy  | 空隙数据处理策略          | 否   | skip   |

以下代码片段仅保留当月总销售额超过200的存储桶：

```json
POST /sales/_search
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "total_sales": {
                    "sum": {
                        "field": "price"
                    }
                },
                "sales_bucket_filter": {
                    "bucket_selector": {
                        "buckets_path": {
                          "totalSales": "total_sales"
                        },
                        "script": "params.totalSales > 200"
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
   "took": 11,
   "timed_out": false,
   "_shards": ...,
   "hits": ...,
   "aggregations": {
      "sales_per_month": {
         "buckets": [
            {
               "key_as_string": "2015/01/01 00:00:00",
               "key": 1420070400000,
               "doc_count": 3,
               "total_sales": {
                   "value": 550.0
               }
            },
            {
               "key_as_string": "2015/03/01 00:00:00",
               "key": 1425168000000,
               "doc_count": 2,
               "total_sales": {
                   "value": 375.0
               },
            }
         ]
      }
   }
}
```

### 1.7. Bucket Sort聚合

对其父多桶聚合的桶进行排序的父管道聚合。可以将零个或多个排序字段与相应的排序顺序一起指定。每个存储桶可以根据其key、计数或子聚合进行排序。此外，可以设置参数from和size以截断结果存储桶。

```js
{
    "bucket_sort": {
        "sort": [
            {"sort_field_1": {"order": "asc"}},
            {"sort_field_2": {"order": "desc"}},
            "sort_field_3"
        ],
        "from": 1,
        "size": 3
    }
}
```

参数

| 参数名     | 说明                                     | 必填 | 默认值 |
| ---------- | ---------------------------------------- | ---- | ------ |
| sort       | 排序字段列表                             | 否   |        |
| gap_policy | 在数据中发现空隙时应用的策略             | 否   | skip   |
| from       | 设置值之前位置的存储桶将被截断。         | 否   | 0      |
| size       | 要返回的桶数。默认为父聚合的所有存储桶。 | 否   |        |

以下代码段按降序返回与总销售额最高的3个月对应的存储桶：

```json
POST /sales/_search
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "total_sales": {
                    "sum": {
                        "field": "price"
                    }
                },
                "sales_bucket_sort": {
                    "bucket_sort": {
                        "sort": [
                          {"total_sales": {"order": "desc"}}
                        ],
                        "size": 3
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
   "took": 82,
   "timed_out": false,
   "_shards": ...,
   "hits": ...,
   "aggregations": {
      "sales_per_month": {
         "buckets": [
            {
               "key_as_string": "2015/01/01 00:00:00",
               "key": 1420070400000,
               "doc_count": 3,
               "total_sales": {
                   "value": 550.0
               }
            },
            {
               "key_as_string": "2015/03/01 00:00:00",
               "key": 1425168000000,
               "doc_count": 2,
               "total_sales": {
                   "value": 375.0
               },
            },
            {
               "key_as_string": "2015/02/01 00:00:00",
               "key": 1422748800000,
               "doc_count": 2,
               "total_sales": {
                   "value": 60.0
               },
            }
         ]
      }
   }
}
```

#### 不使用排序截断

也可以使用此聚合来截断结果存储桶，而不进行任何排序。为此，只需使用from和/或size参数而不指定sort。

```json
POST /sales/_search
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "bucket_truncate": {
                    "bucket_sort": {
                        "from": 1,
                        "size": 1
                    }
                }
            }
        }
    }
}
```

结果如下

```js
{
   "took": 11,
   "timed_out": false,
   "_shards": ...,
   "hits": ...,
   "aggregations": {
      "sales_per_month": {
         "buckets": [
            {
               "key_as_string": "2015/02/01 00:00:00",
               "key": 1422748800000,
               "doc_count": 2
            }
         ]
      }
   }
}
```

### 1.8. Serial Differencing聚合

序列差分是一种技术，在不同的时间间隔或周期内，从时间序列中减去其自身的值。例如，数据点 f(x) = f(xt) - f(xt-n)，其中n是正在使用的点。

周期为1相当于没有时间正规化的导数：它只是从一个点到下一个点的变化。单周期对于消除恒定的线性趋势很有用。

单周期对于将数据转换为平稳序列也很有用。在本例中，道琼斯指数的绘制时间约为250天。原始数据不是平稳的，这将使某些技术难以使用。

```js
{
    "serial_diff": {
        "buckets_path": "the_sum",
        "lag": "7"
    }
}
```

参数

| 参数名      | 说明                                                         | 必填 | 默认值 |
| ----------- | ------------------------------------------------------------ | ---- | ------ |
| bucket_path | 我们希望计算平均值的桶的路径                                 | 是   |        |
| lag         | 要从当前值中减去的历史存储桶。例如lag=7将从7个桶之前的值中减去当前值。必须是正的非零整数 | 否   | 1      |
| gap_policy  | 在数据中发现空隙时应用的策略                                 | 否   | skip   |
| format      | 应用于此聚合的输出值的格式                                   | 否   | null   |

序列差异聚合必须嵌入直方图或日期直方图聚合中：

```json
POST /_search
{
   "size": 0,
   "aggs": {
      "my_date_histo": {                  
         "date_histogram": {
            "field": "timestamp",
            "calendar_interval": "day"
         },
         "aggs": {
            "the_sum": {
               "sum": {
                  "field": "lemmings"     
               }
            },
            "thirtieth_difference": {
               "serial_diff": {                
                  "buckets_path": "the_sum",
                  "lag" : 30
               }
            }
         }
      }
   }
}
```

通过首先在字段上指定直方图或日期直方图来建立序列差分。然后，您可以选择在该直方图内添加常规度量，例如总和。最后，将序列差分嵌入直方图中。然后，buckets_path参数用于“指向”直方图内的一个同级度量（有关buckets_path语法的说明，请参阅buckets_path语法）。

## 2. Slibing

### 2.1. Avg Bucket聚合

计算同级聚合中指定度量的（平均）平均值的同级管道聚合。指定的度量必须是数字，而同级聚合必须是多桶聚合。

```json
{
    "avg_bucket": {
        "buckets_path": "the_sum"
    }
}
```

参数

| 参数名      | 说明                         | 必填 | 默认值 |
| ----------- | ---------------------------- | ---- | ------ |
| bucket_path | 我们希望计算平均值的桶的路径 | 是   |        |
| gap_policy  | 在数据中发现空隙时应用的策略 | 否   | skip   |
| format      | 应用于此聚合的输出值的格式   | 否   | null   |

以下代码段计算每月总销售额的平均值：

```json
POST /_search
{
  "size": 0,
  "aggs": {
    "sales_per_month": {
      "date_histogram": {
        "field": "date",
        "calendar_interval": "month"
      },
      "aggs": {
        "sales": {
          "sum": {
            "field": "price"
          }
        }
      }
    },
    "avg_monthly_sales": {
      "avg_bucket": {
        "buckets_path": "sales_per_month>sales" 
      }
    }
  }
}
```

结果

```json
{
   "took": 11,
   "timed_out": false,
   "_shards": ...,
   "hits": ...,
   "aggregations": {
      "sales_per_month": {
         "buckets": [
            {
               "key_as_string": "2015/01/01 00:00:00",
               "key": 1420070400000,
               "doc_count": 3,
               "sales": {
                  "value": 550.0
               }
            },
            {
               "key_as_string": "2015/02/01 00:00:00",
               "key": 1422748800000,
               "doc_count": 2,
               "sales": {
                  "value": 60.0
               }
            },
            {
               "key_as_string": "2015/03/01 00:00:00",
               "key": 1425168000000,
               "doc_count": 2,
               "sales": {
                  "value": 375.0
               }
            }
         ]
      },
      "avg_monthly_sales": {
          "value": 328.33333333333333
      }
   }
}
```

### 2.2. Max Bucket 聚合

同级管道聚合，用同级聚合中指定度量的最大值标识存储桶，并输出存储桶的值和键。指定的度量必须是数字，而同级聚合必须是多桶聚合。

```json
{
    "max_bucket": {
        "buckets_path": "the_sum"
    }
}
```

参数

| 参数名      | 说明                         | 必填 | 默认值 |
| ----------- | ---------------------------- | ---- | ------ |
| bucket_path | 我们希望计算最大值的桶的路径 | 是   |        |
| gap_policy  | 在数据中发现空隙时应用的策略 | 否   | skip   |
| format      | 应用于此聚合的输出值的格式   | 否   | null   |

以下代码段计算每月总销售额的最大值：

```json
POST /sales/_search
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                }
            }
        },
        "max_monthly_sales": {
            "max_bucket": {
                "buckets_path": "sales_per_month>sales" 
            }
        }
    }
}
```

结果

```json
{
   "took": 11,
   "timed_out": false,
   "_shards": ...,
   "hits": ...,
   "aggregations": {
      "sales_per_month": {
         "buckets": [
            {
               "key_as_string": "2015/01/01 00:00:00",
               "key": 1420070400000,
               "doc_count": 3,
               "sales": {
                  "value": 550.0
               }
            },
            {
               "key_as_string": "2015/02/01 00:00:00",
               "key": 1422748800000,
               "doc_count": 2,
               "sales": {
                  "value": 60.0
               }
            },
            {
               "key_as_string": "2015/03/01 00:00:00",
               "key": 1425168000000,
               "doc_count": 2,
               "sales": {
                  "value": 375.0
               }
            }
         ]
      },
      "max_monthly_sales": {
          "keys": ["2015/01/01 00:00:00"], // 最大值结果可能是多个桶
          "value": 550.0
      }
   }
}
```

### 2.3. Min Bucket聚合

同级管道聚合，用同级聚合中指定度量的最小值标识存储桶，并输出存储桶的值和键。指定的度量必须是数字，而同级聚合必须是多桶聚合。

```json
{
    "min_bucket": {
        "buckets_path": "the_sum"
    }
}
```

参数

| 参数名      | 说明                         | 必填 | 默认值 |
| ----------- | ---------------------------- | ---- | ------ |
| bucket_path | 我们希望计算最小值的桶的路径 | 是   |        |
| gap_policy  | 在数据中发现空隙时应用的策略 | 否   | skip   |
| format      | 应用于此聚合的输出值的格式   | 否   | null   |

以下代码段计算每月总销售额的最小值：

```json
POST /sales/_search
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                }
            }
        },
        "min_monthly_sales": {
            "min_bucket": {
                "buckets_path": "sales_per_month>sales" 
            }
        }
    }
}
```

结果

```json
{
   "took": 11,
   "timed_out": false,
   "_shards": ...,
   "hits": ...,
   "aggregations": {
      "sales_per_month": {
         "buckets": [
            {
               "key_as_string": "2015/01/01 00:00:00",
               "key": 1420070400000,
               "doc_count": 3,
               "sales": {
                  "value": 550.0
               }
            },
            {
               "key_as_string": "2015/02/01 00:00:00",
               "key": 1422748800000,
               "doc_count": 2,
               "sales": {
                  "value": 60.0
               }
            },
            {
               "key_as_string": "2015/03/01 00:00:00",
               "key": 1425168000000,
               "doc_count": 2,
               "sales": {
                  "value": 375.0
               }
            }
         ]
      },
      "min_monthly_sales": {
          "keys": ["2015/02/01 00:00:00"], 
          "value": 60.0
      }
   }
}
```

### 2.4. Sum Bucket聚合

同级管道聚合，用同级聚合中指定度量的求和标识存储桶，并输出存储桶的值和键。指定的度量必须是数字，而同级聚合必须是多桶聚合。

```json
{
    "sum_bucket": {
        "buckets_path": "the_sum"
    }
}
```

参数

| 参数名      | 说明                         | 必填 | 默认值 |
| ----------- | ---------------------------- | ---- | ------ |
| bucket_path | 我们希望计算求和的桶的路径   | 是   |        |
| gap_policy  | 在数据中发现空隙时应用的策略 | 否   | skip   |
| format      | 应用于此聚合的输出值的格式   | 否   | null   |

以下代码段计算每月总销售额的汇总：

```json
POST /sales/_search
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                }
            }
        },
        "sum_monthly_sales": {
            "sum_bucket": {
                "buckets_path": "sales_per_month>sales" 
            }
        }
    }
}
```

结果

```js
{
   "took": 11,
   "timed_out": false,
   "_shards": ...,
   "hits": ...,
   "aggregations": {
      "sales_per_month": {
         "buckets": [
            {
               "key_as_string": "2015/01/01 00:00:00",
               "key": 1420070400000,
               "doc_count": 3,
               "sales": {
                  "value": 550.0
               }
            },
            {
               "key_as_string": "2015/02/01 00:00:00",
               "key": 1422748800000,
               "doc_count": 2,
               "sales": {
                  "value": 60.0
               }
            },
            {
               "key_as_string": "2015/03/01 00:00:00",
               "key": 1425168000000,
               "doc_count": 2,
               "sales": {
                  "value": 375.0
               }
            }
         ]
      },
      "sum_monthly_sales": {
          "value": 985.0
      }
   }
}
```

### 2.5. Stats Bucket聚合

同级管道聚合，用同级聚合中指定度量的统计值标识存储桶，并输出存储桶的值和键。指定的度量必须是数字，而同级聚合必须是多桶聚合。

```json
{
    "stats_bucket": {
        "buckets_path": "the_sum"
    }
}
```

参数

| 参数名      | 说明                         | 必填 | 默认值 |
| ----------- | ---------------------------- | ---- | ------ |
| bucket_path | 我们希望计算统计量的桶的路径 | 是   |        |
| gap_policy  | 在数据中发现空隙时应用的策略 | 否   | skip   |
| format      | 应用于此聚合的输出值的格式   | 否   | null   |

以下代码段计算每月总销售额的统计量：

```json
POST /sales/_search
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                }
            }
        },
        "stats_monthly_sales": {
            "stats_bucket": {
                "buckets_path": "sales_per_month>sales" 
            }
        }
    }
}
```

结果

```js
{
   "took": 11,
   "timed_out": false,
   "_shards": ...,
   "hits": ...,
   "aggregations": {
      "sales_per_month": {
         "buckets": [
            {
               "key_as_string": "2015/01/01 00:00:00",
               "key": 1420070400000,
               "doc_count": 3,
               "sales": {
                  "value": 550.0
               }
            },
            {
               "key_as_string": "2015/02/01 00:00:00",
               "key": 1422748800000,
               "doc_count": 2,
               "sales": {
                  "value": 60.0
               }
            },
            {
               "key_as_string": "2015/03/01 00:00:00",
               "key": 1425168000000,
               "doc_count": 2,
               "sales": {
                  "value": 375.0
               }
            }
         ]
      },
      "stats_monthly_sales": {
         "count": 3,
         "min": 60.0,
         "max": 550.0,
         "avg": 328.3333333333333,
         "sum": 985.0
      }
   }
}
```

### 2.6. Extended Stats Bucket聚合

同级管道聚合，用同级聚合中指定度量的扩展统计值标识存储桶，并输出存储桶的值和键。指定的度量必须是数字，而同级聚合必须是多桶聚合。

```json
{
    "extended_stats_bucket": {
        "buckets_path": "the_sum"
    }
}
```

参数

| 参数名      | 说明                             | 必填 | 默认值 |
| ----------- | -------------------------------- | ---- | ------ |
| bucket_path | 我们希望计算扩展统计量的桶的路径 | 是   |        |
| gap_policy  | 在数据中发现空隙时应用的策略     | 否   | skip   |
| format      | 应用于此聚合的输出值的格式       | 否   | null   |

以下代码段计算每月总销售额的扩展统计量：

```json
POST /sales/_search
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                }
            }
        },
        "stats_monthly_sales": {
            "extended_stats_bucket": {
                "buckets_path": "sales_per_month>sales" 
            }
        }
    }
}
```

结果

```js
{
   "took": 11,
   "timed_out": false,
   "_shards": ...,
   "hits": ...,
   "aggregations": {
      "sales_per_month": {
         "buckets": [
            {
               "key_as_string": "2015/01/01 00:00:00",
               "key": 1420070400000,
               "doc_count": 3,
               "sales": {
                  "value": 550.0
               }
            },
            {
               "key_as_string": "2015/02/01 00:00:00",
               "key": 1422748800000,
               "doc_count": 2,
               "sales": {
                  "value": 60.0
               }
            },
            {
               "key_as_string": "2015/03/01 00:00:00",
               "key": 1425168000000,
               "doc_count": 2,
               "sales": {
                  "value": 375.0
               }
            }
         ]
      },
      "stats_monthly_sales": {
         "count": 3,
         "min": 60.0,
         "max": 550.0,
         "avg": 328.3333333333333,
         "sum": 985.0,
         "sum_of_squares": 446725.0,
         "variance": 41105.55555555556,
         "std_deviation": 202.74505063146563,
         "std_deviation_bounds": {
           "upper": 733.8234345962646,
           "lower": -77.15676792959795
         }
      }
   }
}
```

### 2.7. Percentiles Bucket聚合

一种同级管道聚合，它计算同级聚合中指定度量的所有存储桶的百分比。指定的度量必须是数字，而同级聚合必须是多桶聚合。

```
{
    "percentiles_bucket": {
        "buckets_path": "the_sum"
    }
}
```

参数

| 参数名      | 说明                           | 必填 | 默认值                       |
| ----------- | ------------------------------ | ---- | ---------------------------- |
| bucket_path | 我们希望计算扩百分比的桶的路径 | 是   |                              |
| gap_policy  | 在数据中发现空隙时应用的策略   | 否   | skip                         |
| format      | 应用于此聚合的输出值的格式     | 否   | null                         |
| percents    | 需要计算的百分比列表           | 否   | [ 1, 5, 25, 50, 75, 95, 99 ] |
| keyed       | 是否指定返回map代替数组        | 否   | true                         |

以下代码段计算每月总销售量的百分比：

```json
POST /sales/_search
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                }
            }
        },
        "percentiles_monthly_sales": {
            "percentiles_bucket": {
                "buckets_path": "sales_per_month>sales", 
                "percents": [ 25.0, 50.0, 75.0 ] 
            }
        }
    }
}
```

结果

```js
{
   "took": 11,
   "timed_out": false,
   "_shards": ...,
   "hits": ...,
   "aggregations": {
      "sales_per_month": {
         "buckets": [
            {
               "key_as_string": "2015/01/01 00:00:00",
               "key": 1420070400000,
               "doc_count": 3,
               "sales": {
                  "value": 550.0
               }
            },
            {
               "key_as_string": "2015/02/01 00:00:00",
               "key": 1422748800000,
               "doc_count": 2,
               "sales": {
                  "value": 60.0
               }
            },
            {
               "key_as_string": "2015/03/01 00:00:00",
               "key": 1425168000000,
               "doc_count": 2,
               "sales": {
                  "value": 375.0
               }
            }
         ]
      },
      "percentiles_monthly_sales": {
        "values" : {
            "25.0": 375.0,
            "50.0": 375.0,
            "75.0": 550.0
         }
      }
   }
}
```

> 百分位桶返回最近的输入数据点，该点不大于请求的百分位；它不在数据点之间进行插值。
>
> 百分位数是精确计算的，不是近似值（与百分位数度量不同）。这意味着在丢弃数据之前，实现会维护一个内存中的数据排序列表，以计算百分比。如果试图在单个百分位存储桶中计算数百万个数据点上的百分位，可能会遇到内存压力问题。