## 理解DC/OS

DC/OS的设计理念之一就是容器不应该使用主机操作系统上的任何东西。而DC/OS本身也遵循这一理念，除了基础文件系统、磁盘和网络，DC/OS仅依赖Systemd服务，其它的环境和服务如Python，curl等，已被DC/OS封装在自己的运行环境中。

DC/OS作为数据中心操作系统，通常部署在IDC机房，用于构建和管理面向企业级、云端的大规模业务集群。

容器（非单指Docker）是DC/OS中的最小单元，围绕容器，DC/OS提供与之相关的运行，存储，网络，监控，日志，部署，安全和多实例的调度，负载及服务发现等一系列全栈生态解决方案。

在DC/OS之上可以部署由内置Marathon调度CPU、内存、存储和网络资源的任意应用服务，也可以部署与DC/OS的核心Mesos进行深度整合的自定义业务调度框架，诸如[Marathon](https://github.com/mesosphere/marathon)本身，[Aurora](http://aurora.apache.org/)和[Singularity](https://github.com/HubSpot/Singularity)等长期服务调度框架，[Hadoop](https://github.com/mesos/hadoop)，[Spark](http://spark.apache.org/)，[Storm](https://github.com/mesos/storm)，[MPI](https://github.com/mesosphere/mesos-hydra)和[Exelixi](https://github.com/mesosphere/exelixi)等大数据处理服务框架，[Chronos](https://github.com/mesos/chronos)，[Jenkins](https://github.com/jenkinsci/mesos-plugin)等批处理计划服务框架，[Alluxio](http://alluxio.org/)，[Cassandra](https://github.com/mesosphere/cassandra-mesos)，[ElasticSearch](https://github.com/mesos/elasticsearch)和[MrRedis](https://github.com/mesos/mr-redis)等数据存储服务框架，[Vamp](http://vamp.io/)等DevOps工具服务。

基于DC/OS，用户可以快速构建任意业务场景，也可以将遗留应用快速迁移至弹性计算调度环境。