[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/install-elasticsearch.html)

# 托管的ElasticSearch
您可以在自己的硬件上运行ElasticSearch，或者在ElasticCloud上使用我们托管的ElasticSearch服务。
ElasticSearch服务可在AWS和GCP上使用。[免费试用ElasticSearch服务](https://www.elastic.co/cloud/elasticsearch-service/signup)。

# 一、使用压缩包在Linux系统上安装

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

## 7.安装目录布局

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



# 二、使用RPM包安装ES（生产环境推荐）

ES的rmp包可以通过[rpm repository](https://www.elastic.co/guide/en/elasticsearch/reference/current/rpm.html#rpm-repo)或者[网站](https://www.elastic.co/guide/en/elasticsearch/reference/current/rpm.html#install-rpm)下载，支持在基于rpm包安装的操作系统，例如OpenSUSE，SLES，CentOS，RedHat, 和Oracle企业版。

> 注意：RPM安装不支持比较老的rpm版本，比如SLES11和CentOS5，请使用tar包安装

此软件包在Elastic可证下可免费使用。它包含开放源代码和免费的商业功能，以及对付费商业功能的访问。开始为期30天的试用，尝试所有付费商业功能。有关许可级别的信息，请参阅订阅页。

最新稳定版本的Elasticsearch可以在[下载](https://www.elastic.co/downloads/elasticsearch)Elasticsearch页面上找到。其他版本可以在[过去的版本](https://www.elastic.co/downloads/past-releases)页上找到。

Elasticsearch包含了来自JDK维护者（GPLv2+CE）的OpenJDK的捆绑版本。要使用您自己的Java版本，请参见[JVM版本要求](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup.html#jvm-version)



## 1. 导入ES PGP key

我们使用Elasticsearch签名密钥（PGP密钥D88E42B4，可从[https://pgp.mit.edu](https://pgp.mit.edu/)获得）和指纹对所有包进行签名：

> 4609 5ACC 8548 582C 1A26 99A9 D27D 666C D88E 42B4

下载并安装公共签名key

```shell
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```

## 2. 安装rpm

### 2.1. 通过rpm repository安装

在基于RedHat的发行版的/etc/yum.repos.d/目录或基于OpenSuSE的发行版的/etc/zypp/repos.d/目录中创建一个名为elasticsearch.repo的文件，其中包含：

```ini
[elasticsearch]
name=Elasticsearch repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=0
autorefresh=1
type=rpm-md
```

并且您的存储库已准备就绪。现在可以使用以下命令之一安装Elasticsearch：

```shell
# CentOS系统和较老的基于RedHat的系统使用yum安装
sudo yum install --enablerepo=elasticsearch elasticsearch 
# Fedora系统和较新的基于RedHat的系统使用dnf安装
sudo dnf install --enablerepo=elasticsearch elasticsearch 
# OpenSUSE系统使用zypper安装
sudo zypper modifyrepo --enable elasticsearch && \
  sudo zypper install elasticsearch; \
  sudo zypper modifyrepo --disable elasticsearch 
```

> 注意：
>
> 1.默认情况下已禁用配置的存储库。这消除了在升级系统其余部分时意外升级elasticsearch的可能性。每个install或upgrade命令都必须显式地启用存储库，如上面的示例命令所示。
>
> 2.另外，还提供了一个替代包，其中仅包含Apache2.0许可下可用的功能。要安装它，请在elasticsearch.repo文件中使用以下baseurl：
>
> ```sh
> baseurl=https://artifacts.elastic.co/packages/oss-7.x/yum
> ```



### 2.2. 手动下载并安装rpm

rpm包可以通过西面的网站下载和安装：

```shell
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.x.x-x86_64.rpm
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.x.x-x86_64.rpm.sha512
# 比较下载的RPM和发布的校验的SHA，正常输出elasticsearch-{version}-x86_64.RPM:OK。
shasum -a 512 -c elasticsearch-7.x.x-x86_64.rpm.sha512 
# 安装
sudo rpm --install elasticsearch-7.x.x-x86_64.rpm
```

另外，还可以选择下载下面只包含Apache2.0 license协议特性的安装包：

https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-oss-7.6.0-x86_64.rpm

> 在基于systemd的发行版上，安装脚本将尝试设置内核参数（例如vm.max_map_count）；您可以通过屏蔽systemd-sysctl.service单元跳过这一步。

## 3. 启用自动创建系统索引【x-pack】

一些商业特性会在Elasticsearch中自动创建系统索引。默认情况下，Elasticsearch配置为允许自动创建索引，不需要其他步骤。

但是，如果在Elasticsearch中禁用了自动索引创建，则必须在Elasticsearch.yml中配置action.auto_create_index，以允许商业功能创建以下索引：

```yaml
action.auto_create_index: .monitoring*,.watches,.triggered_watches,.watcher-history*,.ml*
```

> 如果您使用的是Logstash或Beats，则很可能需要在action.auto_create_index设置中添加索引名，具体值将取决于本地配置。如果您不确定环境的正确值，可以考虑将该值设置为*以允许自动创建所有索引。



## 4. SysV init和systemd

Elasticsearch在安装后不会自动启动。如何启动和停止Elasticsearch取决于您的系统是使用SysV init还是systemd（由较新的发行版使用）。通过运行此命令，您可以知道正在使用哪个：

```shell
ps -p 1
```



### 4.1. 使用SysV init (CentOS 6) 运行ES

使用chkconfig命令配置ES系统开机时自动启动

```shell
sudo chkconfig --add elasticsearch
```

使用service命令启动和停止ES

```shell
sudo -i service elasticsearch start
sudo -i service elasticsearch stop
```

如果ES由于某些原因启动失败，将会打印失败日志到/var/log/elasticsearch/目录

### 4.2. 使用systemd (CentOS 7+) 运行ES

使用systemctl命令配置ES开机自动启动：

```shell
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch.service
```

使用systemctl命令启动和停止ES：

```shell
sudo systemctl start elasticsearch.service
sudo systemctl stop elasticsearch.service
```

同样启动日志也会输出到/var/log/elasticsearch/目录

默认es不会输出日志到systemd journal，启用journalctl日志必须移除elasticsearch.service文件中ExecStart命令行中的--quiet命令。

当systemd日志启用后，日志信息可以通过journalctl命令查看：

```shell
# tail日志
sudo journalctl -f
# 列出elasticsearch服务的日记条目，请执行以下操作：
sudo journalctl --unit elasticsearch
# 列出elasticsearch服务从给定时间开始的日记条目，请执行以下操作：
sudo journalctl --unit elasticsearch --since  "2016-10-30 18:17:16"
```

## 5. 检查ES是否运行

通过请求本地9200端口测试，返回结果类似如下：

```json
{
  "name" : "Cp8oag6",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "AT69_T_DTp-1qgIJlatQqA",
  "version" : {
    "number" : "7.6.0",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "f27399d",
    "build_date" : "2016-03-30T09:51:41.449Z",
    "build_snapshot" : false,
    "lucene_version" : "8.4.0",
    "minimum_wire_compatibility_version" : "1.2.3",
    "minimum_index_compatibility_version" : "1.2.3"
  },
  "tagline" : "You Know, for Search"
}
```

## 6. 配置ES

ES默认使用/etc/elasticsearch目录作为配置文件存储目录。此目录和此目录中的所有文件的所有权在安装的过程中都设置为root:elasticsearch，并且该目录设置了setgid标志，以便在/etc/elasticsearch下创建的任何文件和子目录也使用此所有权创建。希望能够保持此状态，以便Elasticsearch进程可以通过组权限读取此目录下的文件。

ES默认从/etc/elasticsearch/elasticsearch.yml读取配置文件，rpm同样拥有一个系统配置文件/etc/sysconfig/elasticsearch，在系统配置文件里允许配置如下变量：

| JAVA_HOME          | 设置自定义java路径                                           |
| ------------------ | ------------------------------------------------------------ |
| MAX_OPEN_FILES     | 打开文件最大数量，默认65535                                  |
| MAX_LOCKED_MEMEORY | 最大锁定内存，如果需要使用elasticsearch.yml中配置的bootsrap.memory_lock值，这里需要设置为unlimited |
| MAX_MAP_COUNT      | 进程可以使用的最大内存映射区域，如果使用mmapfs作为索引存储类型，请确保将其设置为高值。有关更多信息，请查看有关max_map_count的[inux内核文档](https://github.com/torvalds/linux/blob/master/Documentation/sysctl/vm.txt)。这是在启动Elasticsearch之前通过sysctl设置的。默认为262144。 |
| ES_PATH_CONF       | 文件目录配置，包含elasticsearch.yml，jvm.properties和log4j2.properties；默认目录/etc/elasticsearch |
| ES_JAVA_OPTS       | 要使用的任何其他JVM系统属性。                                |
| RESTART_ON_UPGRADE | 在包升级时配置重新启动，默认为false。这意味着您必须在手动安装包后重新启动Elasticsearch实例。这样做的原因是为了确保集群中的升级不会导致连续的碎片重新分配，从而导致高网络流量并减少集群的响应时间。 |

> 使用systemd的发行版要求通过systemd而不是/etc/sysconfig/elasticsearch文件配置系统资源限制。有关更多信息，请参阅[系统配置](https://www.elastic.co/guide/en/elasticsearch/reference/current/setting-system-settings.html#systemd)。
>
> 临时：
>
> ulimit -n 65535 # 设置可打开最大文件数量 
>
> ulimit -u 4096 # 设置可打开最大线程数
>
> 永久：
>
> vi /etc/security/limits.conf
>
> // 添加
>
> elasticsearch  -  nofile  65535
>
> elasticsearch  -  nproc  4096

## 7. RPM包安装目录结构

RPM将配置文件、日志和数据目录放置在基于RPM的系统的适当位置：

| 目录名称 | 描述                                                         | 默认路径                     | 配置项       |
| -------- | ------------------------------------------------------------ | ---------------------------- | ------------ |
| home     | ES安装根目录，记为$ES_HOME                                   | /usr/share/elasticsearch     |              |
| bin      | ES可执行文件目录（启动，插件安装等）                         | $ES_HOME/bin                 |              |
| conf     | ES配置文件存储目录，包含elasticsearch.yml                    | /etc/elasticsearch           | ES_PATH_CONF |
| conf     | 系统环境配置文件存储目录，包含堆大小，文件描述符等           | /etc/sysconfig/elasticsearch |              |
| data     | 数据文件存储路径                                             | /var/lib/elasticsearch       | path.data    |
| jdk      | 自带jdk运行目录，可以使用/etc/sysconfig/elasticsearch中的JAVA_HOME配置覆盖 | $ES_HOME/jdk                 | JAVA_HOME    |
| logs     | 日志存储目录                                                 | /var/log/elasticsearch       | path.logs    |
| plugins  | 插件路径，每一个插件存储一个子目录                           | $ES_HOME/plugins             |              |
| repo     | 公共库路径，可以包含多个路径，文件系统存储库可以放在此处指定的任何目录的任何子目录中。 | 未配置                       | path.repo    |





# 下一步

现在已经设置了一个测试ElasticSearch环境。
在开始认真开发或使用ElasticSearch投入生产之前，必须进行一些附加设置：

* 了解如何配置ElasticSearch。
* 配置重要的ElasticSearch设置。
* 配置重要的系统设置。