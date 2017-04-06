## 理解DC/OS

DC/OS的设计理念之一就是容器不应该使用主机操作系统上的任何东西。而DC/OS本身也遵循这一理念，除了基础文件系统、磁盘和网络，DC/OS仅依赖Systemd服务，其它的环境和服务如Python，curl等，已被DC/OS封装在自己的运行环境中。

DC/OS作为数据中心操作系统，通常部署在IDC机房，用于构建和管理面向企业级、云端的大规模业务集群。

容器（非单指Docker）是DC/OS中的最小单元，围绕容器，DC/OS提供与之相关的运行，存储，网络，监控，日志，部署，安全和多实例的调度，负载及服务发现等一系列全栈生态解决方案。

在DC/OS之上可以部署由内置Marathon调度CPU、内存、存储和网络资源的任意应用服务，也可以部署与DC/OS的核心Mesos进行深度整合的自定义业务调度框架，诸如[Marathon](https://github.com/mesosphere/marathon)本身，[Aurora](http://aurora.apache.org/)和[Singularity](https://github.com/HubSpot/Singularity)等长期服务调度框架，[Hadoop](https://github.com/mesos/hadoop)，[Spark](http://spark.apache.org/)，[Storm](https://github.com/mesos/storm)，[MPI](https://github.com/mesosphere/mesos-hydra)和[Exelixi](https://github.com/mesosphere/exelixi)等大数据处理服务框架，[Chronos](https://github.com/mesos/chronos)，[Jenkins](https://github.com/jenkinsci/mesos-plugin)等批处理计划服务框架，[Alluxio](http://alluxio.org/)，[Cassandra](https://github.com/mesosphere/cassandra-mesos)，[ElasticSearch](https://github.com/mesos/elasticsearch)和[MrRedis](https://github.com/mesos/mr-redis)等数据存储服务框架，[Vamp](http://vamp.io/)等DevOps工具服务。

基于DC/OS，用户可以快速构建任意业务场景，也可以将遗留应用快速迁移至弹性计算调度环境。

在本篇，我们将详细探讨组成DC/OS的全栈解决方案的各个服务的功能及原理。

### DC/OS是什么

![](/assets/dcos-architecture-layers.png)
作为数据中心操作系统，DC/OS本身就是一个分布式系统，一个集群管理器，一个容器平台和一个操作系统。

#### 一个分布式系统

作为分布式系统，DC/OS包括由多个Master节点调度管理的一组Agent节点。像其他分布式系统一样，运行在Master节点上的若干组件与其各自的对等体之间通过Leader选举提供一致性服务。

#### 一个集群管理器

作为集群管理器，DC/OS管理在Agent节点上运行的资源和任务。Agent节点向集群提供资源。然后将这些资源捆绑到资源offer中，并提供给注册的调度程序。然后，调度程序接受这些资源offer并将其资源分配给特定任务，将任务间接放置在特定Agent节点上。与外部集群供应器不同，DC/OS本身在集群中运行，并管理其启动的任务的生命周期。此集群管理功能主要由[Apache Mesos](http://mesos.apache.org)提供。

#### 一个容器平台

作为容器平台，DC/OS包括两个内置的任务调度程序（Marathon和DC/OS作业（Metronome））和两个容器运行时（Docker和Mesos）。综合起来，这个功能通常被称为容器编排。除了用于服务和作业的内置调度器之外，DC/OS还支持用于处理更复杂的特定于应用程序的操作逻辑的自定义调度程序。诸如数据库和消息队列之类的有状态服务通常利用这些自定义调度器来处理高级场景（例如：配置，拆除，备份，恢复，迁移，同步，重新平衡等）。

DC/OS上的所有任务都是容器化的。容器可以是从容器仓库（例如[Docker Hub](https://hub.docker.com/)）下载的映像启动，或者它们可以是在运行时容器化的本地可执行文件（例如二进制文件或脚本）。尽管目前每个节点都需要Docker，但由于组件和软件包迁移到使用Mesos Universal Container Runtime进行映像和本地工作负载，因此可能会在以后变为可选。

#### 一个操作系统

作为操作系统，DC/OS提取集群硬件和软件资源，并为应用程序提供通用服务。在集群管理和容器业务流程功能之上，DC/OS提供的常见服务还提供包管理，网络，日志记录和度量，存储和卷以及身份管理等。

不同于Linux，DC/OS不是主机操作系统。 DC/OS跨越多台机器，但依靠每台机器拥有自己的主机操作系统和主机内核。