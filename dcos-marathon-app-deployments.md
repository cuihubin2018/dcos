## 应用服务部署

在Marathon中，每次应用程序或组的定义的改变都被视作是执行一次部署。部署是一组操作，可以执行：

* 启动/停止一个或多个应用

* 升级一个或多个应用

* 伸缩一个或多个应用


部署并不是立即完成，它需要时间。在部署过程中，Marathon会显示该部署为活动状态。可以同时执行多个部署，前提是每个应用一次仅被一个部署操作。如果一个部署请求被提交，如果该部署操作的应用正在被另一个处于活动状态的部署所操纵，该部署将被拒绝。

### 依赖

如果应用程序没有任何依赖，则可以以任何顺序部署而不受限制。如果应用程序之间存在依赖关系，那么必须按特定顺序执行部署操作。

![](/assets/dcos-marathon-app-dependency.png)  
如上图示例，app应用依赖db应用。

* 启动：如果app和db部署到DC/OS中，首先启动db，然后启动app

* 停止：如果app和db需要从DC/OS中移除，首先移除app，然后移除db

* 升级：参见后续“滚动重启”小节

* 伸缩：如果db和app需要伸缩，db首先伸缩，然后才是app


### 滚动重启

开发和运维人员面临的最常见问题之一是如何发布新版本的应用程序。追根溯源，这个过程包括两个阶段：使用新版本启动一组进程，并停止一组旧版本进程。如何处理这一过程有很多种模型。

在Marathon中有一个具有最小健康容量\(minimumHealthCapacity\)的升级策略，它在应用程序级别上定义。minimumHealthCapacity针对应用实例，要求应用程序的某个版本在更新期间始终必须具有的健康实例的数量的百分比。

* **minimumHealthCapacity == 0**：在部署新版本之前，可以杀死所有旧实例。

* **minimumHealthCapacity == 1**：在旧版本停止之前，并行部署新版本的所有实例。

* minimumHealthCapacity介于0和1之间：将旧版本实例数缩放到minimumHealthCapacity设定的百分比，同时将新版本实例数启动到minimumHealthCapacity设定的百分比。如果这一步能成功完成，再将新版本扩张到100％，并停止所有旧版本。


如果存在依赖关系，处理过程就有点复杂。当上述示例的应用程序更新时，将执行以下操作：

* 升级db应用，直到所有db实例都被替换，就绪和健康（将upgradeStartegy考虑在内）

* 升级app应用，直到所有app实例都被替换，就绪和健康（将upgradeStrategy纳入考虑）


如果minimumHealthCapacity大于0.5。集群需要有更多的容量用于更新过程。在这种情况下，同一应用程序的一半以上的新旧实例将并行运行。如果存在依赖性，则这些容量约束将加和计算。如上例中，假定minimumHealthCapacity的值分别按db定义0.6，app定义0.8计算，这意味着在更新的情况下，db要有12个（6旧和6新）实例，app要有32个实例（16旧和16新）在并行运行。

### 强制部署

应用程序一次只能进行一次部署更改。对应用程序的其他更改必须等待，直到第一次部署完成。可以在运行部署时通过force参数来绕过此规则。REST接口允许force参数应用于所有状态更改操作。

注意：force参数应该仅在部署失败的情况下使用！

如果部署时设置了force参数，则受此部署影响的所有部署都将被取消，这可能使系统处于不一致的状态。特别是，当应用程序处于滚动升级的中间过程然后部署被取消时，它可能会处于一些旧的和某些新的实例并行运行的状态。

如果新部署未更新该应用，它将保持这种状态，直到为该应用程序执行新的部署。相比之下，唯一能够安全的强制更新的是那些仅影响单个应用程序的部署。因此，强制执行影响多个应用程序的部署的唯一理由是纠正失败的部署。

### 部署失败

部署包括若干步骤。这些步骤一个接一个地执行。只有上一步成功完成后，才能执行下一步。在某些情况下，部署的步骤永远不会成功完成。例如：

* 新的应用程序不能正确启动

* 新的应用程序未通过健康检查

* 新应用程序的依赖项未声明，并且不可用

* 集群的容量被用尽

* 该应用程序使用Docker容器，并且未遵循“[在Marathon上运行Docker Containers](https://mesosphere.github.io/marathon/docs/native-docker.html)”中列出的更改

* ......


在这种情况下的部署将永远不能完成。要修复这种情况，必须应用新的部署来纠正当前部署的问题。

### /v2/deployments

可以通过/v2/deployments接口访问正在运行的部署列表。每个部署都有几个可用信息：

* affectedApps：哪些应用程序受此部署影响

* steps：为此部署执行的步骤

* currentStep：当前实际执行的步骤


每一步骤都可以有几个操作。步骤内的操作同时执行。可能的操作有：

* **ResolveArtifacts** 解析应用程序的所有工件，并将其保留在工件仓库中

* **StartApplication** 启动指定的应用程序

* **StopApplication** 停止指定的应用程序

* **ScaleApplication** 缩放指定的应用程序

* **RestartApplication** 将指定的应用程序重新启动到minimumHealthStrategy设置的容量

* **KillAllOldTasksOf** 删除指定应用程序之外的其余任务

### 单应用实例部署

通常，一些遗留的应用服务无法同时运行多个实例，通过**标签**：`MARATHON_SINGLE_INSTANCE_APP`可以确保这类服务只有一个实例运行。

1. 在应用定义中，将标签`MARATHON_SINGLE_INSTANCE_APP`的值设置为true：
```
"labels":{
    "MARATHON_SINGLE_INSTANCE_APP": "true",
  }
```

2. 设置`upgradeStrategy`，在调整实例部署时让Marathon可以杀死服务实例进程：
```
"upgradeStrategy":{
    "minimumHealthCapacity": 0,
    "maximumOverCapacity": 0
  }
```

### 参考

[https://mesosphere.github.io/marathon/docs/deployments.html](https://mesosphere.github.io/marathon/docs/deployments.html)

