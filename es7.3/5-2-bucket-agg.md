# 桶聚合（Bucket）

## 1.Terms聚合

一个基于多bucket值源的聚合，其中bucket是动态构建的，每个唯一值一个。

```json
GET /_search
{
    "aggs" : {
        "genres" : {
            "terms" : { "field" : "genre" } // 1
        }
    }
}
```

>  Terms聚合应该是keyword类型的字段或任何其他适合于bucket聚合的数据类型。为了在text字段中使用它，您需要启用fielddata。

结果

```json
{
    ...
    "aggregations" : {
        "genres" : {
            "doc_count_error_upper_bound": 0, // 1
            "sum_other_doc_count": 0, // 2
            "buckets" : [ // 3
                {
                    "key" : "electronic",
                    "doc_count" : 6
                },
                {
                    "key" : "rock",
                    "doc_count" : 3
                },
                {
                    "key" : "jazz",
                    "doc_count" : 2
                }
            ]
        }
    }
}
```

> 1.每个term文档上错误的上限
>
> 2.当有很多唯一值时，Elasticsearch只返回最上面的term；这个数字是所有不属于响应的bucket的文档计数的总和
>
> 3.顶部桶的列表，顶部的含义由排序定义

默认情况下，Terms聚合将返回按文档计数排序的前十个Terms的存储桶。可以通过设置size参数来更改此默认值。

### size

可以设置size参数来定义应该从整个Terms列表中返回多少Terms桶。默认情况下，协调搜索过程的节点将请求每个shard提供自己的最大字词存储桶，一旦所有shard响应，它将把结果减少到最终列表，然后返回给客户端。这意味着，如果唯一Terms的数目大于size，则返回的列表会稍有偏离且不准确（可能是Terms计数稍有偏离，甚至可能是本应位于最大大小存储桶中的Terms未返回）。

如果要在嵌套Terms聚合中检索所有Terms或Terms的所有组合，则应使用复合聚合，该聚合允许对所有可能的Terms进行分页，而不是在Terms聚合中设置大于字段基数的大小。Terms聚合旨在返回顶级Terms，不允许分页。

### doc_count是近似的

如上所述，Terms聚合中的文档计数（以及任何子聚合的结果）并不总是准确的。这是因为每个shard都提供了它自己的视图，说明了Terms的有序列表应该是什么，并将它们组合起来给出了最终的视图。考虑以下场景：

请求从包含3个分片的索引中按文档计数降序获取字段产品中的前5个Terms。在这种情况下，每个分片被要求给出它的前5个条件。

```json
GET /_search
{
    "aggs" : {
        "products" : {
            "terms" : {
                "field" : "product",
                "size" : 5
            }
        }
    }
}
```

三个分片的Terms如下所示，其各自的文档计数在括号中：

![image-20200328162115102](..\_images\image-20200328162115102.png)



分片将返回他们的前5项，因此分片的结果将是：

![image-20200328162238374](..\_images\image-20200328162238374.png)

将每个分片的前5个结果（根据要求）合并到最后的前5个列表中，将生成以下结果：

![image-20200328162321698](..\_images\image-20200328162321698.png)

因为产品A是从所有分片返回的，我们知道它的文档计数值是准确的。产品C仅由分片A和C返回，因此其文档计数显示为50，但这不是一个准确的计数。产品C存在于分片B上，但它的4计数不够高，无法将产品C放入该分片的前5个列表中。产品Z也仅由2个分片返回，但第三个分片不包含该Terms。在合并结果以生成最终Terms列表时，无法知道产品C和产品Z的文档计数中存在错误。产品H在所有3个分片中的文档计数均为44，但未包含在最终Terms列表中，因为它未在以下任何一项上进入前五名分片。

### shard size分片数量

请求的大小越大，结果就越准确，但计算最终结果的成本也越高（这两个原因都是在shard级别管理的优先级队列越大，以及节点和客户端之间的数据传输越大）。

shard_size参数可用于最小化请求的较大大小所带来的额外工作。定义后，它将确定协调节点将从每个shard请求多少Terms。一旦所有分片都响应了，协调节点就会将它们减少到最终结果，最终结果将基于size参数-这样，就可以提高返回项的准确性，并避免将大量存储桶流回到客户端的开销。

shard_size不能小于size（因为它没有多大意义）。如果是，Elasticsearch将覆盖它并将其重置为等于size。

默认的 `shard_size` = `(size * 1.5 + 10)`.

### 计算文档数量错误

有两个错误值可以显示在Terms聚合上。第一个给出了聚合的整体值，该值表示未进入最终Terms列表的Terms的最大潜在文档计数。这是根据从每个分片返回的最后一个Terms的文档计数之和计算的。对于上面给出的示例，该值为46（2+15+29）。这意味着在最坏的情况下，未返回的Terms可能具有第四高的文档计数。

```json
{
    ...
    "aggregations" : {
        "products" : {
            "doc_count_error_upper_bound" : 46,
            "sum_other_doc_count" : 79,
            "buckets" : [
                {
                    "key" : "Product A",
                    "doc_count" : 100
                },
                {
                    "key" : "Product Z",
                    "doc_count" : 52
                }
                ...
            ]
        }
    }
}
```



### 每个桶文档数量错误

通过将show_term_doc_count_error参数设置为true，可以启用第二个错误值：

```json
GET /_search
{
    "aggs" : {
        "products" : {
            "terms" : {
                "field" : "product",
                "size" : 5,
                "show_term_doc_count_error": true
            }
        }
    }
}
```

这将显示聚合返回的每个Terms的错误值，该值表示文档计数中最坏的情况错误，在决定shard_size参数的值时非常有用。这是通过将所有未返回Terms的分片返回的上一个Terms的文档计数相加来计算的。在上面的例子中，产品C的文档计数中的错误是15，因为Shard B是唯一不返回Terms的Shard，它返回的最后一个Terms的文档计数是15。产品C的实际文档计数是54，因此文档计数实际上只有4，即使最坏的情况是它将被关闭15。但是，产品A的文档计数有一个0的错误，因为每个分片都返回了它，我们可以确信返回的计数是准确的。

```json
{
    ...
    "aggregations" : {
        "products" : {
            "doc_count_error_upper_bound" : 46,
            "sum_other_doc_count" : 79,
            "buckets" : [
                {
                    "key" : "Product A",
                    "doc_count" : 100,
                    "doc_count_error_upper_bound" : 0
                },
                {
                    "key" : "Product Z",
                    "doc_count" : 52,
                    "doc_count_error_upper_bound" : 2
                }
                ...
            ]
        }
    }
}
```

只有按文档计数降序排列Terms时，才能用这种方式计算这些错误。当聚合按Terms值本身排序（升序或降序）时，文档计数中没有错误，因为如果一个shard不返回出现在另一个shard的结果中的特定Terms，则它的索引中不能包含该Terms。当聚合按子聚合排序或按文档计数升序排序时，无法确定文档计数中的错误，并为其提供-1值以指示此情况。

### 顺序

通过设置order参数，可以自定义bucket的顺序。默认情况下，存储桶按其文档计数降序排列。可以改变这种行为，如下所述：

> 不鼓励按升序计数或按子聚合排序，因为这样会增加文档计数的错误。当查询单个分片时，或者当正在聚合的字段在索引时用作路由密钥时，这是很好的：在这些情况下，结果将是准确的，因为分片具有不相交的值。然而，否则，错误是无限的。一个仍然有用的特殊情况是按最小或最大聚合排序：计数将不准确，但至少会正确地选择顶部的存储桶。

按文档数升序排列

```json
GET /_search
{
    "aggs" : {
        "genres" : {
            "terms" : {
                "field" : "genre",
                "order" : { "_count" : "asc" }
            }
        }
    }
}
```

按Term字典序升序排列

```json
GET /_search
{
    "aggs" : {
        "genres" : {
            "terms" : {
                "field" : "genre",
                "order" : { "_key" : "asc" }
            }
        }
    }
}
```

按单值度量对子聚合排序存储桶（由聚合名称标识）：

```json
GET /_search
{
    "aggs" : {
        "genres" : {
            "terms" : {
                "field" : "genre",
                "order" : { "max_play_count" : "desc" }
            },
            "aggs" : {
                "max_play_count" : { "max" : { "field" : "play_count" } }
            }
        }
    }
}
```

按多值度量对子聚合排序存储桶（由聚合名称标识）：

```json
GET /_search
{
    "aggs" : {
        "genres" : {
            "terms" : {
                "field" : "genre",
                "order" : { "playback_stats.max" : "desc" }
            },
            "aggs" : {
                "playback_stats" : { "stats" : { "field" : "play_count" } }
            }
        }
    }
}
```

> 管道聚合（pipeline aggs）不能用于排序。

