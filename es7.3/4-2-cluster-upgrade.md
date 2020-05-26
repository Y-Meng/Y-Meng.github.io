# 集群重启升级

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/restart-upgrade.html)

要从版本6.0-6.7直接升级到Elasticsearch 7.3.2，必须关闭集群中的所有节点，将每个节点升级到7.3.2，然后重新启动集群。

如果运行的是6.0之前的版本，请升级到6.8并重新索引旧索引，或启动新的7.3.2群集并从远程重新索引。

# 1.升级准备
在开始升级之前仔细准备是很重要的。一旦开始将群集升级到7.3.2版，就必须完成升级。
一旦群集包含7.3.2版的节点，它可能会对其内部状态进行无法还原的更改。
如果无法完成升级，则应放弃部分升级的群集，在升级之前部署版本的空群集，并从快照还原其内容。

升级之前需要做如下准备：
1. 检查弃用日志，查看是否正在使用任何弃用的功能，并相应地更新代码。
2. 检查中断更改，并对7.3.2版的代码和配置进行必要的更改。
3. 如果您使用任何插件，请确保每个插件都有与Elasticsearch 7.3.2版兼容的版本。
4. 在升级生产群集之前，请在隔离环境中测试升级。
5. 通过快照备份数据！！！

# 2.升级集群

## 2.1.关闭分片分配

当你关闭一个节点时，分配进程将等待index.unassigned.node_left.delayed_timeout（默认为一分钟），然后开始将该节点上的分片复制到群集中的其他节点，这可能会涉及大量I/O。

由于节点很快就要重新启动，因此不需要此I/O。通过在关闭节点之前禁用副本的分配，可以避免分片重新分配：

```shell
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": "primaries"
  }
}
```

## 2.2.停止索引并执行同步刷新

执行一次同步刷新可加快重启时分片恢复。

```shell
POST _flush/synced
```

执行同步刷新时，请检查响应以确保没有失败。响应正文中列出了由于挂起的索引操作而失败的同步刷新操作，尽管请求本身仍然返回200 OK状态。如果失败，请重新发出请求。

## 2.3.暂时停止机器学习任务及数据源关联任务（可选）

如果你的机器学习索引是6.x版本前创建的，你必须重建这些索引。

如果你的机器学习索引是基于6.x版本创建的，你可以进行的操作：

* 让您的机器学习作业在升级期间运行。关闭机器学习节点时，其作业会自动移动到另一个节点并还原模型状态。此选项允许您的作业在升级期间继续运行，但会增加群集的负载。
* 暂时停止与机器学习作业和数据源关联的任务，并使用设置升级模式API阻止打开新作业：

```
POST _ml/set_upgrade_mode?enabled=true
```

禁用升级模式时，作业将使用自动保存的最后一个模型状态继续。此选项避免了在升级期间管理活动作业的开销，并且比显式停止数据馈送和关闭作业更快。

* 停止所有数据源并关闭所有作业。此选项保存关闭时的模型状态。升级后重新打开作业时，它们使用完全相同的模型。但是，保存最新的模型状态比使用升级模式需要更长的时间，特别是当您有很多作业或具有较大模型状态的作业时。

## 2.4.依次关闭所有节点

* 使用systemd运行的ES

  ```shell
  sudo systemctl stop elasticsearch.service
  ```

* 使用SysV init运行的ES

  ```shell
  sudo -i service elasticsearch stop
  ```

* 使用后台进程启动的ES

  ```shell
  kill $(cat pid)
  ```

  

## 2.5.升级所有节点

> 如果要从6.2或更早版本升级并使用X-Pack，请在升级之前运行**bin/elasticsearch plugin remove x-pack** 以删除X-Pack插件。X-Pack功能现在包含在默认发行版中，不再单独安装。如果X-Pack插件存在，则升级后节点不会启动。您需要降级，删除插件，然后重新应用升级。

