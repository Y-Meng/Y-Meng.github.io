[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/install-elasticsearch.html)

# 托管的ElasticSearch
您可以在自己的硬件上运行ElasticSearch，或者在ElasticCloud上使用我们托管的ElasticSearch服务。
ElasticSearch服务可在AWS和GCP上使用。[免费试用ElasticSearch服务](https://www.elastic.co/cloud/elasticsearch-service/signup)。

# 使用压缩包在Linux系统上安装

ElasticSearch在Linux和MacOS平台上安装基于.tar.gz格式压缩包。

此包在Elastic可证下免费使用。
它包含开源和免费的商业功能以及对付费商业功能的访问。
开始为期30天的试用，试用所有付费商业功能。有关许可证级别的信息，请参阅[订阅页](https://www.elastic.co/subscriptions)。

> 压缩包包含一个内嵌的OpenJDK版本。

## 1.下载安装包

linux
```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.3.0-linux-x86_64.tar.gz
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.3.0-linux-x86_64.tar.gz.sha512
shasum -a 512 -c elasticsearch-7.3.0-linux-x86_64.tar.gz.sha512 
tar -xzf elasticsearch-7.3.0-linux-x86_64.tar.gz
cd elasticsearch-7.3.0/ 
```
也可以选择下载只包含Apache2.0协议代码的压缩包：https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-oss-7.3.0-linux-x86_64.tar.gz。

## 2.允许自动创建系统索引

一些商业特性会在ElasticSearch中自动创建系统索引。
默认情况下，ElasticSearch配置为允许自动创建索引，不需要其他步骤。
但是，如果在ElasticSearch中禁用了自动创建索引，
则必须在ElasticSearch.yml中配置action.auto_create_index以允许商业功能创建以下索引：
```bash
action.auto_create_index: .monitoring*,.watches,.triggered_watches,.watcher-history*,.ml*
```

## 3.使用命令行运行ElasticSearch
```bash
./bin/elasticsearch
```

默认情况下ES在前台运行，打印日志到标准输出流，你可以通过Ctrl-C退出。

## 4.检查ElasticSearch是否运行
可以通过http请求运行ES设备的9200端口，如果有类似下面的响应，证明ES已经正常运行。
```json
{
  "name" : "Cp8oag6",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "AT69_T_DTp-1qgIJlatQqA",
  "version" : {
    "number" : "7.3.0",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "f27399d",
    "build_date" : "2016-03-30T09:51:41.449Z",
    "build_snapshot" : false,
    "lucene_version" : "8.1.0",
    "minimum_wire_compatibility_version" : "1.2.3",
    "minimum_index_compatibility_version" : "1.2.3"
  },
  "tagline" : "You Know, for Search"
}
```

## 5.将ES作为守护进程运行

要将elasticsearch作为守护进程运行，请在命令行上指定-d，并使用-p选项将进程ID记录在文件中： 
```bash
./bin/elasticsearch -d -p pid
```
日志信息可以在$ES_HOME/logs/目录中找到。

要关闭ElasticSearch，请终止PID文件中记录的进程ID：
```bash
pkill -F pid
```

## 6.通过命令行配置ES


默认情况下，elasticsearch从$es_home/config/elasticsearch.yml文件加载其配置。
[配置ElasticSearch](es7.3/3-2-config)中解释了此配置文件的格式。 


可以在配置文件中指定的任何设置也可以在运行命令行上指定，使用-e语法，如下所示： 
```bash
./bin/elasticsearch -d -Ecluster.name=my_cluster -Enode.name=node_1
```

> 通常，任何集群范围的设置（如cluster.name）都应该添加到elasticsearch.yml配置文件中，而任何节点特定的设置（如node.name）都可以在命令行中指定。

# 安装目录布局

安装包是完全独立的。默认情况下，所有文件和目录都包含在$ES_HOME(解压安装包时创建的目录)中。
 
这非常方便，因为您不必创建任何目录即可开始使用ElasticSearch，卸载ElasticSearch与删除$ES_HOME目录一样简单。
但是，建议更改config目录、数据目录和logs目录的默认位置，以便以后不删除重要数据。

类型 | 说明 | 默认位置 | 设置
-|-|-|-
home | ES根目录($ES_HOME) | 安装包解压目录 | -
bin | ES启动和安装插件等可执行脚本所在目录 | $ES_HOME/bin | -
conf | 配置文件elasticsearch.yml所在目录 | $ES_HOME/config | ES_PATH_CONF
data | ES索引分片文件所在目录，支持多个目录 | $ES_HOME/data | path.data
logs | 日志文件所在目录 | $ES_HOME/logs | path.logs
plugins | 插件目录，每一个插件一个子目录 | $ES_HOME/plugins | -
repo | 共享文件系统存储库位置。可以容纳多个位置。文件系统存储库可以放在此处指定的任何目录的任何子目录中。 | - | path.repo
script | 脚本文件目录 | $ES_HOME/scripts | path.scripts

# 下一步
现在已经设置了一个测试ElasticSearch环境。
在开始认真开发或使用ElasticSearch投入生产之前，必须进行一些附加设置：

* 了解如何配置ElasticSearch。
* 配置重要的ElasticSearch设置。
* 配置重要的系统设置。