还可以基于层次结构中的“更深层”聚合来订购存储桶。只要聚合路径是单个bucket类型，则支持此操作，其中路径中的最后一个聚合可以是单个bucket类型，也可以是metrics类型。如果是单个bucket类型，则顺序将由bucket中的文档数（即doc_count）定义，如果是metric s类型，则应用与上面相同的规则（如果是多值metrics聚合，则路径必须指明要排序的metric名称，如果是单值metrics聚合，则应用排序在这个价值上）。

路径格式如下：

```properties
AGG_SEPARATOR       =  '>' ;
METRIC_SEPARATOR    =  '.' ;
AGG_NAME            =  <the name of the aggregation> ;
METRIC              =  <the name of the metric (in case of multi-value metrics aggregation)> ;
PATH                =  <AGG_NAME> [ <AGG_SEPARATOR>, <AGG_NAME> ]* [ <METRIC_SEPARATOR>, <METRIC> ] ;
```

```json
GET /_search
{
    "aggs" : {
        "countries" : {
            "terms" : {
                "field" : "artist.country",
                "order" : { "rock>playback_stats.avg" : "desc" }
            },
            "aggs" : {
                "rock" : {
                    "filter" : { "term" : { "genre" :  "rock" }},
                    "aggs" : {
                        "playback_stats" : { "stats" : { "field" : "play_count" }}
                    }
                }
            }
        }
    }
}
```

以上将根据摇滚歌曲中的平均播放次数对艺术家的国家桶进行排序。



可以使用多个条件对存储桶进行排序，方法是提供一系列排序条件，例如：

```json
GET /_search
{
    "aggs" : {
        "countries" : {
            "terms" : {
                "field" : "artist.country",
                "order" : [ { "rock>playback_stats.avg" : "desc" }, { "_count" : "desc" } ]
            },
            "aggs" : {
                "rock" : {
                    "filter" : { "term" : { "genre" : "rock" }},
                    "aggs" : {
                        "playback_stats" : { "stats" : { "field" : "play_count" }}
                    }
                }
            }
        }
    }
}
```

以上将根据摇滚歌曲中的平均播放次数对艺术家的国家/地区桶进行排序，然后按其doc_计数降序排列。

如果两个bucket在所有order条件下共享相同的值，bucket的term value将用作按字母升序排列的连接断路器，以防止bucket的顺序不确定。



### 最小文档数量

使用min_doc_count选项，可以只返回匹配超过配置命中数的Terms：

```json
GET /_search
{
    "aggs" : {
        "tags" : {
            "terms" : {
                "field" : "tags",
                "min_doc_count": 10
            }
        }
    }
}
```

上面的聚合将只返回在10次或更多命中中找到的标记。默认值为1。

Terms在shard级别收集和排序，并在第二步中与从其他shard收集的Terms合并。但是，shard没有关于全局文档计数的可用信息。如果将一个Terms添加到候选列表中，则决策仅取决于使用本地切分频率在切分上计算的顺序。只有在合并所有分片的本地Terms统计之后，才会应用最小文档计数标准。在某种程度上，决定增加一个候选人的任期是没有非常确定的任期是否真的会达到所需的最低博士人数。如果候选列表中填充了低频项，这可能会导致最终结果中缺少许多（全局）高频项。为了避免这种情况，可以增加shard_size参数，以便在shard上允许更多的候选项。但是，这会增加内存消耗和网络流量。

`shard_min_doc_count` 参数：

参数shard_min_doc_count调节shard是否确实应该将该Terms添加到候选列表中（与min_doc_count相关）。只有当其在集合内的本地分片频率高于shard_min_doc_count时，才会考虑Terms。如果字典中包含许多低频Terms，而您对这些Terms不感兴趣（例如拼写错误），则可以设置shard_min_doc_count参数，以筛选出分片级别上的候选Terms，即使合并了本地计数，这些候选Terms仍有合理的确定性，无法达到所需的min_doc_count。shard_min_doc_count默认设置为0，除非显式设置，否则无效。

### 脚本

通过脚本生成term

```json
GET /_search
{
    "aggs" : {
        "genres" : {
            "terms" : {
                "script" : {
                    "source": "doc['genre'].value",
                    "lang": "painless"
                }
            }
        }
    }
}
```

这将把脚本参数解释为具有默认脚本语言且没有脚本参数的内联脚本。要使用存储的脚本，请使用以下语法：

```json
GET /_search
{
    "aggs" : {
        "genres" : {
            "terms" : {
                "script" : {
                    "id": "my_script",
                    "params": {
                        "field": "genre"
                    }
                }
            }
        }
    }
}
```

### 值脚本

```json
GET /_search
{
    "aggs" : {
        "genres" : {
            "terms" : {
                "field" : "genre",
                "script" : {
                    "source" : "'Genre: ' +_value",
                    "lang" : "painless"
                }
            }
        }
    }
}
```

### 过滤值

可以筛选将为其创建存储桶的值。这可以使用include和exclude参数来完成，这些参数基于正则表达式字符串或精确值数组。此外，include子句可以使用正则表达式进行筛选。

* 使用正则表达式过滤

```json
GET /_search
{
    "aggs" : {
        "tags" : {
            "terms" : {
                "field" : "tags",
                "include" : ".*sport.*",
                "exclude" : "water_.*"
            }
        }
    }
}
```

在上面的示例中，将为其中包含单词sport的所有标记创建桶，但以water_u开头的标记除外（因此不会聚合标记water_usports）。include正则表达式将确定“允许”聚合哪些值，而exclude将确定不应聚合的值。当两者都被定义时，exclude具有优先权，这意味着先计算include，然后才计算exclude。

* 使用精确值过滤

对于基于精确值的匹配，include和exclude参数可以采用一个字符串数组，表示在索引中找到的Terms：

```json
GET /_search
{
    "aggs" : {
        "JapaneseCars" : {
             "terms" : {
                 "field" : "make",
                 "include" : ["mazda", "honda"]
             }
         },
        "ActiveCarManufacturers" : {
             "terms" : {
                 "field" : "make",
                 "exclude" : ["rover", "jensen"]
             }
         }
    }
}
```

* 使用分区表达式

有时，在一个请求/响应对中有太多的独特Terms需要处理，因此将分析分解为多个请求是很有用的。这可以通过在查询时将字段的值分组为多个分区并在每个请求中仅处理一个分区来实现。请考虑此请求，它正在查找最近未登录任何访问权限的帐户：

```json
GET /_search
{
   "size": 0,
   "aggs": {
      "expired_sessions": {
         "terms": {
            "field": "account_id",
            "include": {
               "partition": 0,
               "num_partitions": 20
            },
            "size": 10000,
            "order": {
               "last_access": "asc"
            }
         },
         "aggs": {
            "last_access": {
               "max": {
                  "field": "access_date"
               }
            }
         }
      }
   }
}
```

此请求正在查找某个客户帐户子集的最后一个登录访问日期，因为我们可能要使某些已很长时间未被看到的客户帐户过期。num_partitions设置要求将唯一的帐户id均匀地组织为20个分区（0到19）。并且此请求中的分区设置筛选为只考虑分区0中的帐户ID。随后的请求应该要求分区1然后分区2等来完成过期帐户分析。

注意，返回的结果数的size设置需要使用num_partitions进行调整。对于这个特定的帐户过期示例，用于平衡size和num_partitions的值的过程如下：

1. 使用基数聚合估计唯一帐户id值的总数
2. 为num_partitions选择一个值，将从1开始的数字分解为更易于管理的块
3. 为每个分区的响应数选择一个size值
4. 运行一个测试请求

如果我们有一个断路器错误，我们试图在一个请求中做太多，必须增加num_partitions。如果请求成功，但按日期排序的测试响应中的最后一个帐户ID仍然是一个我们可能要过期的帐户，则我们可能缺少感兴趣的帐户，并且将我们的号码设置得太低。我们可以按如下步骤做：

* 增加size参数以返回每个分区的更多结果（可能会占用大量内存）或
* 增加num_partitions以考虑每个请求更少的帐户（可能会增加总体处理时间，因为我们需要发出更多请求）
* 最终，这是在管理处理单个请求所需的Elasticsearch资源和客户端应用程序必须发出以完成任务的请求量之间的平衡。

### 多字段Terms聚合

Terms聚合不支持从同一文档中的多个字段收集Terms。原因是Termsagg不收集字符串Terms值本身，而是使用全局序数生成字段中所有唯一值的列表。全局序数导致了一个重要的性能提升，这在多个字段中是不可能的。

有两种方法可用于跨多个字段执行Termsagg：

* 脚本

使用脚本从多个字段检索Terms。这将禁用全局序数优化，并将比从单个字段中收集Terms慢，但它为您提供了在搜索时实现此选项的灵活性。

* copy_to字段

如果提前知道要从两个或多个字段中收集Terms，请在映射中使用copy_to在索引时创建一个新的专用字段，该字段包含两个字段中的值。您可以在此单个字段上进行聚合，这将受益于全局序数优化。

