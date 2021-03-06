## Prometheus

[Prometheus](https://github.com/prometheus)是一个最初在[SoundCloud](http://soundcloud.com/)上构建的开源系统监控和告警工具包。Prometheus于2016年加入了[Cloud Native Computing Foundation](https://cncf.io/)，作为继Kubernetes之后的第二个加入托管的项目。

### 特性

* 多维数据模型（由度量名称和键/值对标识的时间序列）

* 提供灵活的查询语言来利用这些维度

* 单个服务器节点是自治的，不依赖分布式存储

* 通过HTTP上的PULL模型进行时间序列数据收集

* 通过中间网关（Pushgateway）支持PUSH模型收集时间序列数据

* 通过服务发现或静态配置发现采集的目标（Target）

* 支持多种模式的图形和仪表板

### 组件

* Prometheus生态系统由多个组件组成，其中许多组件是可选的：

* Prometheus Server作为核心组件，负责采集和存储时间序列数据

* 客户端库可用于开发检测应用程序代码

* 推送网关（Pushgateway）组件用于支持短期任务（short-lived jobs）

* 基于Rails/SQL实现的GUI仪表板构建器

* 多种指特定的标信息采集器（exporters），如：HAProxy，StatsD，Ganglia等

* 告警管理器组件，支持多种方式报警

* 提供命令行查询工具

* 其它众多支持工具

### 架构

![](/assets/dcos-prometheus-architecture.png)

上图说明了Prometheus及其一些生态系统组件的总体架构。Prometheus通过直接或者通过Pushgateway间接从采集器（Exporters）抓取指标样本并存储在本地，然后对这些样本数据记录执行设定的统计分析规则，以此创建新的统计分析时间序列或生成警报。

### 适用性

Prometheus适用于记录任何纯数值时间序列。它适用于以机器为中心的监控以及高度动态的面向服务的体系结构的监控。在微服务领域，其对多维数据收集和查询的支持是一个特别的优势。

Prometheus重视可靠性。即使在故障情况下，也可以随时查看系统的可用统计信息。但如果需要100％的准确性，例如对于每个请求的计费，Prometheus不是一个好的选择，因为收集的数据可能不会是详细和完整的。
