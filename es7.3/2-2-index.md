# 索引文档

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/getting-started-index.html)


集群启动并运行后，就可以索引一些数据了。ElasticSearch有多种输入数据的方式，但最终它们都做了相同的事情：将JSON文档放入ElasticSearch索引。 


您可以通过一个简单的POST请求来直接执行此操作，该请求标识要将文档添加到的索引，并在请求正文中指定一个或多个**field：value**对：
```bash
PUT /customer/_doc/1
{
  "name": "John Doe"
}
```
此请求将自动创建客户索引（如果该索引尚不存在），添加一个ID为1的新文档，并存储和索引名称字段。

由于这是一个新文档，响应显示操作的结果是创建了文档的版本1： 
```json
{
  "_index" : "customer",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 2,
    "failed" : 0
  },
  "_seq_no" : 26,
  "_primary_term" : 4
}
```
新文档可以立即从群集中的任何节点上查询。您可以使用指定其文档ID的GET请求来检索它：
```bash
GET /customer/_doc/1
```
响应指示找到具有指定ID的文档，并显示索引的原始源字段。
```json
{
  "_index" : "customer",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "_seq_no" : 26,
  "_primary_term" : 4,
  "found" : true,
  "_source" : {
    "name": "John Doe"
  }
}
```

# 批量处理

除了能够索引、更新和删除单个文档之外，ElasticSearch还提供了使用批量API批量执行上述任何操作的能力。
此功能非常重要，因为它提供了一种非常有效的机制，可以以尽可提高的网络交互速度尽可能快速地执行多个操作。

作为一个简单示例，以下调用在一个批量操作中索引两个文档（ID-1和ID-2）：
```bash
POST /customer/_bulk?pretty
{"index":{"_id":"1"}}
{"name": "John Doe" }
{"index":{"_id":"2"}}
{"name": "Jane Doe" }
```

此示例更新第一个文档（ID为1），然后在一个批量操作中删除第二个文档（ID为2）：
```bash
POST /customer/_bulk
{"update":{"_id":"1"}}
{"doc": { "name": "John Doe becomes Jane Doe" } }
{"delete":{"_id":"2"}}
```
请注意，对于删除操作，删除后没有对应的源文档，因为删除只需要删除文档的ID。

批量API不会由于其中一个操作失败而失败。如果一个操作由于任何原因失败，它将继续处理后面的其余操作。
当批量API返回时，它将为每个操作提供一个状态（以相同的发送顺序），以便您可以检查特定操作是否失败。

# 示例数据集

既然我们已经大致了解了基础知识，那么让我们试着研究一个更现实的数据集。
这里准备了一个客户银行账户信息的虚构JSON文档样本。每个文档都有以下架构：
```json
{
    "account_number": 0,
    "balance": 16623,
    "firstname": "Bradshaw",
    "lastname": "Mckenzie",
    "age": 29,
    "gender": "F",
    "address": "244 Columbus Place",
    "employer": "Euron",
    "email": "bradshawmckenzie@euron.com",
    "city": "Hobucken",
    "state": "CO"
}
```

此数据是使用www.json-generator.com生成的，因此请忽略数据的实际值和语义，因为这些都是随机生成的。 

您可以从[这里](https://github.com/elastic/elasticsearch/blob/master/docs/src/test/resources/accounts.json?raw=true)下载示例数据集（accounts.json）。
将其提取到当前目录，然后按如下方式将其加载到集群中：
```bash
curl -H "Content-Type: application/json" -XPOST "localhost:9200/bank/_bulk?pretty&refresh" --data-binary "@accounts.json"
curl "localhost:9200/_cat/indices?v"
```
反馈结果：
```text
health status index uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   bank  l7sSYV2cQXmu6_4rJWVIww   5   1       1000            0    128.6kb        128.6kb
```
这意味着我们刚刚成功地将1000个文档批量索引到银行索引中。