### 收集模式

推迟计算子聚合

对于具有许多唯一项和少量必需结果的字段，可以更有效地延迟子聚合的计算，直到删除顶级父级agg。通常，聚合树的所有分支都在一个深度优先过程中展开，然后才进行任何修剪。在某些情况下，这可能会非常浪费，并可能会影响内存限制。一个示例问题场景是查询电影数据库以查找10位最受欢迎的演员及其5位最常见的联合主演：

```json
GET /_search
{
    "aggs" : {
        "actors" : {
             "terms" : {
                 "field" : "actors",
                 "size" : 10
             },
            "aggs" : {
                "costars" : {
                     "terms" : {
                         "field" : "actors",
                         "size" : 5
                     }
                 }
            }
         }
    }
}
```

尽管参与者的数量可能相对较少，而且我们只需要50个结果桶，但是在计算过程中，桶的组合爆炸-一个参与者可以生成n个桶，其中n是参与者的数量。明智的选择是，首先确定10位最受欢迎的演员，然后再为这10位演员考察最受欢迎的联合主演。这种替代策略我们称之为广度优先收集模式，而不是深度优先收集模式。

对于基数大于请求大小或基数未知的字段（例如数字字段或脚本），广度优先是默认模式。可以覆盖默认的启发式，并在请求中直接提供收集模式：

```json
GET /_search
{
    "aggs" : {
        "actors" : {
             "terms" : {
                 "field" : "actors",
                 "size" : 10,
                 "collect_mode" : "breadth_first" // 可选值为breadth_first和depth_first
             },
            "aggs" : {
                "costars" : {
                     "terms" : {
                         "field" : "actors",
                         "size" : 5
                     }
                 }
            }
         }
    }
}
```

使用width_first模式时，会缓存落入最上面存储桶的一组文档以供后续重播，因此这样做会产生与匹配文档数成线性的内存开销。请注意，在使用广度优先设置时，order参数仍然可以用于引用来自子聚合的数据-父聚合知道，在调用任何其他子聚合之前，需要先调用此子聚合。

嵌套聚合（如top_hits）需要在使用width_first collection模式的聚合下访问分数信息，需要在第二次传递时重放查询，但仅限于属于top bucket的文档。

### 执行提示

有不同的机制可以执行Terms聚合：

* 通过直接使用字段值来聚合每个bucket的数据（map）
* 通过使用字段的全局序号并为每个全局序号分配一个bucket（global_ordinals）

Elasticsearch尝试使用合理的默认值，因此这通常不需要配置。

* **global_ordinals** 是keyword字段的默认选项，它使用全局序数动态分配存储桶，因此内存使用量与属于聚合范围的文档值的数量成线性关系。
* **map** 只有极少数文档与查询匹配时才应考虑。否则，基于序数的执行模式要快得多。默认情况下，只有在脚本上运行聚合时才使用map，因为它们没有序号。

```json
GET /_search
{
    "aggs" : {
        "tags" : {
             "terms" : {
                 "field" : "tags",
                 "execution_hint": "map" // 可选值global_ordinals和map
             }
         }
    }
}
```

请注意，如果不适用，Elasticsearch将忽略此执行提示，并且这些提示上没有向后兼容性保证。

### 缺失值

missing参数定义了如何处理缺少值的文档。默认情况下，它们将被忽略，但也可以将它们视为具有值。

```json
GET /_search
{
    "aggs" : {
        "tags" : {
             "terms" : {
                 "field" : "tags",
                 "missing": "其他" 
             }
         }
    }
}
```

### 混合字段类型

当在多个索引上聚合时，聚合字段的类型可能在所有索引中都不相同。有些类型彼此兼容（整数和长或浮点和双精度），但当类型是十进制数和非十进制数的混合时，Terms聚合将把非十进制数提升为十进制数。这可能会导致bucket值的精度损失。

## 2.Rare Terms聚合

一种基于多bucket值源的聚合，它查找位于分布长尾且不频繁的“稀有”项 。从概念上讲，这类似于按计数升序排序的Terms聚合。如Terms聚合文档中所述，按计数升序对Termsagg进行实际排序有无限的错误。相反，您应该使用rare-terms agg

```json
GET /_search
{
    "aggs" : {
        "genres" : {
    		"rare_terms": {
       			"field": "genre",
        		"max_doc_count": 1
   		 	}
		}
    }
}
```

可选参数：

| 参数名        | 描述                                                         | 必填 | 默认值 |
| ------------- | ------------------------------------------------------------ | ---- | ------ |
| field         | 字段名                                                       | 是   |        |
| max_doc_count | 一个Terms应该出现在的最大文档数。                            | 否   | 1      |
| precision     | 内部CuckooFilters的精度。较小的精度导致更好的近似，但内存使用率更高。不能小于0.00001 | 否   | 0.01   |
| include       | 聚合中必须包含的terms                                        | 否   |        |
| exclude       | 聚合中不包含的terms                                          | 否   |        |
| missing       | 缺失值的默认值                                               | 否   |        |

响应结果

```json
{
    ...
    "aggregations" : {
        "genres" : {
            "buckets" : [
                {
                    "key" : "swing",
                    "doc_count" : 1
                }
            ]
        }
    }
}
```

在本例中，我们看到的惟一bucket是“swing”bucket，因为它是一个文档中出现的惟一Terms。如果我们将max_doc_count增加到2，我们将看到更多的bucket：

```json
GET /_search
{
    "aggs" : {
        "genres" : {
            "rare_terms" : {
                "field" : "genre",
                "max_doc_count": 2
            }
        }
    }
}
```

现在显示的“jazz”Terms的doc_计数为2：

```json
{
    ...
    "aggregations" : {
        "genres" : {
            "buckets" : [
                {
                    "key" : "swing",
                    "doc_count" : 1
                },
                {
                    "key" : "jazz",
                    "doc_count" : 2
                }
            ]
        }
    }
}
```

### 最大文档数

max_doc_count参数用于控制Terms可以具有的文档计数的上限。对于agg所拥有的稀有Termsagg没有大小限制。这意味着符合最大文档计数条件的Terms将被返回。聚合以这种方式运行，通过提升影响Terms聚合的问题来避免顺序。

然而，这意味着如果选择不正确，可以返回大量结果。为限制此设置的危险，最大文档计数为100。

### 最大桶限制

稀有Terms聚合比其他聚合更容易触发search.max_bucket软限制，这是由于它的工作方式。当聚合正在收集结果时，将按每个分片计算max_bucket软限制。一个Terms在分片上可能是“罕见的”，但一旦所有分片结果合并在一起，就可能变成“不罕见的”。这意味着单个分片收集的桶数往往比真正罕见的多，因为它们只有自己的本地视图。这个列表最终被修剪为协调节点上正确的、较小的稀有项列表……但是分片可能已经触发了max_buckets软限制并中止了请求。

当在可能包含许多“稀有”项的字段上进行聚合时，可能需要增加“最大存储桶”软限制。或者，您可能需要找到一种方法来筛选结果以返回较少的稀有值（较小的时间跨度、按类别筛选等），或者重新评估您对“稀有”的定义（例如，如果某个东西出现100000次，它真的是“稀有”吗？）

### 文档数是近似的

确定数据集中“稀有”Terms的简单方法是将所有值放在一个映射中，在访问每个文档时递增计数，然后返回最下面的n行。这甚至不能扩展到中等大小的数据集。由于问题的长尾性质意味着，如果不简单地从所有分片收集所有值，就不可能找到“top n”底值，因此仅保留每个分片中的“top n”值的分片方法（alaTerms聚合）失败。

相反，稀有Terms聚合使用不同的近似算法：