* 使用Debian或rpm包升级：
  * 使用rpm或dpkg安装新的包，所有文件都安装在操作系统的适当位置，并且不会覆盖Elasticsearch配置文件。
* 通过zip或tar包升级：
  * 解压zip或tar包到一个新目录，如果不使用外部配置(config)和数据(data)目录，这一点非常重要。
  * 设置ES_PATH_CONF环境变量以指定外部配置目录和jvm.options文件的位置。如果不使用外部配置目录，请将旧配置复制到新安装目录。
  * 在config/elasticsearch.yml中设置path.data以指向外部数据目录。如果不使用外部数据目录，请将旧data目录复制到新安装目录。
  * 在config/elasticsearch.yml中设置path.logs以指向外部配置目录。如果不使用外部配置目录，请将旧logs目录复制到新安装目录

> 重要：如果使用了监视功能，请在升级Elasticsearch时重新使用数据目录。监视通过使用存储在数据目录中的持久UUID来标识唯一的Elasticsearch节点。

> 提取zip或tar包时，elasticsearch-n.n.n目录包含elasticsearch config、data和logs目录。
>
> 我们建议将这些目录移出Elasticsearch目录，以便在升级Elasticsearch时不会删除它们。要指定新位置，请使用ES _PATH _CONF环境变量和path.data和path.logs设置。有关更多信息，请参阅重要的Elasticsearch配置。
>
> Debian和RPM包将这些目录放在每个操作系统的适当位置。在生产中，我们建议使用deb或rpm包安装。

如果从6.x群集升级，还必须通过在符合主节点条件的节点上设置cluster.initial_master_nodes设置来配置群集引导。

1. **升级所有插件**：使用elasticsearch插件脚本安装每个已安装的elasticsearch插件的升级版本。升级节点时必须升级所有插件。

2. 如果使用Elasticsearch安全功能定义域，请验证您的域设置是最新的。域设置的格式在7.0版中发生了更改，特别是域类型的位置发生了更改。请参见[域设置](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/security-settings.html#realm-settings)。

3. **启动每个升级节点**：如果有专用的主节点，请先启动它们，等待它们形成群集并选择主节点，然后再继续处理数据节点。你可以通过查看日志来检查进度。只要有足够多的符合主节点条件的节点发现彼此，它们就会形成一个集群并选择一个主节点。此时，可以使用“\_cat/health”和“\_cat/nodes”监视加入群集的节点：

4. **等待所以节点加入集群并报告健康状态未yellow**：当一个节点加入集群时，它开始恢复本地存储的所有主分片。_cat/health API最初报告的状态为红色，表示尚未分配所有主分片。一旦节点恢复其本地分片，群集状态将切换为黄色，表示所有主分片都已恢复，但并非所有副本分片都已分配。这是意料之中的，因为您尚未重新启用分配。将副本的分配延迟到所有节点都为黄色时，允许主节点将副本分配给已经具有本地分片副本的节点。

5. **重新启用分片分配**：当所有节点已加入群集并恢复其主分片时，通过将cluster.routing.allocation.enable还原为其默认值可重新启用分配：

   ```
   PUT _cluster/settings
   {
     "persistent": {
       "cluster.routing.allocation.enable": null
     }
   }
   ```

   重新启用分配后，群集将开始向数据节点分配副本分片。此时，恢复索引和搜索是安全的，但是如果您可以等到所有主分片和副本分片都已成功分配并且所有节点的状态为绿色，则集群将更快地恢复。

6. **重启机器学习任务**：如果暂时停止与机器学习作业关联的任务，请使用设置升级模式API将其返回到活动状态：

   ```
   POST _ml/set_upgrade_mode?enabled=false
   ```

   如果在升级之前关闭了所有机器学习作业，请打开作业并从Kibana启动数据源，或使用[open jobs](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/ml-open-job.html)和[start datafeed](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/ml-start-datafeed.html) API。