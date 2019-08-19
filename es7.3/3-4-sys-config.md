# 重要的系统设置

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/system-config.html)

理想情况下，ElasticSearch应该单独在服务器上运行，并使用所有可用的资源。为此，需要将操作系统配置为允许运行ElasticSearch的用户访问超过默认允许的资源。

在开始生产环境使用之前，必须考虑以下设置：

* 禁用交换区
* 增加文件描述符
* 确保足够的内存
* 确保有足够的线程
* JVM DNS缓存设置
* 未使用NoExec装载临时目录 

## 开发模式和生产模式

默认情况下，ElasticSearch假定您正在开发模式下工作。如果上述任何设置配置不正确，将向日志文件写入警告，但能够启动并运行ElasticSearch节点。

一旦您配置了network.host等网络设置，elasticsearch就会假定您正在进入生产环境，并将上述警告升级为异常。
这些异常将阻止ElasticSearch节点启动。这是一个重要的安全措施，可以确保不会因为服务器配置错误而丢失数据。

## 系统设置

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/setting-system-settings.html)

配置系统设置的位置取决于安装ElasticSearch所使用的软件包以及所使用的操作系统。
使用.zip或.tar.gz包时，可以配置系统设置：
* 临时使用ulimit命令
* 永久修改/etc/security/limits.conf

当使用RPM或Debian包时，大多数系统设置都在系统配置文件中设置。但是，使用systemd的系统要求在systemd配置文件中指定系统限制。

### ulimit命令

在Linux系统上，ulimit可用于临时更改资源限制。在切换到运行ElasticSearch的用户之前，通常需要将限制设置为root。
例如，要将打开的文件句柄数（ulimit-n）设置为65536，可以执行以下操作：
```bash
sudo su  # 切换root权限
ulimit -n 65535 # 设置可打开最大文件数量 
su elasticsearch  # 切换为ES用户
```

新限制仅在当前会话期间应用。
您可以向ulimit-a查询当前应用的所有限制。

### /etc/security/limits.conf

在Linux系统上，可以通过编辑/etc/security/limits.conf文件为特定用户设置持久限制。
要将ElasticSearch用户的最大打开文件数设置为65536，请在limits.conf文件中添加以下行： 
```bash
elasticsearch  -  nofile  65535
```

此更改仅在下次ElasticSearch用户打开新会话时生效。

> Ubuntu忽略init.d启动的进程的limits.conf文件。要启用limits.conf文件，请编辑/etc/pam.d/su并取消对以下行的注释：# session    required   pam_limits.so

### Sysconfig配置文件【rpm和debian】

使用RPM或Debian包时，可以在系统配置文件中指定系统设置和环境变量，该文件位于：
```bash
# RPM
/etc/sysconfig/elasticsearch

# Debian
/etc/default/elasticsearch
```

但是，对于使用systemd的系统，需要通过systemd指定系统限制。

### systemd 配置

在使用systemd的系统上使用rpm或debian包时，必须通过systemd指定系统限制。

systemd服务文件（/usr/lib/systemd/system/elasticsearch.service）包含默认应用的限制。

要覆盖它们，请添加一个名为/etc/systemd/system/elasticsearch.service.d/override.conf的文件
（或者，可以运行sudo systemctl edit elasticsearch，在默认编辑器中自动打开文件）。设置此文件中的任何更改，例如：
```bash
[Service]
LimitMEMLOCK=infinity
```
完成后，运行以下命令重新加载：
```bash
sudo systemctl daemon-reload
```

## 禁用交换区

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-configuration-memory.html)

大多数操作系统尝试尽可能多地将内存用于文件系统缓存，并尽量交换未使用的应用程序内存。
这可能导致部分JVM堆甚至其可执行页面被交换到磁盘。

交换（swapping）对性能、节点稳定性都非常不利，应不惜一切代价避免交换。
它可能会导致垃圾收集持续几分钟而不是几毫秒，并可能导致节点响应缓慢，甚至断开与集群的连接。
在弹性分布式系统中，让操作系统杀死节点更有效。

禁用交换有三种方法。首选选项是完全禁用交换。如果不选择这样，那么是否希望最小化交换区与内存锁定取决于您的环境。

### 禁用所有交换区

通常，ElasticSearch是在一个服务器上运行的唯一服务，它的内存使用由JVM选项控制。不需要启用交换区。

在Linux系统上，可以通过运行以下命令临时禁用交换区：
```bash
sudo swapoff -a
```

这个设置不需要重新启动ElasticSearch。 

要永久禁用它，您需要编辑/etc/fstab文件并注释掉包含单词swap的任何行。
 
在Windows上，完全通过系统属性→高级→性能→高级→虚拟内存禁用分页文件，可以实现等效功能。 

### 配置swappiness

Linux系统上可用的另一个选项是确保sysctl值vm.swappiness设置为1。
这减少了内核交换的倾向，在正常情况下不应该导致交换，同时仍然允许整个系统在紧急情况下交换。

### 启用bootstrap.memory_lock

另一种选择是在Linux/Unix系统上使用mlockall，或者在Windows上使用virtualllock，尝试将进程地址空间锁定到RAM中，防止任何ElasticSearch内存被交换出去。这可以通过将此行添加到config/elasticsearch.yml文件来完成：
```bash
bootstrap.memory_lock: true
```

