## 基于Vamp的部署实践

VAMP是一种开源解决方案，能够为基于容器和微服务的系统提供简便易用的金丝雀测试和发布功能。此外它还提供了强大的工作流功能，例如自动缩放之后进行恰当的排空（Drain）。

![](/assets/Vamp-architecture-and-components.png)

### Vamp概述

VAMP本身并不是一个完整的微服务/容器类PaaS堆栈，而只是专注于专门为通用微服务/容器堆栈提供高层次的金丝雀测试/发布和自动缩放功能。VAMP能与很多容器和微服务平台集成，例如Mesosphere的Open DC/OS、Apache Mesos/Mesosphere Marathon、Docker Swarm以及（很快即将支持的）Kubernetes，当然还有诸如Cisco的Mantl、CapGemini的Apollo，或Rancher Labs的Rancher等堆栈，以及诸如MS Azure Container Service等容器云，这些容器的编排程序都能与VAMP进行集成。

VAMP可以将整个集群的度量指标进行汇总（毕竟你可能并不关心集群中某一实例的运行状况而只关心集群中的实例整体），同时可以确保缩容后的服务能恰当地排空，并能照顾到粘滞会话（Sticky-session）和存活时间（Time-to-live）问题。然而自动缩放仅仅是度量指标驱动的优化工作流的实现之一。VAMP的工作流引擎和度量指标以及事件API使得用户可以轻松地创建各种类型的自定义事件以及度量指标驱动的优化工作流，例如“日出而作日落而息（Follow-the-sun）”、“装箱（bin-packing）”、“云爆发（cloud bursting）”、成本/性能优化，以及“灯火管制（brown-out）”场景。

VAMP可以对持续改进环路的三个核心领域（部署编排和缩放，可编程的路由，度量指标/大数据汇总）进行编排，同时可以很容易地创建并交付其他优化工作流。例如“日出而作日落而息”场景（白天资源扩容，夜间资源缩容）、装箱问题（优化基础结构的利用率），以及灯火管制反馈环路（使用流量调节技术将整个系统的速度降低到某一可接受的程度以避免彻底中断）。

### Vamp组件及功能

参考

http://www.infoq.com/cn/news/2016/07/vamp-microservices-platform