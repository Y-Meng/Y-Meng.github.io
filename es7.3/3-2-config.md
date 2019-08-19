# 配置ES

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/settings.html)


ElasticSearch具有良好的默认值，只需要很少的配置。可以使用群集更新设置API在正在运行的群集上更改大多数设置。

配置文件应包含特定于节点的设置（如node.name和路径），或节点为了能够加入群集而需要的设置（如cluster.name和network.host）。

## 配置文件路径
ES有三个配置文件：
* ElasticSearch.yml 用于配置ElasticSearch
* jvm.options 用于配置ElasticSearch JVM设置
* log4j2.properties 配置ElasticSearch日志记录属性

这些文件位于config目录中，其默认位置取决于安装是否来自存档分发压缩包（tar.gz或zip）或包分发（debian或rpm包）。

对于存档分发压缩包，配置目录位置默认为$es_home/config。
配置目录的位置可以通过ES_PATH_CONF 环境变量更改，如下所示：
```bash
ES_PATH_CONF=/path/to/my/config 
./bin/elasticsearch
```
或者，您可以通过命令行或shell profile 文件 EXPORT ES_PATH_CONF环境变量。

对于其他Linux分发包，配置文件默认路径在/etc/elasticsearch。

## 配置文件格式
配置文件格式是yaml，下面是修改data和logs目录的例子
```yaml
path:
    data: /var/lib/elasticsearch
    logs: /var/log/elasticsearch
```
设置也可以像下面一样展开
```yaml
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
```

## 环境变量替换

配置文件中使用${…}符号引用的环境变量将替换为环境变量的值，例如： 
```yaml
node.name:    ${HOSTNAME}
network.host: ${ES_NETWORK_HOST}
```

# 设置JVM选项

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/jvm-options.html)


您很少需要更改Java虚拟机（JVM）选项。如果这样做，最可能的更改是设置堆大小。
本文的其余部分详细解释了如何设置JVM选项。 

设置jvm首选方法是通过jvm.options配置文件设置jvm选项（包括系统属性和jvm标志）。
此文件的默认位置是config/jvm.options（从tar或zip分发版安装时）和/etc/elasticsearch/jvm.options（从debian或rpm包安装时）。 

此文件包含以下特殊语法的以行分隔的JVM参数列表：

* 忽略空白行
* 以#开头的行被视为注释并被忽略。 
* 以-开头的行被视为独立于JVM版本应用的JVM选项。
* 以数字开头后跟:后跟-的行被视为一个JVM选项，该选项仅在JVM版本与数字匹配时适用。
* 以数字开头后跟-后跟:后跟-的行被视为一个JVM选项，仅当JVM的版本大于或等于该数字时才适用。
* 以数字开头后跟-后跟数字后跟:后跟-的行被视为一个JVM选项，仅当JVM的版本在两个数字间（包含两数字）时才适用。
* 其余格式都会被拒绝
```bash
# 这是一行注释
-Xmx2g
# 下面是指定特殊版本的配置
8:-Xmx2g
# 下面是指定大于等于特殊版本的配置
8-:-Xmx2g
# 下面是指定特殊版本段的配置
8-9:-Xmx2g
```

您可以向这个文件添加自定义的JVM标志，并将这个配置检查到您的版本控制系统中。 

设置Java虚拟机选项的另一种机制是通过ES_JAVA_OPTS 环境变量。例如：
```bash
export ES_JAVA_OPTS="$ES_JAVA_OPTS -Djava.io.tmpdir=/path/to/temp/dir"
./bin/elasticsearch
```



JVM有一个内置的机制来观察java_tool_options环境变量。
我们故意忽略打包脚本中的这个环境变量。
其主要原因是，在某些操作系统（如Ubuntu）上，默认情况下通过此环境变量安装的代理不希望干扰ElasticSearch。 

此外，一些其他Java程序支持JAVA_OPTS环境变量。
这不是JVM中内置的机制，而是生态系统中的约定。
但是，我们不支持这个环境变量，而是支持通过jvm.options文件或环境变量ES_JAVA_OPTS 设置jvm选项。

# 安全配置

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/secure-settings.html)



有些设置是敏感的，依赖文件系统权限来保护它们的值是不够的。
对于这个用例，ElasticSearch提供了一个密钥库和elasticsearch-keystore工具来管理密钥库中的设置。

> 这里的所有命令都应该使用运行ElasticSearch的用户运行。

> 注意：只有一些设置设计为从密钥库中读取。但是，密钥库没有有效性来阻止不支持的设置。向密钥库添加不支持的设置将导致ElasticSearch向密钥库添加其他不支持的设置将导致ElasticSearch无法启动。请参阅每个设置的文档，以查看它是否作为密钥库的一部分受到支持。

> 所有对密钥库的所有修改只有在重新启动ElasticSearch之后才会生效。 

> ElasticSearch密钥库目前只提供模糊处理。以后会增加密码保护。

这些设置，就像elasticsearch.yml配置文件中的常规设置一样，需要在集群中的每个节点上指定。
当前，所有安全设置都是特定于节点的设置，每个节点上必须具有相同的值。

## 1.创建keystore
```bash
bin/elasticsearch-keystore create
```
文件elasticsearch.keystore将与elasticsearch.yml一起创建。

## 2.列出keystore中的设置
```bash
bin/elasticsearch-keystore list
```