* 值在第一次看到时被放置在map中。
* 这个Terms的每次加法都会增加map中的一个计数器
* 当计数器 > max_doc_count的值时，这个term被移除map并放置到[CuckooFilter](https://www.cs.cmu.edu/~dga/papers/cuckoo-conext2014.pdf)中
* 每个term都要查询CuckooFilter 。如果该值在筛选器中，则已知该值已高于阈值并已跳过。

执行后，map是最大文档计数阈值下的“稀有”项映射。然后这个map和布CuckooFilter 与所有其他分片合并。如果存在大于阈值的项（或出现在其他分片的CuckooFilter 中），则该项将从合并列表中移除。值的最终映射将作为“稀有”Terms返回给用户。

CuckooFilter 有返回误报的可能性（当一个值实际上不存在时，它们可以说它存在于它们的集合中）。由于CuckooFilter 用于查看某个Terms是否超过阈值，这意味着布CuckooFilter中的假阳性将错误地说某个值在非阈值时是常见的（从而将其从最终的桶列表中排除）。实际上，这意味着聚合会表现出假负行为，因为过滤器的使用与人们通常认为的近似集成员关系草图“相反”。

关于CuckooFilter更多的信息可以查看[论文：Fan, Bin, et al. "Cuckoo filter: Practically better than bloom.“](https://www.cs.cmu.edu/~dga/papers/cuckoo-conext2014.pdf)



### 精度

虽然内部布CuckooFilter 本质上是近似的，但假阴性率可以通过一个精确的参数来控制。这允许用户使用更多的运行时内存来获得更精确的结果。

默认精度为0.001，最小的（例如，最精确和最大的内存开销）为0.00001。下面是一些图表，这些图表演示了聚合的准确性如何受到精确性和不同项数量的影响。

### 过滤值

同Terms聚合

### 嵌套、稀有词项和子聚合打分

RareTerms聚合必须在广度优先模式下运行，因为它需要在违反文档计数阈值时删减Terms。这一要求意味着RareTerms聚合与某些需要depth\_first的聚合组合不兼容。特别是，对嵌套中的子聚合进行打分会强制整个聚合树以深度优先模式运行。这将引发异常，因为RareTerms无法处理depth_first。

作为一个具体的例子，如果稀有Terms聚合是嵌套聚合的子级，并且稀有Terms的子聚合之一需要文档分数（如top-hits聚合），则会引发异常。

## 3.Range聚合

一种基于多bucket值源的聚合，允许用户定义一组范围，每个范围代表一个bucket。在聚合过程中，将根据每个bucket范围和“bucket”相关/匹配文档检查从每个文档提取的值。请注意，此聚合包括每个范围的from值，而不包括to值。

```json
GET /_search
{
    "aggs" : {
        "price_ranges" : {
            "range" : {
                "field" : "price",
                "ranges" : [
                    { "to" : 100.0 },
                    { "from" : 100.0, "to" : 200.0 },
                    { "from" : 200.0 }
                ]
            }
        }
    }
}
```

返回结果

```json
{
    ...
    "aggregations": {
        "price_ranges" : {
            "buckets": [
                {
                    "key": "*-100.0",
                    "to": 100.0,
                    "doc_count": 2
                },
                {
                    "key": "100.0-200.0",
                    "from": 100.0,
                    "to": 200.0,
                    "doc_count": 2
                },
                {
                    "key": "200.0-*",
                    "from": 200.0,
                    "doc_count": 3
                }
            ]
        }
    }
}
```

### 返回数据key结构

将keyed标志设置为true将为每个bucket关联一个唯一的字符串键，并将范围作为散列而不是数组返回：

```json
GET /_search
{
    "aggs" : {
        "price_ranges" : {
            "range" : {
                "field" : "price",
                "keyed" : true,
                "ranges" : [
                    { "to" : 100 },
                    { "from" : 100, "to" : 200 },
                    { "from" : 200 }
                ]
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
        "price_ranges" : {
            "buckets": {
                "*-100.0": {
                    "to": 100.0,
                    "doc_count": 2
                },
                "100.0-200.0": {
                    "from": 100.0,
                    "to": 200.0,
                    "doc_count": 2
                },
                "200.0-*": {
                    "from": 200.0,
                    "doc_count": 3
                }
            }
        }
    }
}
```

还可以为每个范围自定义key值：

```json
GET /_search
{
    "aggs" : {
        "price_ranges" : {
            "range" : {
                "field" : "price",
                "keyed" : true,
                "ranges" : [
                    { "key" : "cheap", "to" : 100 },
                    { "key" : "average", "from" : 100, "to" : 200 },
                    { "key" : "expensive", "from" : 200 }
                ]
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
        "price_ranges" : {
            "buckets": {
                "cheap": {
                    "to": 100.0,
                    "doc_count": 2
                },
                "average": {
                    "from": 100.0,
                    "to": 200.0,
                    "doc_count": 2
                },
                "expensive": {
                    "from": 200.0,
                    "doc_count": 3
                }
            }
        }
    }
}
```

### 脚本

范围聚合接受脚本参数。此参数允许定义将在聚合执行期间执行的内联脚本。

```json
GET /_search
{
    "aggs" : {
        "price_ranges" : {
            "range" : {
                "script" : {
                    "lang": "painless",
                    "source": "doc['price'].value"
                },
                "ranges" : [
                    { "to" : 100 },
                    { "from" : 100, "to" : 200 },
                    { "from" : 200 }
                ]
            }
        }
    }
}
```

使用预定义脚本

```json
POST /_scripts/convert_currency
{
  "script": {
    "lang": "painless",
    "source": "doc[params.field].value * params.conversion_rate"
  }
}
```

```json
GET /_search
{
    "aggs" : {
        "price_ranges" : {
            "range" : {
                "script" : {
                    "id": "convert_currency", // 上面请求定义的脚本
                    "params": { 
                        "field": "price",
                        "conversion_rate": 0.835526591
                    }
                },
                "ranges" : [
                    { "from" : 0, "to" : 100 },
                    { "from" : 100 }
                ]
            }
        }
    }
}
```

### 值脚本

假设产品价格是美元，但我们希望得到欧元的价格区间。我们可以使用值脚本在聚合之前转换价格（假设转换率为0.8）

```json
GET /sales/_search
{
    "aggs" : {
        "price_ranges" : {
            "range" : {
                "field" : "price",
                "script" : {
                    "source": "_value * params.conversion_rate",
                    "params" : {
                        "conversion_rate" : 0.8
                    }
                },
                "ranges" : [
                    { "to" : 35 },
                    { "from" : 35, "to" : 70 },
                    { "from" : 70 }
                ]
            }
        }
    }
}
```

### 下级聚合

下面的示例不仅将文档“bucket”到不同的bucket，还计算每个价格范围内的价格统计信息

```json
GET /_search
{
    "aggs" : {
        "price_ranges" : {
            "range" : {
                "field" : "price",
                "ranges" : [
                    { "to" : 100 },
                    { "from" : 100, "to" : 200 },
                    { "from" : 200 }
                ]
            },
            "aggs" : {
                "price_stats" : {
                    "stats" : { "field" : "price" }
                }
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
    "price_ranges": {
      "buckets": [
        {
          "key": "*-100.0",
          "to": 100.0,
          "doc_count": 2,
          "price_stats": {
            "count": 2,
            "min": 10.0,
            "max": 50.0,
            "avg": 30.0,
            "sum": 60.0
          }
        },
        {
          "key": "100.0-200.0",
          "from": 100.0,
          "to": 200.0,
          "doc_count": 2,
          "price_stats": {
            "count": 2,
            "min": 150.0,
            "max": 175.0,
            "avg": 162.5,
            "sum": 325.0
          }
        },
        {
          "key": "200.0-*",
          "from": 200.0,
          "doc_count": 3,
          "price_stats": {
            "count": 3,
            "min": 200.0,
            "max": 200.0,
            "avg": 200.0,
            "sum": 600.0
          }
        }
      ]
    }
  }
}
```

如果子聚合也基于与范围聚合相同的值源（如上面示例中的stats聚合），则可以省略它的值源定义。以下将返回与上述相同的响应：

```json
GET /_search
{
    "aggs" : {
        "price_ranges" : {
            "range" : {
                "field" : "price",
                "ranges" : [
                    { "to" : 100 },
                    { "from" : 100, "to" : 200 },
                    { "from" : 200 }
                ]
            },
            "aggs" : {
                "price_stats" : {
                    "stats" : {} // 无需再填写值源字段定义
                }
            }
        }
    }
}
```



## 4.Date Range聚合

专用于日期值的范围聚合。此聚合与普通范围聚合的主要区别在于，from和to值可以用日期数学表达式表示，还可以指定返回from和to响应字段的日期格式。请注意，此聚合包括每个范围的from值，而不包括to值。

```json
POST /sales/_search?size=0
{
    "aggs": {
        "range": {
            "date_range": {
                "field": "date",
                "format": "MM-yyyy",
                "ranges": [
                    { "to": "now-10M/M" }, // < 现在减去10个月，四舍五入到月初
                    { "from": "now-10M/M" } // >= 现在减去10个月
                ]
            }
        }
    }
}
```

在上面的例子中，我们创建了两个range bucket，第一个将“bucket”10个月以前的所有文档，第二个将“bucket”10个月以后的所有文档。

结果

```json
{
    ...
    "aggregations": {
        "range": {
            "buckets": [
                {
                    "to": 1.4436576E12,
                    "to_as_string": "10-2015",
                    "doc_count": 7,
                    "key": "*-10-2015"
                },
                {
                    "from": 1.4436576E12,
                    "from_as_string": "10-2015",
                    "doc_count": 0,
                    "key": "10-2015-*"
                }
            ]
        }
    }
}
```

### 缺失值

缺少的参数定义了如何处理缺少值的文档。默认情况下，它们将被忽略，但也可以将它们视为具有值。这是通过添加一组fieldname:value映射来完成的，以指定每个字段的默认值。

```json
POST /sales/_search?size=0
{
   "aggs": {
       "range": {
           "date_range": {
               "field": "date",
               "missing": "1976/11/30", // 默认值
               "ranges": [
                  {
                    "key": "Older",
                    "to": "2016/02/01"
                  }, 
                  {
                    "key": "Newer",
                    "from": "2016/02/01",
                    "to" : "now/d"
                  }
              ]
          }
      }
   }
}
```

### 日期模式

### 时区

通过指定时区参数，可以将日期从其他时区转换为UTC。

时区可以指定为ISO 8601 UTC偏移量（例如+01:00或-08:00），也可以指定为TZ数据库中的时区ID之一。

时区参数也应用于日期数学表达式中的舍入。例如，要在CET时区中循环到一天的开始，可以执行以下操作：

```json
POST /sales/_search?size=0
{
   "aggs": {
       "range": {
           "date_range": {
               "field": "date",
               "time_zone": "CET",
               "ranges": [
                  { "to": "2016/02/01" }, // 会被转化为2016-02-01T00:00:00.000+01:00
                  { "from": "2016/02/01", "to" : "now/d" }, 
                  { "from": "now/d" }
              ]
          }
      }
   }
}
```



## 5.Histogram聚合

一种基于源代码的多bucket值聚合，可应用于从文档中提取的数值。它动态地在值上构建固定大小（也称为间隔）的存储桶。例如，如果文档有一个包含价格（数字）的字段，我们可以将此聚合配置为动态构建间隔为5的存储桶（如果价格为5美元）。执行聚合时，将对每个文档的价格字段进行求值，并将其舍入到最接近的存储桶-例如，如果价格为32，存储桶大小为5，则舍入将产生30，因此文档将“落入”与键30关联的存储桶中。为了使其更正式，下面是使用的舍入函数：

```
bucket_key = Math.floor((value - offset) / interval) * interval + offset
```

interval必须是正十进制，而offset必须是[0，interval]中的十进制（大于或等于0且小于interval的十进制）。

下面的代码片段“bucket”根据产品的价格以50为间隔排列：

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "prices" : {
            "histogram" : {
                "field" : "price",
                "interval" : 50
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
        "prices" : {
            "buckets": [
                {
                    "key": 0.0,
                    "doc_count": 1
                },
                {
                    "key": 50.0,
                    "doc_count": 1
                },
                {
                    "key": 100.0,
                    "doc_count": 0
                },
                {
                    "key": 150.0,
                    "doc_count": 2
                },
                {
                    "key": 200.0,
                    "doc_count": 3
                }
            ]
        }
    }
}
```

### 最小文档数

上述回复显示，没有任何文件的价格在[100,150]的范围内。默认情况下，响应将用空桶填充直方图中的空白。由于min_doc_count设置，有可能更改和请求具有更高最小计数的存储桶：

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "prices" : {
            "histogram" : {
                "field" : "price",
                "interval" : 50,
                "min_doc_count" : 1
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
        "prices" : {
            "buckets": [
                {
                    "key": 0.0,
                    "doc_count": 1
                },
                {
                    "key": 50.0,
                    "doc_count": 1
                },
                {
                    "key": 150.0,
                    "doc_count": 2
                },
                {
                    "key": 200.0,
                    "doc_count": 3
                }
            ]
        }
    }
}
```

默认情况下，直方图返回数据本身范围内的所有存储桶，即最小值（带直方图的）文档将确定最小存储桶（带最小键的存储桶），最大值文档将确定最大存储桶（带最大键的存储桶）。通常，当请求空桶时，这会导致混淆，特别是当数据也被过滤时。

我们来看个例子：

假设您正在筛选您的请求，以获取值在0到500之间的所有文档，此外，您还希望使用间隔为50的直方图对每个价格的数据进行切片。您还可以指定“min_doc_count”：0，因为您希望获得所有存储桶，甚至是空的存储桶。如果碰巧所有产品（文档）的价格都高于100，那么您将得到的第一个bucket将是以100为键的bucket。这是令人困惑的，因为很多时候，你也希望得到0-100之间的桶。

现在通过使用extended_bounds，您可以“强制”直方图聚合开始根据特定的最小值构建存储桶，还可以继续构建最大值的存储桶（即使不再有文档）。只有当min_doc_count为0时，使用扩展的_界限才有意义（如果min_doc_count大于0，则永远不会返回空桶）。

extended_bounds不是筛选存储桶。也就是说，如果extended_bounds.min高于从文档中提取的值，则文档仍将指示第一个bucket是什么（extended_bounds.max和最后一个bucket也是如此）。对于过滤bucket，应该使用适当的from/to设置将直方图聚合嵌套在范围筛选器聚合下。

```json
POST /sales/_search?size=0
{
    "query" : {
        "constant_score" : { "filter": { "range" : { "price" : { "to" : "500" } } } }
    },
    "aggs" : {
        "prices" : {
            "histogram" : {
                "field" : "price",
                "interval" : 50,
                "extended_bounds" : {
                    "min" : 0,
                    "max" : 500
                }
            }
        }
    }
}
```

### 顺序

默认情况下，返回的存储桶按key序排序，但可以使用“顺序”设置控制顺序行为。支持与Terms聚合相同的排序功能。

### 偏移量

默认情况下，bucket键从0开始，然后以均匀间隔的间隔步数继续，例如，如果间隔为10，则前三个bucket（假设其中有数据）将为[0，10），[10，20），[20，30）。可以使用偏移选项移动bucket边界。

这可以用一个例子来说明。如果有10个文档的值在5到14之间，则使用间隔10将生成两个bucket，每个bucket包含5个文档。如果使用额外的偏移量5，则只有一个bucket[5，15]包含所有10个文档。

### 返回格式

默认情况下，bucket作为有序数组返回。也可以将响应请求为散列，而不是由bucket键进行键控：

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "prices" : {
            "histogram" : {
                "field" : "price",
                "interval" : 50,
                "keyed" : true
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
        "prices": {
            "buckets": {
                "0.0": {
                    "key": 0.0,
                    "doc_count": 1
                },
                "50.0": {
                    "key": 50.0,
                    "doc_count": 1
                },
                "100.0": {
                    "key": 100.0,
                    "doc_count": 0
                },
                "150.0": {
                    "key": 150.0,
                    "doc_count": 2
                },
                "200.0": {
                    "key": 200.0,
                    "doc_count": 3
                }
            }
        }
    }
}
```

### 缺失值

missing参数定义了如何处理缺少值的文档。默认情况下，它们将被忽略，但也可以将它们视为具有值。

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "quantity" : {
             "histogram" : {
                 "field" : "quantity",
                 "interval": 10,
                 "missing": 0 
             }
         }
    }
}
```



## 6.Date Histogram聚合

这种多bucket聚合与普通直方图类似，但只能与日期值一起使用。因为在Elasticsearch中，日期在内部表示为long，所以也可以对日期使用标准直方图，但不一定那么准确。这两个api的主要区别在于，这里可以使用日期/时间表达式指定间隔。基于时间的数据需要特殊的支持，因为基于时间的间隔并不总是固定的长度。

### 日历间隔和固定间隔

在配置日期直方图聚合时，可以用两种方式指定间隔：日历感知时间间隔和固定时间间隔。

* 日历感知间隔可以理解日光节约会改变特定日期的长度，月份有不同的天数，闰秒可以固定在特定年份上。
* 相比之下，固定间隔总是国际单位的倍数，并且不会根据日历上下文而更改。

> **合并的interval参数已经废弃**
>
> 7.2版本以前，日历和固定时间间隔都配置在一个时间间隔字段中，这导致语义混乱。指定1d将被假定为日历感知时间，而2d将被解释为固定时间。要获得固定时间的“一天”，用户需要指定下一个较小的单位（在本例中为24小时）。
>
> 这种组合的行为对于用户来说通常是未知的，即使知道这种行为，也很难使用和混淆。
>
> 此行为已被弃用，取而代之的是两个新的显式字段：calendar_interval和fixed_interval。
>
> 通过预先在日历和间隔之间强制进行选择，用户可以立即清楚地看到间隔的语义，并且不存在任何歧义。以后将删除旧的interval字段。

### 日历间隔

使用Calendar_interval参数配置可识别日历的间隔。日历间隔只能用单位的“单数”表示（1d、1M等）。不支持倍数（如2d），将引发异常。

日历间隔的可接受单位为：

* **minute(m,1m)**
* **hour (h, 1h)**
* **day (**`d`**,** `1d`**)**
* **week (**`w`**,** `1w`**)**
* **month (**`M`**,** `1M`**)**
* **quarter (**`q`**,** `1q`**)**
* **year (**`y`**,** `1y`**)**

例子

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "sales_over_time" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            }
        }
    }
}
```

### 固定间隔

与日历相关的时间间隔不同，固定时间间隔是固定数量的国际单位制，无论它们在日历上的位置如何，都不会偏离。每秒总是由1000毫秒组成。这允许以任何支持的单位的倍数指定固定间隔。

但是，这意味着固定的时间间隔不能表示其他单位，例如月，因为一个月的持续时间不是固定的数量。尝试指定日历间隔（如月或季度）将引发异常。

固定间隔的可接受单位为：

* **milliseconds (ms)**
* **seconds (s)**
* **minutes (m)**
* **hours (h)**
* **days (d)**

例子

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "sales_over_time" : {
            "date_histogram" : {
                "field" : "date",
                "fixed_interval" : "30d"
            }
        }
    }
}
```

### 注意

在所有情况下，当指定的结束时间不存在时，实际的结束时间是指定结束后最接近的可用时间。

广泛分布的应用程序还必须考虑一些不确定的因素，例如在12:01上午开始和停止夏令时的国家，因此在一年一次的星期六结束后，每个星期天结束一分钟，然后是另外一个59分钟的星期六，以及决定跨越国际日期线的国家。这样的情况会使不规则的时区偏移看起来很容易。

一如既往，严格的测试，特别是围绕时间变化事件的测试，将确保您的时间间隔规范是您想要的。

为了避免意外的结果，所有连接的服务器和客户端必须同步到可靠的网络时间服务。

不支持分数时间值，但您可以通过切换到另一个时间单位（例如，1.5h可以改为90m）来解决此问题。

您还可以使用时间单位分析支持的缩写来指定时间值。



### 返回键

在内部，日期表示为64位数字，表示自纪元（1970年1月1日午夜UTC）以来的时间戳（以毫秒为单位）。这些时间戳作为存储桶的key返回。key_as_string与使用格式参数规范转换为格式化日期字符串的时间戳相同：

> 如果不指定格式，则使用字段映射中指定的第一个日期格式。

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "sales_over_time" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "1M",
                "format" : "yyyy-MM-dd" 
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
        "sales_over_time": {
            "buckets": [
                {
                    "key_as_string": "2015-01-01",
                    "key": 1420070400000,
                    "doc_count": 3
                },
                {
                    "key_as_string": "2015-02-01",
                    "key": 1422748800000,
                    "doc_count": 2
                },
                {
                    "key_as_string": "2015-03-01",
                    "key": 1425168000000,
                    "doc_count": 2
                }
            ]
        }
    }
}
```

### 时区

日期时间以UTC格式存储在Elasticsearch中。默认情况下，所有的bucketing和rounding也是在UTC中完成的。使用时区参数指示bucketing应使用不同的时区。

您可以将时区指定为ISO 8601 UTC偏移量（例如+01:00或-08:00）或IANA时区数据库中指定的时区ID，例如“美国/洛杉矶”。

考虑下面的例子

```json
PUT my_index/_doc/1?refresh
{
  "date": "2015-10-01T00:30:00Z"
}

