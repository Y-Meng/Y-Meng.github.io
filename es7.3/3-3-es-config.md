# ES的重要配置

## path.data 和 path.logs

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/path-settings.html)

如果使用.zip或.tar.gz压缩安装包，数据和日志目录目录默认在$ES_HOME的子目录。
如果将这些重要文件夹保留在其默认位置，则在将ElasticSearch升级到新版本时，删除这些文件夹的风险很高。

在生产使用中，您肯定希望更改数据和日志文件夹的位置，如：
```yaml
path:
  logs: /var/log/elasticsearch
  data: /var/data/elasticsearch
```

RPM和Debian发行版已经为数据和日志使用自定义路径。

path.data设置可以设置为多个路径，在这种情况下，所有路径都将用于存储数据（尽管属于单个shard的文件都将存储在同一个数据路径上）：

```yaml
path:
  data:
    - /mnt/elasticsearch_1
    - /mnt/elasticsearch_2
    - /mnt/elasticsearch_3
```

## cluster.name

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster.name.html)



节点只能在与群集中的所有其他节点共享cluster.name时加入群集。
默认名称是ElasticSearch，但您应该将其更改为描述集群的合适的名称。 
```yaml
cluster.name: logging-prod
```

请确保不要在不同的环境中重用相同的集群名称，否则最终可能会导致节点加入错误的集群。

## node.name

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/node.name.html)

ElasticSearch使用node.name作为特定ElasticSearch实例的可读标识符，因此它包含在许多API的响应中。
它默认为启动ElasticSearch时计算机拥有的主机名，但可以在ElasticSearch.yml中显式配置，如下所示：

```yaml
node.name: node-1
```

## network.host

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/node.name.html)

默认情况下，ElasticSearch仅绑定到环回地址-，例如127.0.0.1和[::1]。这足以在服务器上运行单个开发节点。

> 事实上，可以从单个节点上相同的$ES_HOME启动多个节点。这对于测试ElasticSearch形成集群的能力很有用，但它不是推荐用于生产的配置。

为了与其他服务器上的节点组成集群，您的节点需要绑定到非环回地址。虽然有许多网络设置，但通常只需配置network.host：
```yaml
network.host: 192.168.1.10
```

network.host设置还了解一些特殊值，如_local_、_site_、_global_和修改器，如：ip4和：ip6，
其详细信息可在[network.host的特殊值](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-network.html#network-interface-values)中找到。

一旦为network.host提供自定义设置，elasticsearch就假定您正在从开发模式切换到生产模式，并将许多系统启动检查从警告升级到异常。
更多信息请参见开发模式与生产模式。

## 重要的服务发现和集群信息设置

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/discovery-settings.html)

在进入生产之前，应该配置两个重要的发现和集群形成设置，以便集群中的节点可以彼此发现并选择主节点。

* discovery.seed_hosts

在没有任何网络配置的情况下，ElasticSearch将绑定到可用的环回地址，并扫描本地端口9300到9305，以尝试连接到同一服务器上运行的其他节点。
这提供了一种自动集群的体验，而无需进行任何配置。

当您要与其他主机上的节点组成群集时，必须使用discovery.seed_hosts设置提供群集中其他节点的列表，
这些节点符合主服务器的条件，并且可能是活动的和可联系的，以便为发现过程设定种子。
此设置通常应包含群集中所有符合主服务器条件的节点的地址。此设置包含主机数组或逗号分隔的字符串。
每个值的形式应为host:port或host（其中port默认为设置transport.profiles.default.port，如果未设置，则返回transport.port）。
请注意，IPv6主机必须加括号。此设置的默认值为127.0.0.1，[::1]。

* cluster.initial_master_nodes

当您第一次启动一个全新的ElasticSearch集群时，有一个集群引导步骤，它确定在第一次选举中计票的主合格节点集。
在开发模式下，在没有配置发现设置的情况下，此步骤由节点本身自动执行。
由于这种自动引导固有的不安全性，当您在生产模式下启动一个全新集群时，必须明确列出主合格节点，其投票应在第一次选举中计算。
此列表是使用cluster.initial_master_nodes设置设置的。

```yaml
discovery.seed_hosts:
   - 192.168.1.10:9300
   - 192.168.1.11 # 端口将默认为transport.profiles.default.port，如果未指定，则返回transport.port。 
   - seeds.mydomain.com # 如果主机名解析为多个IP地址，则节点将尝试在所有解析的地址中发现其他节点。
cluster.initial_master_nodes: 
   - master-node-a
   - master-node-b
   - master-node-c
# 最初的主节点应该通过node.name来标识，node.name默认为其主机名。确保cluster.initial_master_nodes中的值与node.name完全匹配。
# 如果您的节点名使用完全限定的域名（如master-node-a.example.com），则必须在此列表中使用完全限定的名称；
# 反之，如果node.name是没有任何尾部限定符的裸主机名，则还必须省略cluster.initial_master_nodes中的尾部限定符。
```

有关详细信息，请参阅[引导群集](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-bootstrap-cluster.html)以及发现和[群集形成设置](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-settings.html)。

## 设置堆大小

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/heap-size.html)


