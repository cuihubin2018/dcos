## 环境搭建

DC/OS当前支持本机、云、自定义云和内部集群部署。在本机部署DC/OS是通过Vagrant在本机构建的一个虚拟机集群，它要求至少16GB内存空间；DC/OS为AWS，Azure，Packet和Google Compute Engine提供了适配这些平台的可配置的云配置模板；DC/OS支持在AWS或任何云中为现有配置自定义安装程序；DC/OS为内部集群部署提供了GUI，命令行和高级安装三种模式，且要求至少有4个节点，每个节点2CPU和4GB内存。

**注意，**在规划DC/OS集群时，第一个要考虑是DC/OS服务的排他性。DC/OS内部服务及在DC/OS集群中部署的应用服务会固定或随机占用大量的端口，因此要确保集群主机操作系统的“清洁”，不要与其他外部应用服务并存。

同时，当前版本（1.8.x）的DC/OS在部署集群后，集群的Master节点是不能像Agent节点那样增加的，如果需要变更Master节点数必须重装整个集群。因此规划DC/OS集群时，另一个要考虑的问题是DC/OS的Master节点数。如果本机部署仅用于体验，可设置1个Master节点，在测试环境用集群和线上环境集群推荐至少3个Master节点，最多5个。DC/OS集群的Master节点数并不是越多越好。

第三个注意点是在构建内部集群时，为了支持后续DC/OS版本升级，当前唯一推荐的安装方式是使用**高级安装**模式。