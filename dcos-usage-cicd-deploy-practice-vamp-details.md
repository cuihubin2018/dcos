## 深入理解Vamp

Vamp有一些可以操作基本的实体（entities）或工件（artifacts），这些可以被归类为静态资源描述和动态运行实体。Vamp提供的API操作针对静态资源描述大多数都是同步的，针对动态运行实体则基本上是异步的。

- 静态资源描述

 - Blueprints：描述了`Breeds`在运行时如何工作以及它们应具有的属性。

 - Breeds：描述单一服务及其依赖。

 - Scales：定义所部署的服务的规模。

- 动态运行时实体

 - Deployments：是运行中的Blueprints。可以为一个Blueprint构建多个部署（Deployments），并在运行时对每个部署执行操作。此外，还可以将任何正在运行的部署转换为Blueprint。

 - Gateways：是“稳定”路由端点 - 由端口（incoming）和路由（outgoing）定义。

 - Workflows：是部署在集群上的应用程序（服务），用于动态更改运行时配置（例如SLA，缩放，条件权重更新）。

Vamp对跨团队协作共同构建复杂的应用部署提供了良好的支持。通过占位符（Placeholders）：

- Breeds在存在之前就可以在Blueprints中引用。

- 当引用SLA时，不需要知道SLA的内容。

- 可以引用其他人定义并在某个时间才填充的变量。

Vamp只会在部署时检查这些引用。

### Breeds

### Deployments

### Environment variables

### Gateways

### Conditions

### Events

### SLA(Service Level Agreement)

### Escalations

### Referencing artifacts

### Workflows

### Sticky Sessions

### Service Discovery

### Virtual Hosts