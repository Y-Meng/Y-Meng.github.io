[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/bootstrap-checks.html)

# 启动引导检查（bootstrap checks）

总的来说，有很多用户都有过遇到意外问题的经历，因为他们没有配置重要的设置。
在以前版本的Elasticsearch中，对其中一些设置的错误配置被记录为警告。
可以理解，用户有时会错过这些日志消息。
为了确保这些设置得到应有的注意，Elasticsearch在启动时会进行引导检查。

这些引导检查检查各种Elasticsearch和系统设置，并将它们与对Elasticsearch的操作安全的值进行比较。
如果Elasticsearch处于开发模式，则任何失败的引导检查都会在Elasticsearch日志中显示为警告。
如果Elasticsearch处于生产模式，任何失败的引导检查都将导致Elasticsearch拒绝启动。

有些引导检查总是强制执行，以防止Elasticsearch在不兼容的设置下运行。这些检查是单独记录的。

## 开发模式 VS 生产模式

默认情况下，Elasticsearch绑定到用于HTTP和transport（内部）通信的环回地址（127.0.0.1）。
这对于下载和试用Elasticsearch以及日常开发来说都是很好的，但是对于生产系统来说是没有用的。
要加入群集，Elasticsearch节点必须可以通过transport通信访问。
要通过非环回地址加入群集，节点必须将传输绑定到非环回地址，并且不使用单节点发现。
因此，如果一个Elasticsearch节点不能通过非环回地址与另一台机器形成集群，我们认为它处于开发模式；
如果它能够通过非环回地址加入集群，我们认为它处于生产模式。

注意：
> 可以通过hppt.host和transport.host独立配置HTTP和transport；这对于在不触发生产模式的情况下将单个节点配置为可通过HTTP访问以进行测试非常有用。

## 单节点发现
我们认识到，有些用户需要将transport绑定到外部接口，以测试他们对传输客户端（transport client）的使用情况。
对于这种情况，我们提供discovery type single node（通过将discovery.type设置为single-node进行配置）；
在这种情况下，节点将选择自己为主节点，而不会将集群与任何其他节点联接。

## 强制引导检查
如果在生产环境中运行单个节点，则可以逃避引导检查（不将传输绑定到外部接口，或将传输绑定到外部接口并将发现类型设置为单个节点）。
在这种情况下，可以通过将系统属性es.enforce.bootstrap.checks设置为true；
或通过在设置JVM选项将-Des.enforce.bootstrap.checks=true添加到环境变量es_JAVA_OPTS中强制执行引导检查。
如果你处于这种特殊情况，我们强烈鼓励你这样做。此系统属性可用于强制执行与节点配置无关的引导检查。

# 启动引导检查项

## 1.堆大小检查（Heap size check）
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/_heap_size_check.html)

如果JVM以不相等的初始堆大小和最大堆大小启动，则在系统使用期间，由于JVM堆的大小调整，它可能容易引起JVM暂停。
为了避免这些调整大小的暂停，最好使用初始堆大小等于最大堆大小的JVM启动。
此外，如果启用bootstrap.memory_lock，JVM将在启动时锁定堆的初始大小。
如果初始堆大小不等于最大堆大小，则在调整大小之后，不会将所有JVM堆锁定在内存中。
要通过堆大小检查，必须配置堆大小。


## 2.文件描述符检查（File descriptor check）
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/_file_descriptor_check.html)

文件描述符（file descriptor）是用于跟踪打开的"文件(file)"的Unix结构。
不过，在Unix中，一切皆文件。例如，“文件”可以是物理文件、虚拟文件（例如/proc/loadavg）或网络套接字（socket）。
Elasticsearch需要大量的文件描述符（例如，每个shard由多个片段和其他文件组成，加上与其他节点的连接等）。
此引导检查在OS X和Linux上强制执行。要通过文件描述符检查，可能需要配置文件描述符。

## 3.内存锁定检查（memory lock check）
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/_memory_lock_check.html)