PUT my_index/_doc/2?refresh
{
  "date": "2015-10-01T01:30:00Z"
}

GET my_index/_search?size=0
{
  "aggs": {
    "by_day": {
      "date_histogram": {
        "field":     "date",
        "calendar_interval":  "day"
      }
    }
  }
}
```

如果不指定时区，则使用UTC。这将导致这两个文档放在同一个日期段中，该日期段从2015年10月1日UTC午夜开始：

```json
{
  ...
  "aggregations": {
    "by_day": {
      "buckets": [
        {
          "key_as_string": "2015-10-01T00:00:00.000Z",
          "key":           1443657600000,
          "doc_count":     2
        }
      ]
    }
  }
}
```

如果将时区指定为-01:00，则该时区中的午夜比UTC午夜早一小时：

```json
GET my_index/_search?size=0
{
  "aggs": {
    "by_day": {
      "date_histogram": {
        "field":     "date",
        "calendar_interval":  "day",
        "time_zone": "-01:00"
      }
    }
  }
}
```

现在，第一份文件在2015年9月30日落入桶中，而第二份文件在2015年10月1日落入桶中：

```json
{
  ...
  "aggregations": {
    "by_day": {
      "buckets": [
        {
          "key_as_string": "2015-09-30T00:00:00.000-01:00", 
          "key": 1443574800000,
          "doc_count": 1
        },
        {
          "key_as_string": "2015-10-01T00:00:00.000-01:00", 
          "key": 1443661200000,
          "doc_count": 1
        }
      ]
    }
  }
}
```

### 偏移量

使用offset参数将每个bucket的起始值更改为指定的正（＋）或负（－）偏移持续时间，例如1h表示一小时，1d表示一天。有关更多可能的持续时间选项，请参见时间单位。

例如，当使用一天的间隔时，每个bucket从午夜到午夜。将offset参数设置为+6h会将每个bucket的运行时间从早上6点到早上6点：

```json
PUT my_index/_doc/1?refresh
{
  "date": "2015-10-01T05:30:00Z"
}

