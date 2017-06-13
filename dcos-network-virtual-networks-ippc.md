## IP-per-Container

### 概述

从网络的角度来看，为了提供类似于虚拟机环境提供的最终用户体验，为每个容器提供自己的IP地址和网络命名空间是非常重要的。为容器提供隔离的网络堆栈可以确保逻辑网络隔离以及容器之间的网络性能隔离。此外，**Ip-per-container**允许用户/开发人员使用传统的网络操作工具（traceroute，tcpdump，wireshark）和他们熟悉的应用程序，并帮助他们在调试网络连接/性能问题方面的提高生产力。从操作的角度来看，识别容器特定流量也变得更容易，从而简化了对容器的网络性能和安全策略的实施。

DC/OS的默认每容器IP解决方案需要与运行DC/OS的网络无关。因此，为了实现容器IP，需要使用虚拟网络。Overlay技术使容器网络拓扑与底层主机网络无关。此外，Overlay技术提供了容器流量与主机流量的完全隔离，使得对容器流量的策略执行更简单并且独立于主机流量。实现Overlay技术的挑战是容器发送和接收流量时封装和解封装容器流量的成本。为了最小化这些封装和拆分操作对容器网络吞吐量的影响，Overlay技术必须将封装/拆分操作作为主机操作系统内核中的数据包处理流水线的一部分。

为了实现DC/OS的容器IP解决方案，使用Overlay技术，因此需要选择Linux内核主动支持的Overlay技术。Linux内核支持的最常见的Overlay技术是**VxLAN**。

DC/OS的Overlay设计做出以下假设/约束：

- 由于集中式IPAM缺乏对可用性和可靠性的保证，方案无法使用集中的IPAM。

- 需要避免第2层的单播泛洪，使网络可扩展。这意味着方案不能依赖广播ARP来获取容器的MAC地址。

- 该解决方案需要支持`MesosContainerizer`和`DockerContainerizer`，即在同一个Overlay上，方案应该能够同时运行Docker和Mesos容器，并允许它们相互通信。

基于上述假设/约束，DC/OS使用基于VxLAN的第三层Overlay技术。



### 术语

- 二层交换

  二层交换是工作在OSI七层协议的2层上，也就是数据链路层，数据链路层只有MAC地址没有IP地址，无法配置静态路由（有个例外是可以在vlan上配IP）。
  
- 三层交换

  三层交换是工作在OSI七层协议的3层上，可以启用路由协议（静态或动态），还可以启用访问控制列表，总之，路由能做的事三层交换基本都能做，三层交换没有NAT。
  
- IPAM

  IPAM用于发现、监视、审核和管理企业网络上使用的IP地址空间。IPAM 可以对运行动态主机配置协议 (DHCP) 和域名服务 (DNS) 的服务器进行管理和监视。
- ARP

  地址解析协议，即ARP（Address Resolution Protocol），是根据IP地址获取物理地址的一个TCP/IP协议。主机发送信息时将包含目标IP地址的ARP请求广播到网络上的所有主机，并接收返回消息，以此确定目标的物理地址；收到返回消息后将该IP地址和物理地址存入本机ARP缓存中并保留一定时间，下次请求时直接查询ARP缓存以节约资源。