> 如果试图分配的内存超过可用内存，mlockall可能会导致JVM或shell会话退出！

启动ElasticSearch后，通过检查此请求的输出中的mlockall值，可以查看是否成功应用了此设置：
```bash
GET _nodes?filter_path=**.mlockall
```
如果看到mlockall为false，则表示mlockall请求失败。您还将看到日志中包含更多信息的一行，其中包含“Unable to lock JVM Memory”等信息。

在Linux/Unix系统上，最可能的原因是运行ElasticSearch的用户没有锁定内存的权限。这可授予如下：

* .zip和.tar.gz
在启动elasticsearch之前，使用root权限运行ulimit-l unlimited，或者在/etc/security/limits.conf中将memlock设置为unlimited。

* RPM和Debian
在系统配置文件中将max_locked_memory设置为unlimited（无限制）（对于使用systemd的系统，请参阅下面的内容）。

* systemd
在systemd配置中将LimitMEMLOCK 设置为infinity。 

mlockall失败的另一个可能原因是JNA临时目录（通常是/tmp的子目录）使用noexec选项装载。
这可以通过使用ES_JAVA_OPTS 环境变量为JNA指定一个新的临时目录来解决：
```bash
export ES_JAVA_OPTS="$ES_JAVA_OPTS -Djna.tmpdir=<path>"
./bin/elasticsearch
```

或者在jvm.options配置文件中设置这个jvm标志。

## 文件描述符

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/file-descriptors.html)

> 这只与Linux和MacOS相关，如果在Windows上运行ElasticSearch，则可以安全地忽略它。在Windows上，JVM使用仅受可用资源限制的API。

ElasticSearch使用许多文件描述符或文件句柄。文件描述符用完可能是灾难性的，而且很可能会导致数据丢失。
确保将运行ElasticSearch的用户的打开文件描述符的数量限制增加到65536或更高。

对于.zip和.tar.gz包，请在启动elasticsearch之前使用root权限执行ulimit-n 65535，或者在/etc/security/limits.conf中将nofile设置为65535。

## 虚拟内存

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/vm-max-map-count.html)

ElasticSearch默认使用mmapfs目录存储其索引。mmap计数的默认操作系统限制可能太低，这可能导致内存不足异常。

在Linux上，可以通过以root用户身份运行以下命令来增加限制：
```bash
sysctl -w vm.max_map_count=262144
```
要永久设置此值，请更新/etc/sysctl.conf中的vm.max_map_count设置。
要在重新启动后验证，请运行sysctl vm.max_map_count。

RPM和Debian软件包将自动配置此设置。无需进一步配置。

## 线程数量

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/max-number-of-threads.html)

ElasticSearch为不同类型的操作使用许多线程池。
重要的是，它能够在需要时创建新的线程。确保ElasticSearch用户可以创建的线程数至少为4096个。

这可以通过在启动elasticsearch之前使用root用户执行ulimit-u 4096完成，
或者通过在/etc/security/limits.conf中将nproc设置为4096来完成。

当在systemd下作为服务运行时，包分发将自动为elasticsearch进程配置线程数。不需要其他配置。

## DNS缓存设置

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/networkaddress-cache-ttl.html)

ElasticSearch运行时有一个安全管理器。有了安全管理器，jvm默认无限期地缓存正的主机名解析，默认情况下缓存负的主机名解析10秒。
ElasticSearch使用默认值覆盖此行为，以将正查找缓存60秒，并将负查找缓存10秒。这些值应该适用于大多数环境，包括DNS解析随时间变化的环境。
如果没有，你可以编辑jvm选项中的值es.networkaddress.cache.ttl和es.networkaddress.cache.negative.ttl。
注意，除非删除es.networkaddress.cache.ttl和es.networkaddress.cache.negative.ttl的设置，
否则Java弹性安全策略中的networkaddress.cache.ttl＝<timeout>和networkaddress.cache.negative.ttl=<timeout>将被忽略。 

## 未使用NoExec装载临时目录

[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/executable-jna-tmpdir.html)

> 这之和linux系统有关

ES使用Java原生访问（Java Native Access,JNA）库执行一些依赖于平台的本机代码。
在Linux上，支持此库的本机代码在运行时从JNA存档中提取。
默认情况下，此代码被提取到elasticsearch临时目录，该目录默认为/tmp子目录。
或者，可以使用jvm标志-Djna.tmpdir=<path>控制此位置。
当本机库作为可执行文件映射到JVM虚拟地址空间时，提取此代码的位置的基础挂载点不能使用noexec进行加载，因为这会阻止JVM进程将此代码映射为可执行文件。
在一些强化的Linux安装中，这是/tmp的默认安装选项。一个指示底层装载是用noexec装载的是，在启动时，jna将无法用，跑出java.lang.UnsatisfiedLinkerError异常，
并显示“failed to map segment from shared object”消息。请注意，异常消息在JVM版本之间可能有所不同。
此外，依赖于通过JNA执行本机代码的ElasticSearch组件将失败，并显示消息表明这是因为JNA不可用。
如果您看到此类错误消息，则必须重新装载用于JNA的临时目录，不使用NOEXEC装载。