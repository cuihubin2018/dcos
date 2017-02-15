## Mesos框架的高可用性设计

Mesos框架是用来管理任务的。对于一个高可用Mesos框架，它必须能在各种故障场景中一贯正确的管理任务。框架开发者在进行开发时应该考虑下述最常见的故障条件：

- 一个框架Scheduler可能因为Mesos Master崩溃或网络故障而无法与其建立连接，如果Master已经配置为使用高可用性模式，这会导致选举一个Mesos Master的副本成为新的Leader。在这种情况下，Scheduler应与新的Master Leader重新注册，并确保任务的状态是一致的。

- 框架调度程序Scheduler正在运行的主机可能会故障，为了确保该框架仍然可用并且能够继续调度新的任务，框架开发者应确保在不同的节点上运行调度程序Scheduler的多个副本，并且在一个框架Scheduler Leader失效后，Scheduler的一个备份副本能够提升成为新的Leader。虽然在下面章节提供了一些建议，Mesos本身并没有规定框架开发者应该如何处理这种情况。可以使用长时间运行的任务调度框架如Apache Aurora或Marathon来调度所开发的框架程序的多个副本。

- 任务正在运行的主机可能会故障，或者，主机本身没有问题而是由于网络隔断等原因使得运行在主机节点上的Mesos Agent可能无法与Mesos Master通信。

需要注意的是这些故障可能同时发生。

### Mesos架构

在讨论上面列出的故障的具体解决方案之前，需要强调的是Mesos的某些方面的设计是如何影响高可用性的：



### 参考

- http://mesos.apache.org/documentation/latest/high-availability-framework-guide/

- https://www.youtube.com/watch?v=rvX0P4elmmc&index=34&t=3s