当JVM进行主要的垃圾收集时，它会涉及堆的每个页面。
如果这些页面中的任何一个被交换（swap）到磁盘上，它们将不得不被交换回内存中。
这会消耗大量的磁盘性能，Elasticsearch更希望这些性能来服务于处理请求。
有几种方法可以将系统配置为不允许交换。
一种方法是通过mlockall（Unix）或virtual lock（Windows）请求JVM将堆锁定在内存中。
这是通过Elasticsearch设置bootstrap.memory_lock完成的。
但是，在某些情况下，此设置可以传递给Elasticsearch，但Elasticsearch无法锁定堆
（例如，如果Elasticsearch用户没有memlock unlimited）。
内存锁定检查验证如果bootstrap.memory_lock设置已启用，则JVM是否能够成功锁定堆。
要通过内存锁检查，可能需要配置bootstrap.memory_lock。

## 4.最大线程数检查（maximum number of threads check）
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/max-number-threads-check.html)
Elasticsearch通过将请求分解为阶段并将这些阶段传递给不同的线程池执行器来执行请求。
Elasticsearch中有不同的线程池执行器，用于执行各种任务。
因此，Elasticsearch需要创建大量线程的能力。
检查线程的最大数量确保了弹性搜索过程有权在正常使用下创建足够的线程。
此检查仅在Linux上强制执行。如果你在Linux上，通过最大数量的线程检查，你必须配置你的系统以允许弹性搜索过程创建至少4096个线程的能力。
这可以通过/etc/security/limits.conf使用nproc设置来完成（注意，您可能还必须增加root用户的限制）。

## 5.最大文件大小检查（max file size 检查）
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/_max_file_size_check.html)
作为单个碎片组件的段文件和作为translog组件的translog代可能会变大（超过数G）。
在Elasticsearch进程可以创建的最大文件大小受到限制的系统上，这可能导致写入失败。
因此，这里最安全的选项是最大文件大小是不受限制的，这就是最大文件大小引导检查所执行的操作。
要通过最大文件检查，必须将系统配置为允许Elasticsearch进程能够写入大小不受限制的文件。
这可以通过/etc/security/limits.conf完成，使用fsize设置为unlimited（注意，您可能还必须增加root用户的限制）。


## 6.最大虚拟内存容量检查（maximum size virtual memory check）
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/max-size-virtual-memory-check.html)

Elasticsearch和Lucene使用mmap非常有效地将索引的部分映射到Elasticsearch地址空间。
这使得某些索引数据不在JVM堆中，而在内存中进行快速访问。
为了有效，Elasticsearch应该有无限的地址空间。
最大大小的虚拟内存检查强制了Elasticsearch搜索过程具有无限的地址空间，只在Linux上执行。
要传递最大大小的虚拟内存检查，必须配置系统以允许搜索过程具有无限的地址空间的能力。
这可以通过/etc/security/limits.conf使用as设置来完成，设置为unlimited（注意，您可能还需要增加root用户的限制）。

## 7.最大映射数检查（maximum map count check）
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/_maximum_map_count_check.html)

从上一点继续延伸，为了有效地使用mmap，Elasticsearch还需要创建许多内存映射区域的能力。
最大映射计数检查检查内核允许进程至少有262144个内存映射区域，并且仅在Linux上执行。
要通过最大的地图计数检查，您必须通过sysctl配置vm.max_map_count至少为262144。

另外，如果使用MMAPFS或HybridFS作为索引的存储类型，则只需要最大的MAP计数检查。
如果不允许使用mmap，则不会强制执行此引导检查。

## 8.客户机JVM检查（Client JVM check）
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/_client_jvm_check.html)

OpenJDK派生的JVM提供了两种不同的JVM：客户机JVM（client jvm）和服务器JVM（server jvm）。
这些jvm使用不同的编译器从Java字节码生成可执行的机器代码。
客户机JVM在启动服务器和JVM以最大化性能的同时调整启动时间和内存占用。
两个vm之间的性能差异可能很大。客户端JVM检查确保Elasticsearch不在客户端JVM中运行。
要通过客户机JVM检查，必须使用服务器VM启动Elasticsearch。在现代系统和操作系统中，服务器VM是默认的。