PUT my_index/_doc/2?refresh
{
  "date": "2015-10-01T06:30:00Z"
}

GET my_index/_search?size=0
{
  "aggs": {
    "by_day": {
      "date_histogram": {
        "field":     "date",
        "calendar_interval":  "day",
        "offset":    "+6h"
      }
    }
  }
}
```

上面的请求不是从午夜开始的单个存储桶，而是从早上6点开始将文档分组为存储桶：

```json
{
  ...
  "aggregations": {
    "by_day": {
      "buckets": [
        {
          "key_as_string": "2015-09-30T06:00:00.000Z",
          "key": 1443592800000,
          "doc_count": 1
        },
        {
          "key_as_string": "2015-10-01T06:00:00.000Z",
          "key": 1443679200000,
          "doc_count": 1
        }
      ]
    }
  }
}
```

### 返回键

将keyed标志设置为true会将唯一的字符串键与每个bucket关联起来，并将范围作为散列而不是数组返回：

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "sales_over_time" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "1M",
                "format" : "yyyy-MM-dd",
                "keyed": true
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
        "sales_over_time": {
            "buckets": {
                "2015-01-01": {
                    "key_as_string": "2015-01-01",
                    "key": 1420070400000,
                    "doc_count": 3
                },
                "2015-02-01": {
                    "key_as_string": "2015-02-01",
                    "key": 1422748800000,
                    "doc_count": 2
                },
                "2015-03-01": {
                    "key_as_string": "2015-03-01",
                    "key": 1425168000000,
                    "doc_count": 2
                }
            }
        }
    }
}
```



## 7.Filter聚合

定义当前文档集上下文中与指定filter器匹配的所有文档的单个存储桶。通常，这将用于将当前聚合上下文缩小到一组特定的文档。

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "t_shirts" : {
            "filter" : { "term": { "type": "t-shirt" } },
            "aggs" : {
                "avg_price" : { "avg" : { "field" : "price" } }
            }
        }
    }
}
```

结果

```json
{
    ...
    "aggregations" : {
        "t_shirts" : {
            "doc_count" : 3,
            "avg_price" : { "value" : 128.33333333333334 }
        }
    }
}
```



## 8.Filters聚合

定义一个多bucket聚合，其中每个bucket与一个过滤器关联。每个bucket将收集与其关联筛选器匹配的所有文档。

```json
PUT /logs/_bulk?refresh
{ "index" : { "_id" : 1 } }
{ "body" : "warning: page could not be rendered" }
{ "index" : { "_id" : 2 } }
{ "body" : "authentication error" }
{ "index" : { "_id" : 3 } }
{ "body" : "warning: connection timed out" }

GET logs/_search
{
  "size": 0,
  "aggs" : {
    "messages" : {
      "filters" : {
        "filters" : {
          "errors" :   { "match" : { "body" : "error"   }},
          "warnings" : { "match" : { "body" : "warning" }}
        }
      }
    }
  }
}
```

在上面的例子中，我们分析日志消息。聚合将构建两个日志消息集合（bucket）——一个用于包含错误的所有日志消息，另一个用于包含警告的所有日志消息。

```json
{
  "took": 9,
  "timed_out": false,
  "_shards": ...,
  "hits": ...,
  "aggregations": {
    "messages": {
      "buckets": {
        "errors": {
          "doc_count": 1
        },
        "warnings": {
          "doc_count": 2
        }
      }
    }
  }
}
```

### 匿名过滤器

filters字段也可以作为过滤器数组提供，如下所示：

```json
GET logs/_search
{
  "size": 0,
  "aggs" : {
    "messages" : {
      "filters" : {
        "filters" : [
          { "match" : { "body" : "error"   }},
          { "match" : { "body" : "warning" }}
        ]
      }
    }
  }
}
```

结果

```json
{
  "took": 4,
  "timed_out": false,
  "_shards": ...,
  "hits": ...,
  "aggregations": {
    "messages": {
      "buckets": [
        {
          "doc_count": 1
        },
        {
          "doc_count": 2
        }
      ]
    }
  }
}
```

### 其他桶

other_bucket参数可以设置为向响应添加一个bucket，该响应将包含与任何给定筛选器都不匹配的所有文档。此参数的值可以如下所示：

* false 不计算其他文档
* true 返回其他文档桶（默认名\_other\_）

```json
PUT logs/_doc/4?refresh
{
  "body": "info: user Bob logged out"
}

