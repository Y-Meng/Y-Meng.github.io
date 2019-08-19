# 启动运行ES集群

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/getting-started-install.html)


要试用ElasticSearch，您可以在ElasticSearch服务上创建一键式云部署，或者在您自己的Linux、MacOS或Windows机器上设置多节点ElasticSearch集群。 

## 在Linux、MacOS或Windows机器上运行ES

在ElasticSearch服务上创建集群时，会自动获得一个三节点集群。
通过从tar或zip存档安装，您可以在本地启动多个ElasticSearch实例，以查看多节点集群的行为。 

在本地运行三节点ElasticSearch集群：

1.下载系统对应的ES安装包

Linux: [elasticsearch-7.3.0-linux-x86_64.tar.gz](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.3.0-linux-x86_64.tar.gz)
```bash
curl -L -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.3.0-linux-x86_64.tar.gz
```
macOS: [elasticsearch-7.3.0-darwin-x86_64.tar.gz](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.3.0-darwin-x86_64.tar.gz)
```bash
curl -L -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.3.0-darwin-x86_64.tar.gz
```
Windows: [elasticsearch-7.3.0-windows-x86_64.zip](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.3.0-windows-x86_64.zip)

2.解压安装包
Linux:
```bash
tar -xvf elasticsearch-7.3.0-linux-x86_64.tar.gz
```
macOS:
```bash
tar -xvf elasticsearch-7.3.0-darwin-x86_64.tar.gz
```
Windows PowerShell:
```bash
Expand-Archive elasticsearch-7.3.0-windows-x86_64.zip
```

3.在bin目录启动ES:
Linux and macOS:
```bash
cd elasticsearch-7.3.0/bin
./elasticsearch
```

Windows:
```bash
cd %PROGRAMFILES%\Elastic\Elasticsearch\bin
.\elasticsearch.exe
```

现在您有一个单节点ElasticSearch集群正在运行！ 

4.再启动两个ElasticSearch实例，这样您就可以看到典型的多节点集群的行为。您需要为每个节点指定唯一的数据和日志路径。 

Linux and macOS:
```bash
./elasticsearch -Epath.data=data2 -Epath.logs=log2
./elasticsearch -Epath.data=data3 -Epath.logs=log3
```

Windows:
```bash
.\elasticsearch.exe -Epath.data=data2 -Epath.logs=log2
.\elasticsearch.exe -Epath.data=data3 -Epath.logs=log3
```
附加节点被分配唯一的ID。因为您在本地运行所有三个节点，所以它们会自动将集群与第一个节点连接起来。

5.使用cat health api验证您的三节点集群是否正在运行。cat apis以比原始JSON更容易读取的格式返回关于集群和索引的信息。 

通过向ElasticSearch REST API提交HTTP请求，可以直接与集群交互。
本指南中的大多数示例允许复制对应的curl命令，并从命令行将请求提交到本地ElasticSearch实例。
如果已经安装并运行了kibana，那么也可以打开kibana并通过dev控制台提交请求。

> 当您准备好在自己的应用程序中开始使用ElasticSearch时，您将需要使用ElasticSearch[开发语言客户端](https://www.elastic.co/guide/en/elasticsearch/client/index.html)。

```bash
GET /_cat/health?v
```

响应应表明ElasticSearch集群的状态为绿色，它有三个节点： 
```bash
epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1565052807 00:53:27  elasticsearch green           3         3      6   3    0    0        0             0                  -                100.0%
```

> 如果只运行一个ElasticSearch实例，集群状态将保持黄色。单节点集群是完全正常的，但不能将数据复制到另一个节点以提供弹性。副本分片必须可用，群集状态为绿色。如果集群状态为红色，则某些数据不可用。

## 更多安装方式
通过从软件压缩文件安装ElasticSearch，您可以轻松地在本地安装和运行多个实例，以便进行尝试。
要运行单个实例，可以在Docker容器中运行ElasticSearch，使用Linux上的DEB或RPM包安装ElasticSearch，使用MacOS上的自制软件安装，或使用Windows上的MSI包安装程序安装。
有关详细信息，请参见[安装ElasticSearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/install-elasticsearch.html)。