## CNI for Mesos 容器

本文档描述了`network/cni`隔离器，此隔离器通过实现容器网络接口（CNI）规范为Mesos容器化提供网络隔离机制。`network/cni`隔离器可以让使用Mesos容器化启动的容器连接到几种不同类型的IP网络。容器可以使用的网络技术范围包括从传统的第3层/第2层网络，如VLAN，ipvlan，macvlan到用于容器编排的新型网络，如Calico，Weave和Flannel。Mesos容器化默认启用了`network/cni`隔离器。

### 动机

### 使用

### 示例