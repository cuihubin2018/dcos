## FAQ

### DCOS集群的DNS域名服务器变更后，如果重新配置DCOS集群？

修改 `/opt/mesosphere/etc/dns_config` （最多只支持[__3个DNS地址__](https://docs.mesosphere.com/1.8/administration/installing/custom/configuration-parameters/#resolvers)）

重启 `dcos-spartan.service`

在所有节点上重复上述步骤







