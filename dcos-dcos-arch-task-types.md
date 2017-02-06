## 任务类型

DC/OS可以运行许多不同类型的工作负载，这些工作负载是由任务（Tasks）组成的。DC/OS的任务实际上是由DC/OS内置调度程序或在DC/OS上运行的其他调度程序服务调度的**Mesos任务**。

### Executors

任务（Tasks）是在启动时由调度程序（Scheduler）指定的Mesos执行程序（Executors）执行。在Mesos中，调度程序（Scheduler）及其执行程序（Executors）组成一个服务框架（Framework）。在DC/OS中，通常直接使用任务，调度器和执行器三个概念术语。

#### 内置的Executor

Mesos为所有调度器提供了内置的执行器，当然，调度器还可以有自定义的执行器。

- Command Executor 执行shell脚本命令或Docker容器

- Default Executor (Mesos 1.1) 执行一组shell脚本命令或Docker容器

更多详细信息请参考[Mesos Framework Development Guide](https://mesos.apache.org/documentation/latest/app-framework-development-guide/)。

### Schedulers

在DC/OS中，用户通常不直接创建或与任务交互。

#### 内置调度器

- Marathon调度器 支持持续和并行运行的Apps 和 Pods 服务。

- Metronome调度器 调度立即执行或按自定义调度规则执行的Job任务。

#### 自定义调度器

用户根据业务需求自己实现的调度器，常见的有Kafka调度器，Cassandra调度器和Spark调度器等。