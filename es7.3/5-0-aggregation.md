[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/search-aggregations.html)

# 聚合

聚合框架帮助提供基于搜索查询的聚合数据。它基于称为聚合（aggregations）的简单构建块，可以组合这些构建块以构建数据的复杂摘要。

聚合可以看作是在一组文档上构建分析信息的工作单元。执行的上下文定义了该文档集是什么（例如，在搜索请求的已执行查询/筛选器的上下文中执行顶级聚合）。

聚合有许多不同的类型，每种类型都有自己的目的和输出。为了更好地理解这些类型，通常更容易将它们分为四个主要系列：

## 1.桶聚合（Bucket）

生成存储桶的聚合系列，其中每个存储桶都与一个键和一个文档条件相关联。执行聚合时，将对上下文中的每个文档计算所有bucket条件，当某个条件匹配时，将认为该文档属于相关bucket。在聚合过程结束时，我们将得到一个bucket列表——每个bucket都有一组“属于”它的文档。

## 2.度量聚合（Metric）

在一组文档上跟踪和计算度量的聚合。

## 3.矩阵聚合（Matrix）

对多个字段进行操作并基于从请求的文档字段中提取的值生成矩阵结果的聚合系列。与度量和bucket聚合不同，此聚合系列尚**不支持脚本**。

## 4.流水线（Pipeline）

聚合其他聚合的输出及其相关度量的聚合。

接下来是有趣的部分。由于每个bucket有效地定义了一个文档集（属于bucket的所有文档），因此可以在bucket级别上潜在地关联聚合，这些聚合将在该bucket的上下文中执行。这就是聚合的真正威力所在：聚合可以嵌套！



> Bucketing聚合可以有子聚合（Bucketing或metric）。将为其父聚合生成的存储桶计算子聚合。嵌套聚合的级别/深度没有硬限制（可以将聚合嵌套在“父”聚合下，而“父”聚合本身是另一个更高级别聚合的子聚合）。



> 聚合操作基于double表示。因此，在绝对值大于2^53的long上运行时，结果可能是近似的。



# 聚合结构

以下代码片段捕获聚合的基本结构：

```json
"aggregations" : {
    "<aggregation_name>" : {
        "<aggregation_type>" : {
            <aggregation_body>
        }
        [,"meta" : {  [<meta_data_body>] } ]?
        [,"aggregations" : { [<sub_aggregation>]+ } ]?
    }
    [,"<aggregation_name_2>" : { ... } ]*
}
```

JSON中的aggregations对象（也可以使用键aggs）保存要计算的聚合。

每个聚合都与用户定义的逻辑名称相关联（例如，如果聚合计算平均价格，则将其命名为avg_price是有意义的）。这些逻辑名称还将用于唯一标识响应中的聚合。 

每个聚合都有一个特定的类型（<aggregation_type>在上面的代码片段中），通常是命名聚合体中的第一个键。根据聚合的性质，每种类型的聚合都定义自己的主体（例如，特定字段上的avg聚合将定义计算平均值的字段）。

在聚合类型定义的同一级别上，可以选择定义一组附加聚合，但这仅在您定义的聚合具有bucketing性质时才有意义。在这种情况下，将为bucketing聚合生成的所有bucket计算您在bucketing聚合级别上定义的子聚合。例如，如果在范围聚合下定义一组聚合，则将为定义的范围存储桶计算子聚合。

### 数值来源

一些聚合处理从聚合文档中提取的值。通常，这些值将从使用聚合字段键设置的特定文档字段中提取。还可以定义一个脚本来生成值（每个文档）。

为聚合配置字段（field）和脚本（script）设置时，脚本将被视为值脚本（value script）。正常脚本在文档级别进行评估（即脚本可以访问与文档相关的所有数据），而值脚本在值级别进行评估。在这种模式下，将从配置的字段中提取值，并使用脚本对这些值脚本应用“转换”。

> 使用脚本时，还可以定义lang和params设置。前者定义所使用的脚本语言（假设Elasticsearch中有适当的语言，默认情况下或作为插件）。后者允许将脚本中的所有“动态”表达式定义为参数，从而使脚本能够在调用之间保持自身的静态（这将确保在Elasticsearch中使用缓存的已编译脚本）。

Elasticsearch使用映射中字段的类型，以确定如何运行聚合并格式化响应。但是，Elasticsearch无法找出这一信息的情况有两种：未映射字段（例如，在跨多个索引的搜索请求中，只有一些字段具有字段映射）和纯脚本。对于这些情况，可以使用value_type选项向Elasticsearch提供提示，该选项接受以下值：string、long（适用于所有整数类型）、double（适用于所有十进制类型，如float或scaled_float）、date、ip和boolean。