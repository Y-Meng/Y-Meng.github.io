[原文](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup.html)

# 安装部署

本节包括有关如何设置ElasticSearch并使其运行的信息，包括：

* 下载
* 安装
* 启动
* 配置 

# 支持的平台

官方支持的操作系统和JVM的矩阵可以在这里找到：[支持矩阵](https://www.elastic.co/support/matrix)。
ElasticSearch在列出的平台上进行了测试，但也有可能在其他平台上工作。

# Java(JVM)版本

ElasticSearch基于开发，并在每个发布包中包含OpenJDK的捆绑包。
捆绑的JVM是推荐的JVM，位于ElasticSearch主目录的JDK目录中。

要使用自己的Java版本，设置Java的JAVA_HOME环境变量。
如果必须使用与捆绑JVM不同的Java版本，我们建议使用支持的LTS版本的Java。
如果使用已知的不支持版本Java，弹性搜索将拒绝启动。
使用自己的JVM时，捆绑的JVM目录可能会被删除。 
