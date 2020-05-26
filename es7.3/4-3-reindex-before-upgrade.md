# 在升级前重建索引

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/reindex-upgrade.html)

Elasticsearch可以读取以前主要版本中创建的索引。如果在5.x或之前版本中创建了索引，则必须在升级到7.3.2之前重新编制索引或删除索引。如果存在不兼容的索引，则Elasticsearch节点将无法启动。5.x或更早索引的快照无法还原到7.x群集，即使它们是由6.x群集创建的。

这个限制也适用于Kibana和X-Pack特性使用的内部索引。因此，在使用7.3.2中的Kibana和X-Pack功能之前，必须确保内部索引具有兼容的索引结构。

有两个选项可用于重新创建旧索引的索引：

* 升级前在当前6.x群集上重新建立索引。
* 部署一个新的7.x集群，然后从其他集群重建索引。这使能够重新索引位于运行任何版本Elasticsearch的集群上的索引。

> 注意：升级基于时间的索引
>
> 如果你使用基于时间的索引，你可能不需要将6.x之前的索引升级到7.3.2。基于时间的索引中的数据通常会随着时间的推移而变得不那么有用，并且会随着它们在保留期之后的老化而被删除。
>
> 除非您的保留期异常长，否则您可以等待升级到6.x，直到删除所有6.x之前的索引。



## 1.在当前集群重建索引

您可以使用Kibana6.8中的升级助手自动重新索引5.x索引，以便结转到7.3.2。

要手动将旧索引重新索引到位，请执行以下操作：

1. 创建具有7.x兼容性的索引;
2. 将refresh_interval设置为-1，将副本的数量number_of_replicas设置为0，以便高效地重新索引;
3. 使用reindex API将文档从5.x索引复制到新索引中。在重建引期间，可以使用脚本对文档数据和元数据执行任何必要的修改;
4. 根据旧索引上的配置重新设置refresh_interval和number_of_replicas值；
5. 等待索引状态变为green；
6. 在单个更新别名请求中：
   1. 删除旧索引
   2. 增加一个别名配置将旧索引名指向新索引
   3. 将旧索引上存在的所有别名添加到新索引上

> 注意：
>
> 如果使用机器学习功能，并且机器学习索引是在6.x之前创建的，则必须暂时停止与机器学习作业和数据馈送相关联的任务，并防止在重新索引期间打开新作业。使用设置升级模式API或停止所有数据源并关闭所有机器学习作业。
>
> 如果使用Elasticsearch安全功能，在重新编制.security*内部索引之前，最好在文件域中创建一个临时超级用户帐户。
>
> 1. 在单个节点上，将临时超级用户帐户添加到文件领域。例如，运行elasticsearch users useradd命令：
>
>    ```
>    bin/elasticsearch-users useradd <user_name> \
>    -p <password> -r superuser
>    ```
>
> 2. 重新索引.security*索引时使用这些凭据。也就是说，使用它们登录Kibana并运行升级助手或调用reindex API。您可以使用常规管理凭据重新索引其他内部索引。
>
> 3. 从文件域中删除临时超级用户帐户。例如，运行elasticsearch users userdel命令：
>
>    ```
>    bin/elasticsearch-users userdel <user_name>
>    ```



## 2.跨集群重建索引

可以使用remote中的reindex将索引从旧集群迁移到新的7.3.2集群。这使您能够在不中断服务的情况下从6.8以前的群集移动到7.3.2。

> 注意：
>
> Elasticsearch提供向后兼容性支持，支持将以前主要版本的索引升级到当前主要版本。跳过主要版本意味着您必须自己解决任何向后兼容性问题。
>
> 如果使用机器学习功能并从6.5或更早版本的群集迁移索引，则作业和数据馈送配置信息不会存储在索引中。必须在新群集中重新创建机器学习作业。如果要从6.6或更高版本的群集迁移，最好暂时停止与机器学习作业和数据馈送相关的任务，以防止在稍有不同的时间重新编制索引的不同机器学习索引之间的不一致。使用设置升级模式API或停止所有数据源并关闭所有机器学习作业。

要迁移索引，请执行以下操作：

1. 设置一个新的7.3.2集群，并将现有集群添加到elasticsearch.yml中的reindex.remote.whitelist

   ```shell
   reindex.remote.whitelist: oldhost:9200
   ```

   新集群不必开始完全扩展。在迁移索引并将负载转移到新集群时，可以将节点添加到新集群，并从旧集群中移除节点。

2. 对于需要迁移到新群集的每个索引：

   1. 创建相应映射和设置的索引。将refresh_interval设置为-1，并将number_of_replicas设置为0，以便更快地重新编制索引；

   2. 使用reindex API将文档从远程索引拉入新的7.3.2索引：

      ```shell
      POST _reindex
      {
        "source": {
          "remote": {
            "host": "http://oldhost:9200",
            "username": "user",
            "password": "pass"
          },
          "index": "source",
          "query": {
            "match": {
              "test": "data"
            }
          }
        },
        "dest": {
          "index": "dest"
        }
      }
      ```

      如果通过将wait_for_completion设置为false在后台运行reindex作业，reindex请求将返回一个task_id，您可以使用task API:GET_tasks/task_id来监视reindex作业的进度；

   3. 重新索引作业完成后，将refresh_interval和number_of_replicas设置为所需的值（默认设置为30s和1）；

   4. 一旦重新编制索引完成并且新索引的状态为绿色，就可以删除旧索引。