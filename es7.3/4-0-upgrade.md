# Elasticsearch升级
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/setup-upgrade.html)

Elasticsearch通常可以使用滚动升级过程进行升级，因此升级不会中断服务。支持滚动升级的版本：
* 小版本之间
* 5.6->6.8
* 6.8->7.3.2

下表展示了升级到7.3.2的建议升级路径：

| 升级版本  | 7.3.2推荐升级路径 |
| :---------- | :---------- |
| 7.0–7.2 | Rolling upgrade to 7.3.2 |
| 6.8   | Rolling upgrade to 7.3.2 |
| 6.0–6.7   | 1.Rolling upgrade to 6.8 2.Rolling upgrade to 7.3.2 |
| 5.6    | 1.Rolling upgrade to 6.8  2.Rolling upgrade to 7.3.2 |
| 5.0–5.5   | 1.Rolling upgrade to 5.6  2.Rolling upgrade to 6.8  3.Rolling upgrade to 7.3.2 |

注意：下面的升级路径是不支持的
* 6.8 -> 7.0
* 6.7 -> 7.1-7.3.2

Elasticsearch可以读取以前主要版本中创建的索引。如果在5.x或之前版本中创建了索引，则必须在升级到7.3.2之前重新编制索引或删除索引。
如果存在不兼容的索引，则Elasticsearch节点将无法启动。5.x或更早索引的快照无法还原到7.x群集，即使它们是由6.x群集创建的。
有关升级旧索引的信息，请参阅Reindex to upgrade。

升级到新版本的Elasticsearch时，需要升级ElasticStack中的每个产品。有关详细信息，请参阅elastic stack安装和升级指南。

要从6.7或更早版本直接升级到7.3.2，必须关闭群集，安装7.3.2，然后重新启动。有关详细信息，请参阅完全群集重新启动升级。