- VXLAN

  VXLAN(Virtual Extensible LAN),是一种网络虚似化技术,试图改进大型云计算的部署时的扩展问题.可以说是对VLAN的一种扩展,由于VLAN Header头部限长是12bit, 导致VLAN的限制个数是`2^12=409`6个,无法满足日益增长的需求.目前 VXLAN 的报文 Header 内有 24 bit，可以支持 `2^24`次方的 VNI 个数(VXLAN中通过VNI来识别，相当于VLAN ID). [参考资料](http://blog.csdn.net/achejq/article/details/12945127)
- VTEP

  VXLAN隧道终端 (VXLAN Tunneling End Point)，用于多VXLAN报文进行封装/解封，包括MAC请求报文和正常VXLAN数据报文，在一端封装报文后通过隧道向另一端VTEP发送封装报文，另一端VTEP接收到封装的报文解封装后根据被封装的MAC地址进行转发。VTEP可由支持VXLAN的硬件设备或软件来实现。




### 场景

下图通过数据包的流动阐述了DC/OS中基于Overlay技术实现的容器网络通信方案。

![](/assets/dcos-overlay-fig-1.png)

上图演示了一个拥有2个Agent的DC/OS集群，要让DC/OS的Overlay工作，需要一个足够大的地址空间为之上的容器分配地址。选定的地址空间不能覆盖主机网络的地址空间以避免因为错误的配置影响整个主机网络。在上例中，主机网络使用`10.0.0.0/8`的地址空间，Overlay使用`9.0.0.0/8`的地址空间。

理想情况下，我们希望IPAM为Overlay层上的所有容器执行地址分配。然而，很难通过全球IPAM来解决可靠性和一致性问题。例如，当Agent宕机时，我们如何通知IPAM？IPAM应该提供多大的可用性来保证集群的99％正常运行时间（没有IPAM容器无法运行）？鉴于全球IPAM引发的复杂性，DC/OS选择采用更简单的架构，将地址空间划分为较小的块，并允许Agent节点拥有这些块。在该示例中，`9.0.0.0/8`的空间已被拆分为`/24`个子网，每个子网都由Agent拥有。在示例中，Agent 1已分配`9.0.1.0/24`，Agent 2已分配`9.0.2.0/24`。

给定Agent的一个`/24`子网，子网需要进一步拆分成更小的块，因为Mesos的Agent可能需要同时启动Mesos容器以及Docker容器。与Agent一样，我们可以在Mesos容器和Docker容器之间进行静态地址分配。在这个例子中，我们将`/24`空间静态地刻画成了一个`/25`空间，分别用于Mesos容器和Docker容器。此外，请注意，Mesos容器由`MesosContainerizer`在“**m-dcos**”桥上启动，Docker容器由`DockerContainerizer`使用Docker守护程序在“**d-dcos**”桥上启动。下面看一下数据包在容器到容器通信过程中的具体流转过程。

#### 同一宿主机上的容器到容器通信

假设`9.0.1.0/25`上的Mesos容器想要与`9.0.1.128/25`上与Docker容器通信。报文流转如下：

1. `9.0.1.0/25`上的容器将数据包发送到默认网关，此示例中是“**m-dcos**”网桥，分配了IP地址`9.0.1.1/25`。

2. **m-dcos**网桥将处理数据包，并且由于**m-dcos**网桥存在于主机网络命名空间中，它将向网络堆栈发送数据包以进行路由。

3. 数据包将被发送到**d-dcos**，它将数据包切换到`9.0.1.128/25`子网。

#### 跨宿主机的容器到容器通信

假设`9.0.1.0/25`（Agent 1）上的Mesos容器想要在`9.0.2.128/25`（Agent 2）上与Docker容器通信。报文流转如下：

- 来自9.0.1.0/25的数据包将被发送到“m-dcos”网桥（9.0.1.1/25）上的默认网关。该网桥将处理数据包并将其发送到网络堆栈。由于桥接器已在主机网络命名空间中注册，数据包将使用主机网络命名空间路由表进行路由。

- 主机网络路由表中存在一条路由`9.0.2.0/24 -> 44.128.0.2`（将在下一节中说明如何安装此路由），这实质上是告诉主机此数据包需要通过VxLAN1024发送（主机网络路由表里也存在44.128.0.0/20 -> VxLAN1024的路由条目）。

- 由于44.128.0.2直接连接在VxLAN1024上。内核会尝试对44.128.0.2进行ARP解析。在配置Overlay期间，DC/OS会在ARP缓存中为VTEP端点44.128.0.2安装一个条目，因此内核ARP查找会成功。

- 内核路由模块会将目的MAC设置为`70:B3:D5:00:00:02`的数据包，发送给VxLAN设备VxLAN1024。

- 要转发数据包，VxLAN1024需要一个MAC地址为`70:B3:D5:00:00:01`作为Key的条目。VxLAN转发数据库中的此条目是由DC/OS编程写入，指向Agent 2的IP地址。DC/OS能够对此条目进行编程的原因是因为它知道Agent的IP和所有Agent上存在的VTEP。

- 此时，数据包封装在UDP头（由VxLAN FDB指定）中，并发送到Agent 2上存在的VTEP。

- Agent 2上的VxLAN1024对数据包进行解封，由于目标MAC地址设置为Agent 2上的VxLAN1024的MAC地址，因此该数据包将由Agent 2上的VxLAN1024进行处理。

- 在Agent 2中，路由表具有子网9.0.2.128/25的条目，直接连接到桥“d-dcos”。因此，数据包将被转发到连接到“d-dcos”网桥的容器进行处理。

### 架构

![](/assets/dcos-overlay-fig-2.png)

上图描述了DC/OS实现的用于实现Overlay叠加网络技术的软件体系结构。图中橙色的块是必须建造的缺失组件。

#### DC/OS modules for Mesos

要配置基础的DC/OS叠加网络，需要一个可以为每个Agent分配子网的实体组件。该实体组件还需要配置Linux网桥，以及在自己的子网中启动Mesos和Docker容器。此外，每个Agent上的VTEP需要分配IP地址和MAC地址，并且Agent上的路由表需要配置正确的路由，以便于容器通过DC/OS叠加网络进行通信。

Master节点上的Overlay组件：

- 它将负责将子网分配给每个Agent。下述将更详细地描述Master组件如何使用复制的日志来检查这些信息，以便在故障切换到新的Master时进行恢复。

- 它将监听Agent叠加网络组件以便于注册和恢复其分配的子网。Agent上的Overlay组件还将使用此端点来了解分配给Agent自身的叠加子网（在多个虚拟网络的情况下），分配给叠加网络中每个Mesos和Docker网桥的子网以及分配给容器的VTEP IP和MAC地址。

- 它通过HTTP端点“`overlay-master/state`”来展示DC/OS中所有虚拟网络的状态。响应信息的详细描述参考[此处](https://github.com/dcos/mesos-overlay-modules/blob/master/include/overlay/overlay.proto#L86)。


Agent节点上的Overlay组件：

- 它负责向Master节点上的Overlay组件（Master模块）注册。注册后，它检索分配的Agent子网，分配给其Mesos和Docker网桥的子网以及VTEP信息（VTEP的IP和MAC地址）。

- 基于分配的Agent子网，它负责生成用于MesosContainerizer的`network/cni`隔离器的CNI（容器网络接口）网络配置。

- 它负责创建DockerContainerizer使用的Docker网络。

- 它通过HTTP端点`overlay-agent/overlays`。虚拟网络服务使用此接口检索有关该特定代理的叠加网络信息。

**使用复制的日志来协调Master中的子网分配**

**配置MesosContainerizer和DockerContainerizer以使用已分配的子网**
一旦DC/OS模块从主DC/OS模块检索子网信息，它将执行以下操作，以允许MesosContainerizer和DockerContainerizer在叠加层上启动容器：

对于MesosContainerizer，DC/OS模块可以在指定位置生成CNI配置。 CNI配置将具有network/cni隔离器的桥接信息和IPAM信息，以在m-[虚拟网络名称]桥上配置容器。

对于DockerContainerizer，DC/OS模块在检索子网后，将通过docker network命令创建一个具有规范名称d-[虚拟网络名称]的容器网络。它将使用以下docker命令执行此操作：
```
docker network create \
--driver=bridge \
--subnet=<CIDR> \
--opt=com.docker.network.bridge.name=d-<virtual network name>
--opt=com.docker.network.bridge.enable_ip_masquerade=false
--opt=com.docker.network.driver.mtu=<overlay MTU>
<virtual network name>
```
注意：DockerContainerizer使用DC / OS覆盖的假设是主机正在运行Docker v1.11或更高版本。 
注意：默认的<overlay MTU> = 1420字节。

#### 虚拟网络服务（叠加网络编排）

虚拟网络服务（[Navstar](https://github.com/dcos/navstar.git)）是在每个Agent上运行的叠加协调器服务，负责以下功能。它是一个包含DC/OS叠加网络的非实时组件以及其他与网络相关的DC/OS服务模块的子系统。在每个Agent上运行的虚拟网络服务负责以下功能：

1. 与Agent叠加网络模块对话，获取分配给Agent的子网，VTEP IP和MAC地址。
2. 在Agent上创建VTEP。
3. 将路由写入到各个Agent的各个子网。
4. 使用VTEP IP和MAC地址写入ARP缓存。
5. 使用VTEP MAC地址和隧道端点信息写入VxLAN FDB。
6. 使用[Lashup](https://github.com/dcos/lashup/)（一个分布式CRDT存储引擎）可以将Agent叠加网络信息可靠地传播到集群内的所有Agent。这是虚拟网络服务执行的最重要的功能之一，因为只有拥有群集中所有Agent的全部信息，虚拟网络服务才能对所有Agent上的所有叠加子网的每个Agent进行路由。

### 参考

- https://dcos.io/docs/1.9/overview/design/overlay/#using-replicated-log-to-coordinate-subnet-allocation-in-master-
