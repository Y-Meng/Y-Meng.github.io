# 关闭ES
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/stopping-elasticsearch.html)

Elasticsearch的有序关闭确保了Elasticsearch有机会清理和关闭未完成的资源。
例如，以有序方式关闭的节点将从集群中移除自身，将转换日志同步到磁盘，并执行其他相关的清理活动。
您可以通过正确停止Elasticsearch来帮助确保有序关机。

如果将Elasticsearch作为服务运行，则可以通过安装提供的服务管理功能停止Elasticsearch。

如果直接运行Elasticsearch，则可以通过发送control-C（如果在控制台中运行Elasticsearch）
或将SIGTERM发送到POSIX系统上的Elasticsearch进程来停止Elasticsearch。
您可以通过各种工具（如ps或jps）获取发送信号的PID：
```bash
jps | grep Elasticsearch
14542 Elasticsearch

kill -SIGTERM 14542
```

## 由于致命错误停止

在Elasticsearch虚拟机的生命周期中，可能会出现某些致命错误，使虚拟机处于可疑状态。
这些致命错误包括内存不足错误、虚拟机内部错误和严重的I/O错误。

当Elasticsearch检测到虚拟机遇到此类致命错误时，Elasticsearch将尝试记录该错误，然后将停止虚拟机。
当Elasticsearch启动此类关闭时，它不会按上述顺序进行关闭。
Elasticsearch过程还将返回一个特殊的状态代码，指示错误的性质。