## 9.使用串行收集器检查（use serial collector check）
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/_use_serial_collector_check.html)

针对不同的工作负载，OpenJDK派生的jvm有各种垃圾收集器。
串行收集器尤其适合于单逻辑CPU机器或非常小的堆，这两种机器都不适合运行Elasticsearch。
将串行收集器与Elasticsearch结合使用可能会对性能造成破坏。
串行收集器检查确保Elasticsearch未配置为与串行收集器一起运行。
要通过串行收集器检查，不能使用串行收集器启动Elasticsearch（无论是使用的JVM的默认值，还是使用-XX:+UseSerialGC显式指定的值）。
注意，Elasticsearch附带的默认JVM配置将Elasticsearch配置为使用CMS收集器。

## 10.系统调用过滤器检查（system call filter check）
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/_system_call_filter_check.html)

Elasticsearch根据操作系统（例如，Linux上的seccomp）安装不同风格的系统调用过滤器。
安装这些系统调用过滤器是为了防止执行与forking相关的系统调用，以此作为对Elasticsearch上任意代码执行攻击的防御机制。
系统调用筛选器检查确保如果启用了系统调用筛选器，则它们已成功安装。
若要通过系统调用筛选器检查，必须修复系统上阻止安装系统调用筛选器的任何配置错误（检查日志），
或者通过将bootstrap.system_call_filter设置为false禁用系统调用筛选器，风险自负。

## 11.错误或内存溢出错误检查（OnError or OnOutOfMemoryError checks）
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/_onerror_and_onoutofmemoryerror_checks.html)
如果JVM遇到致命错误（OnError）或OutOfMemoryError，则JVM的OnError和OnOutOfMemoryError选项可以执行任意命令。
但是，默认情况下，Elasticsearch系统调用过滤器（seccomp）已启用，并且这些过滤器防止分叉。
因此，使用OnError或OnOutOfMemoryError和系统调用过滤器是不兼容的。
如果使用了这些JVM选项之一并且启用了系统调用筛选器，则OnError和OnOutOfMemoryError检查将阻止Elasticsearch启动。
这项检查总是有效的。若要通过此检查，请不要启用OnError或OnOutOfMeMyCyror错误。
相反，升级到Java8U92，并使用JVM标志ExitOutOfMemoryError。
虽然这不具备OnError或OnOutOfMemoryError的全部功能，但启用seccomp时将不支持任意分叉。

## 12.早期访问检查（early-access check）
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/_early_access_check.html)
OpenJDK项目提供了即将发布的版本的早期访问快照。这些版本不适合生产。
早期访问检查检测这些早期访问快照。要通过此检查，必须在JVM的发布版本上启动Elasticsearch。

## 13.G1回收器检查（G1GC check）
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/_g1gc_check.html)

众所周知，JDK 8附带的HotSpot JVM的早期版本在启用G1GC收集器时存在可能导致索引损坏的问题。
受影响的版本是那些早于JDK 8u40附带的HotSpot版本的版本。G1GC检查检测这些早期版本的热点JVM。


## 14.全部权限检查（all permission check）
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/_all_permission_check.html)

所有权限检查可确保引导期间使用的安全策略不会将java.security.all permission授予Elasticsearch。
在授予所有权限的情况下运行等同于禁用安全管理器。

## 15.节点发现配置检查（discovery configuration check）
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/_discovery_configuration_check.html)

默认情况下，当Elasticsearch首次启动时，它将尝试发现运行在同一主机上的其他节点。
如果在几秒钟内找不到选定的主节点，则Elasticsearch将形成一个集群，其中包括已发现的任何其他节点。
在开发模式下可以在不进行任何额外配置的情况下形成此群集是很有用的，但这不适合生产，因为这样可能会形成多个群集并因此丢失数据。

此引导检查可确保发现未使用默认配置运行。可以通过设置以下至少一个属性来满足：

* discovery.seed_hosts
* discovery.seed_providers
* cluster.initial_master_nodes