默认情况下，ElasticSearch告诉JVM使用最小和最大为1GB的堆。
当转移到生产环境时，配置堆大小以确保ElasticSearch具有足够的堆可用性是很重要的。 

ElasticSearch将通过xms（最小堆大小）和xmx（最大堆大小）设置分配jvm.options中指定的整个堆。
您应该将这两个设置设置设置为相等。

这些设置的值取决于服务器上可用的RAM数量：
* 将xmx和xms设置为不超过物理RAM的50%。ElasticSearch需要内存用于除JVM堆之外的其他用途，因此留出空间非常重要。例如，ElasticSearch使用堆外缓冲区进行有效的网络通信，依赖操作系统的文件系统缓存来有效地访问文件，而JVM本身也需要一些内存。观察ElasticSearch过程使用的内存比配置了xmx设置的限制多是正常的。
* 将xmx和xms设置为不超过jvm用于压缩对象指针（compressed oops）的阈值；确切的阈值有所不同，但接近32 GB。您可以通过在日志中查找以下行来验证是否低于阈值：
> heap size [1.9gb], compressed ordinary object pointers [true]
* 理想情况下，将xmx和xms设置为不超过基于零的压缩OOP的阈值；精确的阈值有所不同，但在大多数系统中26 GB是安全的，但在某些系统中可能高达30 GB。通过使用jvm选项-XX:+UnlockDiagnosticVMOptions -XX:+PrintCompressedOopsMode启动elasticsearch并查找以下行，可以验证您是否低于此阈值：
> heap address: 0x000000011be00000, size: 27648 MB, zero based Compressed Oops.
显示启用了基于零的压缩OOP。如果没有启用基于零的压缩OOP，那么您将看到如下行：
> heap address: 0x0000000118400000, size: 28672 MB, Compressed Oops with base: 0x00000001183ff000

ElasticSearch可用的堆越多，它可以用于内部缓存的内存就越多，但留给操作系统用于文件系统缓存的内存就越少。
此外，较大的堆可能导致较长的垃圾收集暂停。

以下是通过jvm.options文件设置堆大小的示例：
```bash
-Xms2g # 最小堆内存2g 
-Xmx2g # 最大堆内存2g
```

还可以通过环境变量设置堆大小。这可以通过注释jvm.options文件中的xms和xmx设置并通过ES_JAVA_OPTS设置这些值来完成：
```bash
ES_JAVA_OPTS="-Xms2g -Xmx2g" ./bin/elasticsearch 
ES_JAVA_OPTS="-Xms4000m -Xmx4000m" ./bin/elasticsearch
```

> 注意：为Windows服务配置堆与上述不同。最初为Windows服务填充的值可以如上所述进行配置，但在安装该服务后会有所不同。有关其他详细信息，请参阅[Windows服务文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/zip-windows.html#windows-service)。

## JVM堆转储路径

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/heap-dump-path.html)

默认情况下，elasticsearch将jvm设置为将内存不足异常的堆转储到默认数据目录（RPM和Debian包分发为/var/lib/elasticsearch，以及tar和zip存档安装包的elasticsearch安装根目录下的数据目录）。
如果此路径不适合接收堆转储，则应在jvm.options中修改条目-xx:heapDumpPath=…。
如果指定目录，则JVM将根据正在运行的实例的PID为堆转储生成一个文件名。
如果指定固定的文件名而不是目录，则当JVM需要对内存不足异常执行堆转储时，该文件不能存在，否则堆转储将失败。 

## GC日志

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/gc-logging.html)

默认情况下，ElasticSearch启用GC日志。它们在jvm.options中配置，并默认为与elasticsearch日志相同的默认位置。
默认配置每64MB新增一个日志文件，最多可占用2GB的磁盘空间。

## 临时目录

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/es-tmpdir.html)

默认情况下，ElasticSearch使用启动脚本直接在系统临时目录下创建的专用临时目录。

在一些Linux发行版上，如果文件和目录最近没有被访问，系统实用程序将从/tmp中清除它们。
如果长时间不使用需要临时目录的功能，这可能导致在运行ElasticSearch时删除私有临时目录。
如果随后使用了需要临时目录的功能，则会导致问题。

如果使用.deb或.rpm包安装elasticsearch并在systemd下运行它，那么elasticsearch使用的私有临时目录将从定期清理中排除。

但是，如果您打算在Linux上运行.tar.gz发行版很长一段时间，那么您应该考虑为ElasticSearch创建一个专用的临时目录，该目录不在将清除旧文件和目录的路径下。
此目录应设置权限，以便只有运行ElasticSearch的用户可以访问它。然后在启动ElasticSearch之前，设置$ES_TMPDIR环境变量指向它。

## JVM致命错误日志

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/error-file-path.html)

默认情况下，elasticsearch配置jvm将致命错误日志写入默认日志目录（RPM和Debian包为/var/log/elasticsearch，以及tar和zip存档分发包在elasticsearch安装根目录下）。
这些是当JVM遇到致命错误（例如，分段错误）时产生的日志。如果此路径不适合接收日志，则应在jvm.options中修改条目-xx:errorfile=…转到备用路径。 