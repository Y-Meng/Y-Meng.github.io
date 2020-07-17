# 矩阵聚合

此功能是实验性的，在将来的版本中可能会完全更改或删除。Elastic将尽最大努力解决任何问题，但实验特性不受官方GA特性的支持服务水平协议的约束。

此系列中的聚合操作多个字段，并基于从请求的文档字段中提取的值生成矩阵结果。与度量和bucket聚合不同，此聚合系列尚不支持脚本。

## 矩阵统计

matrix_stats聚合是一个数字聚合，它计算一组文档字段上的以下统计信息：



| 统计量      | 说明                                                         |
| ----------- | ------------------------------------------------------------ |
| count       | 计算中包含的每个字段样本数。                                 |
| mean        | 每个字段平均值                                               |
| variance    | 每个测量样品的分布情况。                                     |
| skewness    | 测量量化平均值周围的不对称分布。                             |
| kurtosis    | 测量量化分布的形状。                                         |
| covariance  | 定量描述一个领域的变化如何与另一个领域相联系的矩阵。         |
| correlation | 协方差矩阵缩放到-1到1的范围（包括-1到1）。描述场分布之间的关系。 |

下面的例子演示了使用矩阵统计来描述收入与贫困之间的关系。

```json
GET /_search
{
    "aggs": {
        "statistics": {
            "matrix_stats": {
                "fields": ["poverty", "income"]
            }
        }
    }
}
```

结果

```js
{
    ...
    "aggregations": {
        "statistics": {
            "doc_count": 50,
            "fields": [{
                "name": "income",
                "count": 50,
                "mean": 51985.1,
                "variance": 7.383377037755103E7,
                "skewness": 0.5595114003506483,
                "kurtosis": 2.5692365287787124,
                "covariance": {
                    "income": 7.383377037755103E7,
                    "poverty": -21093.65836734694
                },
                "correlation": {
                    "income": 1.0,
                    "poverty": -0.8352655256272504
                }
            }, {
                "name": "poverty",
                "count": 50,
                "mean": 12.732000000000001,
                "variance": 8.637730612244896,
                "skewness": 0.4516049811903419,
                "kurtosis": 2.8615929677997767,
                "covariance": {
                    "income": -21093.65836734694,
                    "poverty": 8.637730612244896
                },
                "correlation": {
                    "income": -0.8352655256272504,
                    "poverty": 1.0
                }
            }]
        }
    }
}
```

matrix_stats聚合将每个文档字段视为一个独立的样本。mode参数控制聚合将用于数组或多值字段的数组值。此参数可以采用下列参数之一：

| 参数     | 说明                                     |
| -------- | ---------------------------------------- |
| `avg`    | (default) Use the average of all values. |
| `min`    | Pick the lowest value.                   |
| `max`    | Pick the highest value.                  |
| `sum`    | Use the sum of all values.               |
| `median` | Use the median of all values.            |

缺失值

missing参数定义了如何处理缺少值的文档。默认情况下，它们将被忽略，但也可以将它们视为具有值。这是通过添加一组fieldname:value映射来完成的，以指定每个字段的默认值。

```json
GET /_search
{
    "aggs": {
        "matrixstats": {
            "matrix_stats": {
                "fields": ["poverty", "income"],
                "missing": {"income" : 50000} 
            }
        }
    }
}
```