## 3.添加字符串设置
敏感的字符串设置，如云插件的身份验证凭据，可以使用add命令添加：
```bash
bin/elasticsearch-keystore add the.setting.name.to.set
```
工具将提示输入设置值。要通过stdin传递值，请使用--stdin标志：
```bash
cat /file/containing/setting/value | bin/elasticsearch-keystore add --stdin the.setting.name.to.set
```

## 4.添加文件设置


您可以使用add-file命令添加敏感文件，如云插件的身份验证密钥文件。
请确保将文件路径作为参数包含在设置名称之后。 
```bash
bin/elasticsearch-keystore add-file the.setting.name.to.set /path/example-file.json
```

## 5.删除设置
使用remove命令从keystore中删除设置
```bash
bin/elasticsearch-keystore remove the.setting.name.to.remove
```

## 6.可重新加载的安全设置


就像elasticsearch.yml中的设置值一样，对密钥库内容的更改不会自动应用于正在运行的elasticsearch节点。重新读取设置需要重新启动节点。
但是，某些安全设置标记为可重新加载。这样的设置可以在正在运行的节点上重新读取和应用。 

所有安全设置的值（是否可重载）必须在所有群集节点上相同。
更改所需的安全设置后，使用bin/elasticsearch keystore add命令，调用：
```bash
POST _nodes/reload_secure_settings
```

这个API将解密并重新读取每个集群节点上的整个密钥库，但只应用可重载的安全设置。对其他设置的更改将在下次重新启动之前生效。
调用返回后，重新加载就完成了，这意味着依赖于这些设置的所有内部数据结构都已更改。
一切都应该看起来好像设置从一开始就有了新的值。


更改多个可重新加载的安全设置时，在每个群集节点上修改所有设置，然后发出重新加载安全设置调用，而不是在每次修改后重新加载。 

# 日志配置（部分）
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/logging.html)

ES使用Log4j2作为日志输出工具，可以通过log4j2.properties文件配置日志输出属性。
ElasticSearch公开了三个属性：${sys:es.logs.base_path}, ${sys:es.logs.cluster_name}, 和 ${sys:es.logs.node_name}
可以在配置文件中引用以确定日志文件的位置。
属性${sys:es.logs.base_path}将解析为日志目录，
${sys:es.logs.cluster_name}将解析为群集名称（在默认配置中用作日志文件名的前缀），
${sys:es.logs.node_name}将解析为节点名称（如果显式设置了节点名称）。

假如，有一个配置日志目录（path.logs）是var/log/es；集群名称为production；则
${sys:es.logs.base_path} = /var/log/es
${sys:es.logs.base_path}${sys:file.separator}${sys:es.logs.cluster_name}.log = /var/log/es/production.log
```yaml
######## Server JSON ############################
# 1.配置RollingFile appender
appender.rolling.type = RollingFile 
appender.rolling.name = rolling
# 2.日志输出文件名称
appender.rolling.fileName = ${sys:es.logs.base_path}${sys:file.separator}${sys:es.logs.cluster_name}_server.json 
# 3.使用json格式输出
appender.rolling.layout.type = ESJsonLayout 
# 4.type_name是一个填充ESJsonLayout中类型字段的标志。它可以用来在解析日志时更容易地区分不同类型的日志。
appender.rolling.layout.type_name = server 
# 5.设置日志滚动路径格式，通过i递增
appender.rolling.filePattern = ${sys:es.logs.base_path}${sys:file.separator}${sys:es.logs.cluster_name}-%d{yyyy-MM-dd}-%i.json.gz 
appender.rolling.policies.type = Policies
# 6.使用基于时间滚动的策略
appender.rolling.policies.time.type = TimeBasedTriggeringPolicy 
# 7.时间滚动间隔
appender.rolling.policies.time.interval = 1 
# 8.在日边界上对齐（而不是每隔24小时滚动一次）
appender.rolling.policies.time.modulate = true 
# 9.使用基于文件大小的滚动策略
appender.rolling.policies.size.type = SizeBasedTriggeringPolicy 
# 10.日志文件大于256M滚动文件
appender.rolling.policies.size.size = 256MB 
appender.rolling.strategy.type = DefaultRolloverStrategy
appender.rolling.strategy.fileIndex = nomax
# 11.滚动日志时使用删除操作
appender.rolling.strategy.action.type = Delete 
appender.rolling.strategy.action.basepath = ${sys:es.logs.base_path}
# 12.仅删除与文件模式匹配的日志
appender.rolling.strategy.action.condition.type = IfFileName 
# 13.模式是只删除主日志
appender.rolling.strategy.action.condition.glob = ${sys:es.logs.cluster_name}-* 
# 14.仅当累积了太多压缩日志时才删除
appender.rolling.strategy.action.condition.nested_condition.type = IfAccumulatedFileSize 
# 15.压缩日志的大小条件为2 GB
appender.rolling.strategy.action.condition.nested_condition.exceeds = 2GB 
################################################
```

# 审核设置【x-pack】
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/auditing-settings.html#event-audit-settings)

# 跨集群副本设置【x-pack】
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/ccr-settings.html)

# 索引生命周期管理【x-pack】
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-settings.html)

# 协议设置【x-pack】
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/license-settings.html)

# 机器学习设置【x-pack】
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/ml-settings.html)

# 监控设置【x-pack】
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/monitoring-settings.html)

# 安全设置【x-pack】
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-settings.html)

# SQL访问设置【x-pack】
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/sql-settings.html)

# 报警设置【x-pack】
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/notification-settings.html)