GET logs/_search
{
  "size": 0,
  "aggs" : {
    "messages" : {
      "filters" : {
        "other_bucket_key": "other_messages", // 其他桶命名指定
        "filters" : {
          "errors" :   { "match" : { "body" : "error"   }},
          "warnings" : { "match" : { "body" : "warning" }}
        }
      }
    }
  }
}
```

结果

```json
{
  "took": 3,
  "timed_out": false,
  "_shards": ...,
  "hits": ...,
  "aggregations": {
    "messages": {
      "buckets": {
        "errors": {
          "doc_count": 1
        },
        "warnings": {
          "doc_count": 2
        },
        "other_messages": {
          "doc_count": 1
        }
      }
    }
  }
}
```



## 9.父聚合

一种特殊的单bucket聚合，它选择具有指定类型的父文档，如join字段中定义的。

这个聚合有一个单选项 type：子类型应该选择的类型

例如，假设我们有一个问答索引。应答类型在映射中具有以下联接字段：

```json
PUT parent_example
{
  "mappings": {
     "properties": {
       "join": {
         "type": "join",
         "relations": {
           "question": "answer"
         }
       }
     }
  }
}
```

question文档包含标记字段，answer文档包含所有者字段。通过父级聚合，即使两个字段存在于两种不同的文档中，所有者bucket也可以映射到单个请求中的标记bucket。

question文档的一个例子

```json
PUT parent_example/_doc/1
{
  "join": {
    "name": "question"
  },
  "body": "<p>I have Windows 2003 server and i bought a new Windows 2008 server...",
  "title": "Whats the best way to file transfer my site from server to a newer one?",
  "tags": [
    "windows-server-2003",
    "windows-server-2008",
    "file-transfer"
  ]
}
```

answer文档的一个例子

```json
PUT parent_example/_doc/2?routing=1
{
  "join": {
    "name": "answer",
    "parent": "1"
  },
  "owner": {
    "location": "Norfolk, United Kingdom",
    "display_name": "Sam",
    "id": 48
  },
  "body": "<p>Unfortunately you're pretty much limited to FTP...",
  "creation_date": "2009-05-04T13:45:37.030"
}

