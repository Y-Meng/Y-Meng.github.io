# xpack 引导检查
[原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.3/bootstrap-checks-xpack.html)

## 加密敏感数据检查（encrypt sensitive data check）

如果使用Watcher并已选择加密敏感数据（通过将xpack.watcher.encrypt_sensitive_data设置为true），则还必须在安全设置存储中放置密钥。

要通过此引导检查，必须在群集中的每个节点上设置xpack.watcher.encryption_key。有关详细信息，请参阅Watcher中的加密敏感数据。

## PKI域检查（PKI realm check）

如果使用Elasticsearch安全功能和公钥基础结构（PKI）域，则必须在群集上配置传输层安全性（TLS），并在网络层（传输层或http）上启用客户端身份验证。
有关详细信息，请参阅PKI用户身份验证和在群集上设置TLS。

若要通过此引导检查，如果启用了PKI领域，则必须在至少一个网络通信层上配置TLS并启用客户端身份验证。

## 角色映射检查（role mapping check）

如果使用本机域或文件域以外的域对用户进行身份验证，则必须创建角色映射。这些角色映射定义分配给每个用户的角色。

如果使用文件管理角色映射，则必须配置一个YAML文件并将其复制到群集中的每个节点。
默认情况下，角色映射存储在ES_PATH_CONF/role_mapping.yml中。
或者，可以为每种领域类型指定不同的角色映射文件，并在elasticsearch.yml文件中指定其位置。
有关详细信息，请参见使用角色映射文件。

要通过这个引导检查，角色映射文件必须存在并且必须是有效的。角色映射文件中列出的可分辨名称（DNs）也必须有效。

## SSL/TLS检查（SSL/TLS check）
如果启用Elasticsearch安全功能，则除非您有试用许可证，否则必须为节点间通信配置SSL/TLS。（单节点配置环回地址无需此配置）。

要通过此引导检查，必须在集群中设置SSL/TLS。

## Token SSL 检查（Token SSL check）
如果使用Elasticsearch安全功能并且启用了内置令牌服务，则必须将群集配置为对HTTP接口使用SSL/TLS。使用令牌服务需要使用HTTPS。

特别是，如果在elasticsearch.yml文件中将xpack.security.authc.token.enabled设置为true，则还必须将xpack.security.http.ssl.enabled设置为true。
有关这些设置的详细信息，请参阅安全设置和HTTP。
若要通过此引导检查，必须启用HTTPS或禁用内置令牌服务。