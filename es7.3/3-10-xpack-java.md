# xpack java客户端
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/setup-xpack-client.html)

注意：
> 7.0.0后弃用，TransportClient已被弃用，取而代之的是Java高级REST客户端，并将在Elasticsearch 8.0中删除。迁移指南描述了迁移所需的所有步骤。

如果要在安装了X-Pack的集群中使用Java TransportClient，则必须下载并配置X-Pack TransportClient。

1. 添加xpack transport jar文件到classpath。