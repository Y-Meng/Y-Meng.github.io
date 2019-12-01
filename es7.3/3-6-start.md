# 启动Elasticsearch
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/starting-elasticsearch.html)

启动ES的方式取决于你安装的方式。

## 1.tar.gz存档包（Archive package *.tar.gz）

如果你通过*.tar.gz存档包安装ES，可以通过命令行来启动。

```bash
./bin/elasticsearch
```

默认ES在前台运行，打印日志到标准输出（stdout），可以通过ctrl-c停止。

### 作为守护进程运行
```bash
./bin/elasticsearch -d -p pid
```
日志信息会存放在$ES_HOME/logs/目录。
关闭ES，通过杀死pid文件的进程ID
```bash
pkill -F pid
```

注：
> RPM和Debian包中提供启动脚本负责启动和停止Elasticsearch进程。

## 2.zip存档包（Archive package *.zip）

如果在windows系统上通过zip包安装，可以通过下面命令行启动：
```batch
.\bin\elasticsearch.bat
```
默认运行在前台，可以通过ctrl-c停止退出

## 3.Debian package

Elasticsearch在安装后不会自动启动。如何启动和停止Elasticsearch取决于您的系统是使用SysV init还是systemd（由较新的发行版使用）。
通过运行此命令，您可以知道正在使用哪个：
```bash
ps -p 1
```

### 通过SysV init
使用update-rc.d命令配置es在系统启动时自动启动
```bash
sudo update-rc.d elasticsearch defaults 95 10
```
Es可以通过service命令启动和停止
```bash
sudo -i service elasticsearch start
sudo -i service elasticsearch stop
```
如果ES由于任何原因启动失败会将失败原因输出到标准输出stdout，日志文件存储在/var/log/elasticsearch

### 通过systemd
配置ES开机自启
```bash
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl enable elasticsearch.service
```
通过systemctl启动和停止es
```bash
sudo systemctl start elasticsearch.service
sudo systemctl stop elasticsearch.service
```
日志存储在/var/log/elasticsearch

默认情况下，Elasticsearch服务不会在systemd日志中记录信息。
要启用journalctl日志记录，必须从elasticsearch.service文件的ExecStart命令行中删除--quiet选项。

启用systemd日志记录时，可以使用journalctl命令获得日志记录信息：
```bash
sudo jouralctl -f
```
列出es的journal日志条目
```bash
sudo journalctl --unit elasticsearch
```
要列出elasticsearch服务从给定时间开始的日记条目，请执行以下操作：
```bash
sudo journalctl --unit elasticsearch --since  "2016-10-30 18:17:16"
```

## 4.docker镜像

如果安装了Docker镜像，可以从命令行启动Elasticsearch。
根据您使用的是开发模式还是生产模式，有不同的方法。请参见从命令行运行Elasticsearch。

## 5.MSI安装包

如果使用.msi包在Windows上安装了Elasticsearch，则可以从命令行启动Elasticsearch。
如果希望它在启动时自动启动，而不需要任何用户交互，请将Elasticsearch安装为Windows服务。

安装后，如果不是作为服务安装并配置为在安装完成时启动，则可以从命令行启动Elasticsearch，如下所示：
```bash
.\bin\elasticsearch.exe
```

## 6.rpm packages

同 debian包