PUT parent_example/_doc/3?routing=1&refresh
{
  "join": {
    "name": "answer",
    "parent": "1"
  },
  "owner": {
    "location": "Norfolk, United Kingdom",
    "display_name": "Troll",
    "id": 49
  },
  "body": "<p>Use Linux...",
  "creation_date": "2009-05-05T13:45:37.030"
}
```

可以构建以下将两者连接在一起的请求：

```json
POST parent_example/_search?size=0
{
  "aggs": {
    "top-names": {
      "terms": {
        "field": "owner.display_name.keyword",
        "size": 10
      },
      "aggs": {
        "to-questions": {
          "parent": {
            "type" : "answer" 
          },
          "aggs": {
            "top-tags": {
              "terms": {
                "field": "tags.keyword",
                "size": 10
              }
            }
          }
        }
      }
    }
  }
}
```

上面的示例返回最上面的答案所有者和每个所有者最上面的问题标签。

```json
{
  "took": 9,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total" : {
      "value": 3,
      "relation": "eq"
    },
    "max_score": null,
    "hits": []
  },
  "aggregations": {
    "top-names": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "Sam",
          "doc_count": 1, 
          "to-questions": {
            "doc_count": 1, 
            "top-tags": {
              "doc_count_error_upper_bound": 0,
              "sum_other_doc_count": 0,
              "buckets": [
                {
                  "key": "file-transfer",
                  "doc_count": 1
                },
                {
                  "key": "windows-server-2003",
                  "doc_count": 1
                },
                {
                  "key": "windows-server-2008",
                  "doc_count": 1
                }
              ]
            }
          }
        },
        {
          "key": "Troll",
          "doc_count": 1,
          "to-questions": {
            "doc_count": 1,
            "top-tags": {
              "doc_count_error_upper_bound": 0,
              "sum_other_doc_count": 0,
              "buckets": [
                {
                  "key": "file-transfer",
                  "doc_count": 1
                },
                {
                  "key": "windows-server-2003",
                  "doc_count": 1
                },
                {
                  "key": "windows-server-2008",
                  "doc_count": 1
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



## 10.子聚合

一种特殊的单bucket聚合，它选择具有指定类型的子文档，如join字段中定义的。

这个聚合有一个单选项 type：子类型应该选择的类型

例如，假设我们有一个问答索引。应答类型在映射中具有以下联接字段：

```json
PUT parent_example
{
  "mappings": {
     "properties": {
       "join": {
         "type": "join",
         "relations": {
           "question": "answer"
         }
       }
     }
  }
}
```

question文档包含标记字段，answer文档包含所有者字段。通过父级聚合，即使两个字段存在于两种不同的文档中，所有者bucket也可以映射到单个请求中的标记bucket。

question文档的一个例子

```json
PUT parent_example/_doc/1
{
  "join": {
    "name": "question"
  },
  "body": "<p>I have Windows 2003 server and i bought a new Windows 2008 server...",
  "title": "Whats the best way to file transfer my site from server to a newer one?",
  "tags": [
    "windows-server-2003",
    "windows-server-2008",
    "file-transfer"
  ]
}
```

answer文档的一个例子

```json
PUT parent_example/_doc/2?routing=1
{
  "join": {
    "name": "answer",
    "parent": "1"
  },
  "owner": {
    "location": "Norfolk, United Kingdom",
    "display_name": "Sam",
    "id": 48
  },
  "body": "<p>Unfortunately you're pretty much limited to FTP...",
  "creation_date": "2009-05-04T13:45:37.030"
}

PUT parent_example/_doc/3?routing=1&refresh
{
  "join": {
    "name": "answer",
    "parent": "1"
  },
  "owner": {
    "location": "Norfolk, United Kingdom",
    "display_name": "Troll",
    "id": 49
  },
  "body": "<p>Use Linux...",
  "creation_date": "2009-05-05T13:45:37.030"
}
```

可以构建以下将两者连接在一起的请求：

```json
POST child_example/_search?size=0
{
  "aggs": {
    "top-tags": {
      "terms": {
        "field": "tags.keyword",
        "size": 10
      },
      "aggs": {
        "to-answers": {
          "children": {
            "type" : "answer" 
          },
          "aggs": {
            "top-names": {
              "terms": {
                "field": "owner.display_name.keyword",
                "size": 10
              }
            }
          }
        }
      }
    }
  }
}
```

上面的示例返回顶级问题标记，每个标记返回顶级答案所有者。

```json
{
  "took": 25,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped" : 0,
    "failed": 0
  },
  "hits": {
    "total" : {
      "value": 3,
      "relation": "eq"
    },
    "max_score": null,
    "hits": []
  },
  "aggregations": {
    "top-tags": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "file-transfer",
          "doc_count": 1, 
          "to-answers": {
            "doc_count": 2, 
            "top-names": {
              "doc_count_error_upper_bound": 0,
              "sum_other_doc_count": 0,
              "buckets": [
                {
                  "key": "Sam",
                  "doc_count": 1
                },
                {
                  "key": "Troll",
                  "doc_count": 1
                }
              ]
            }
          }
        },
        {
          "key": "windows-server-2003",
          "doc_count": 1, 
          "to-answers": {
            "doc_count": 2, 
            "top-names": {
              "doc_count_error_upper_bound": 0,
              "sum_other_doc_count": 0,
              "buckets": [
                {
                  "key": "Sam",
                  "doc_count": 1
                },
                {
                  "key": "Troll",
                  "doc_count": 1
                }
              ]
            }
          }
        },
        {
          "key": "windows-server-2008",
          "doc_count": 1, 
          "to-answers": {
            "doc_count": 2, 
            "top-names": {
              "doc_count_error_upper_bound": 0,
              "sum_other_doc_count": 0,
              "buckets": [
                {
                  "key": "Sam",
                  "doc_count": 1
                },
                {
                  "key": "Troll",
                  "doc_count": 1
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

## 11.composite复合聚合

从不同字段来源创建复合存储桶的多存储桶聚合。

与其他多bucket聚合不同，复合聚合可用于有效地对来自多级聚合的所有bucket进行分页。此聚合提供了一种方法，可以流式处理特定聚合的所有存储桶，类似于scroll对文档所做的操作。

组合bucket是根据为每个文档提取/创建的值的组合构建的，每个组合都被视为一个组合bucket。

例如有下面的文档：

```json
{
    "keyword": ["foo", "bar"],
    "number": [23, 65, 76]
}
```

当关键字和数字用作聚合的值源时，创建以下组合存储桶：

```json
{ "keyword": "foo", "number": 23 }
{ "keyword": "foo", "number": 65 }
{ "keyword": "foo", "number": 76 }
{ "keyword": "bar", "number": 23 }
{ "keyword": "bar", "number": 65 }
{ "keyword": "bar", "number": 76 }
```

### 值来源

sources参数控制应用于构建复合存储桶的源。定义源的顺序很重要，因为它还控制键的返回顺序。

> 提供给每个源的名称必须是唯一的。

有三种不同类型的值源：

* Terms

Terms值源相当于一个简单的Terms聚合。这些值是从字段或脚本中提取的，与Terms聚合完全相同。

```json
GET /_search
{
    "aggs" : {
        "my_buckets": {
            "composite" : {
                "sources" : [
                    { "product": { "terms" : { "field": "product" } } }
                ]
            }
        }
     }
}
```

与Terms聚合一样，还可以使用脚本为复合存储桶创建值：

```json
GET /_search
{
    "aggs" : {
        "my_buckets": {
            "composite" : {
                "sources" : [
                    {
                        "product": {
                            "terms" : {
                                "script" : {
                                    "source": "doc['product'].value",
                                    "lang": "painless"
                                }
                            }
                        }
                    }
                ]
            }
        }
    }
}
```

* Histogram

直方图值源可以应用于数值，以在数值上建立固定的大小间隔。interval参数定义如何转换数值。例如，设置为5的间隔将把任何数值转换为其最近的间隔，101的值将转换为100，这是100到105之间间隔的关键。

```json
GET /_search
{
    "aggs" : {
        "my_buckets": {
            "composite" : {
                "sources" : [
                    { "histo": { "histogram" : { "field": "price", "interval": 5 } } }
                ]
            }
        }
    }
}
```

* Date Histogram

日期直方图与直方图值源类似，只是时间间隔由日期/时间表达式指定：

```json
GET /_search
{
    "aggs" : {
        "my_buckets": {
            "composite" : {
                "sources" : [
                    { "date": { "date_histogram" : { "field": "timestamp", "calendar_interval": "1d" } } }
                ]
            }
        }
    }
}
```

### 混合不同的值来源

sources参数接受一个值数组source。可以混合不同的值源来创建复合bucket。例如：

```json
GET /_search
{
    "aggs" : {
        "my_buckets": {
            "composite" : {
                "sources" : [
                    { "date": { "date_histogram": { "field": "timestamp", "calendar_interval": "1d" } } },
                    { "product": { "terms": {"field": "product" } } }
                ]
            }
        }
    }
}
```

这将从两个值源（日期直方图和Terms）创建的值创建组合存储桶。每个bucket由两个值组成，一个用于聚合中定义的每个值源。允许任何类型的组合，并且数组中的顺序保留在复合存储桶中。

```json
GET /_search
{
    "aggs" : {
        "my_buckets": {
            "composite" : {
                "sources" : [
                    { "shop": { "terms": {"field": "shop" } } },
                    { "product": { "terms": { "field": "product" } } },
                    { "date": { "date_histogram": { "field": "timestamp", "calendar_interval": "1d" } } }
                ]
            }
        }
    }
}
```

### 顺序order

默认情况下，组合存储桶按其自然顺序排序。值按其值的升序排序。当请求多个值源时，对每个值源进行排序，将组合桶的第一个值与另一个组合桶的第一个值进行比较，如果它们等于组合桶中的下一个值，则用于断开连接。这意味着组合bucket[foo，100]被认为小于[foobar，0]，因为foo被认为小于foobar。通过在值源定义中将order直接设置为asc（默认值）或desc（降序），可以定义每个值源的排序方向。例如：

```json
GET /_search
{
    "aggs" : {
        "my_buckets": {
            "composite" : {
                "sources" : [
                    { "date": { "date_histogram": { "field": "timestamp", "calendar_interval": "1d", "order": "desc" } } },
                    { "product": { "terms": {"field": "product", "order": "asc" } } }
                ]
            }
        }
    }
}
```

当比较来自日期直方图源的值时，将按降序排列组合存储桶；当比较来自Terms源的值时，将按升序排列组合存储桶。

### 缺失桶missing_bucket

默认情况下，忽略没有给定源值的文档。通过将missing_bucket设置为true（默认为false），可以将它们包括在响应中：

```json
GET /_search
{
    "aggs" : {
        "my_buckets": {
            "composite" : {
                "sources" : [
                    { "product_name": { "terms" : { "field": "product", "missing_bucket": true } } }
                ]
            }
        }
     }
}
```

在上面的示例中，源产品名将为没有字段产品值的文档发出显式空值。源中指定的顺序指示空值应排在第一位（升序，asc）还是最后一位（降序，desc）。

### 容量size

可以设置size参数来定义应该返回多少个组合存储桶。每个复合bucket被视为一个bucket，因此将size设置为10将返回从values源创建的前10个复合bucket。响应包含数组中每个组合存储桶的值，该数组包含从每个值源提取的值。

### after

如果组合存储桶的数量太多（或未知），无法在单个响应中返回，则可以在多个请求中拆分检索。由于复合存储桶本质上是扁平的，因此请求的大小正好是响应中将返回的复合存储桶的数量（假设它们至少是要返回的复合存储桶的大小）。如果应该检索所有组合存储桶，最好使用较小的大小（例如100或1000），然后使用after参数检索下一个结果。例如：

```json
GET /_search
{
    "aggs" : {
        "my_buckets": {
            "composite" : {
                "size": 2,
                "sources" : [
                    { "date": { "date_histogram": { "field": "timestamp", "calendar_interval": "1d" } } },
                    { "product": { "terms": {"field": "product" } } }
                ]
            }
        }
    }
}
```

返回

```json
{
    ...
    "aggregations": {
        "my_buckets": {
            "after_key": { 
                "date": 1494288000000,
                "product": "mad max"
            },
            "buckets": [
                {
                    "key": {
                        "date": 1494201600000,
                        "product": "rocky"
                    },
                    "doc_count": 1
                },
                {
                    "key": {
                        "date": 1494288000000,
                        "product": "mad max"
                    },
                    "doc_count": 2
                }
            ]
        }
    }
}
```

after参数可用于检索上一轮返回的最后一个组合桶之后的组合桶。对于下面的示例，可以在after_键中找到最后一个bucket，并使用以下命令检索下一轮结果：

```json
GET /_search
{
    "aggs" : {
        "my_buckets": {
            "composite" : {
                "size": 2,
                 "sources" : [
                    { "date": { "date_histogram": { "field": "timestamp", "calendar_interval": "1d", "order": "desc" } } },
                    { "product": { "terms": {"field": "product", "order": "asc" } } }
                ],
                "after": { "date": 1494288000000, "product": "mad max" } 
            }
        }
    }
}
```

### 下级聚合

与任何多bucket聚合一样，复合聚合可以包含子聚合。这些子聚合可用于计算此父聚合创建的每个组合存储桶的其他存储桶或统计信息。例如，以下示例计算每个复合存储桶中字段的平均值：

```json
GET /_search
{
    "aggs" : {
        "my_buckets": {
            "composite" : {
                 "sources" : [
                    { "date": { "date_histogram": { "field": "timestamp", "calendar_interval": "1d", "order": "desc" } } },
                    { "product": { "terms": {"field": "product" } } }
                ]
            },
            "aggregations": {
                "the_avg": {
                    "avg": { "field": "price" }
                }
            }
        }
    }
}
```

返回

```json
{
    ...
    "aggregations": {
        "my_buckets": {
            "after_key": {
                "date": 1494201600000,
                "product": "rocky"
            },
            "buckets": [
                {
                    "key": {
                        "date": 1494460800000,
                        "product": "apocalypse now"
                    },
                    "doc_count": 1,
                    "the_avg": {
                        "value": 10.0
                    }
                },
                {
                    "key": {
                        "date": 1494374400000,
                        "product": "mad max"
                    },
                    "doc_count": 1,
                    "the_avg": {
                        "value": 27.0
                    }
                },
                {
                    "key": {
                        "date": 1494288000000,
                        "product" : "mad max"
                    },
                    "doc_count": 2,
                    "the_avg": {
                        "value": 22.5
                    }
                },
                {
                    "key": {
                        "date": 1494201600000,
                        "product": "rocky"
                    },
                    "doc_count": 1,
                    "the_avg": {
                        "value": 10.0
                    }
                }
            ]
        }
    }
}
```

[更多](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/search-aggregations-bucket.html)

