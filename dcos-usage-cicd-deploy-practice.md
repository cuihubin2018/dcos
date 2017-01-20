## 部署实践

在DC/OS中实现蓝-绿部署或金丝雀部署，一种方案是需要通过Marathon和Marathon-LB配合实现。

### 部署方法

__前提条件__

- 应用通过[Marathon管理部署](/dcos-marathon-app-deployments.md)，并且实现[健康检查](/dcos-marathon-health-checks.md)。

- 应用程序必须提供一个指标度量接口，以确定应用是否有任何待处理操作。例如，应用程序可以用一个全局[Gauge](/dcos-admin-monitoring-prometheus-concepts.md)暴露当前排队的DB事务的数量。
- 安装JSON命令行处理工具-[jq](https://stedolan.github.io/jq/)（针对CLI模式）。

__操作步骤__

下述步骤中__蓝色__表示应用当前版本，__绿色__表示应用新版本。

1. 通过Marathon部署应用的新版本。为了区分，给应用添加一个新的ID（可以使用Git的`Commit ID`），在本示例步骤中，为新应用的ID添加`green`前缀：

  ```
# launch green
dcos marathon app add green-myapp.json
```
  如果使用API调用而非CLI，需要使用如下命令：
  
  ```
  curl -H "Content-Type: application/json" -X POST -d @green-myapp.json <hosturl>/marathon/v2/apps
  ```

2. 根据需要扩展绿色新版应用的服务实例数据到1个或多个（起始值为0）。注意，当前启动的服务实例仍未提供服务，因为还未配置负载均衡。

 ```
 # scale green
dcos marathon app update /green-myapp instances=1
```

3. 等待确保所有的绿色服务实例都正常启动并通过了健康检查。可以通过下述命令进行确认：

  ```
  # wait until healthy
dcos marathon app show /green-myapp | jq '.tasks[].healthCheckResults[] | select (.alive == false)'
```

4. 使用上述命令片段检查所有的绿色应用服务实例是否健康，如果存在任何异常，终止部署并解决问题。

5. 将新增的绿色应用服务实例添加到负载均衡池（Marathon-LB）。

6. 从当前蓝色应用服务中选择一个或多个实例。

  ```
  # pick tasks from blue
dcos marathon task list /blue-myapp
```

7. 更新负载均衡池配置，将上述选择的服务实例从池中移除。

8. 等待蓝色应用服务实例不再有待处理操作。使用应用提供的指标度量接口确定是否有待处理操作。

9. 一旦蓝色应用服务待处理的操作都已完成，通过该[API](https://mesosphere.github.io/marathon/docs/rest-api.html#post-v2-tasks-delete)杀死并缩减蓝色应用服务，参考下述命令：

  ```
  # kill and scale blue tasks
echo "{\"ids\":[\"<task_id>\"]}" | curl -H "Content-Type: application/json" -X POST -d @- <hosturl>/marathon/v2/tasks/delete?scale=true
```
  此命令将删除特定实例（具有0个待处理操作的实例），并防止它们重新启动。
  
10. 重复上述2-9步骤直至蓝色应用服务不再有实例运行。

11. 监控绿色应用服务状态（持续一段时间），确认一切正常，从DC/OS集群中移除蓝色应用服务。

  ```
  # remove blue
dcos marathon app remove /blue-myapp
```

在生产环境中，通常使用脚本将上述过程集成到部署系统中实现过程的自动化控制。

### 部署脚本

Marathon-LB针对上述部署方法提供了一个部署脚本[zdd.py](https://github.com/mesosphere/marathon-lb/blob/master/zdd.py)，通过该脚本可以实现零宕机部署（Zero-downtime Deployments）。

使用该脚本必须满足以下条件：

- 在应用的Marathon定义中设定`HAPROXY_DEPLOYMENT_GROUP` 和 `HAPROXY_DEPLOYMENT_ALT_PORT`两个标签（label）。

  - `HAPROXY_DEPLOYMENT_GROUP`：此标签唯一标识一对属于蓝色/绿色部署的应用程序，并将用作HAProxy配置中的应用程序名称。
  - `HAPROXY_DEPLOYMENT_ALT_PORT`：需要一个额外的服务端口，因为Marathon要求服务端口在所有应用程序中是唯一的。

- 当前只支持一个服务端口。

- ZDD脚本会调用Marathon的API并使用HAProxy的状态接口来正常终止实例。

- Marathon-LB容器必须以`privileged`模式运行（需要执行`iptables`命令）。

- 如果负责应用负载的HAProxy实例同时管理了TCP长连接，可能导致部署花费超出必要的时间。该脚本默认情况下设置5分钟等待HAProxy负载的连接逐步断开，但任何TCP长连接将导致HAProxy实例无法终止。

零宕机部署使用Lua模块来完成，该模块通过`/_haproxy_getpids`接口获取的HAProxy进程状态来报告当前正在运行的HAProxy进程数。

一个警告是，如果在同一个LB上有任何长连接，HAProxy将继续运行为这些连接提供服务，直到它们停止，这会导致部署过程无法完成。

### 脚本应用

使用该脚本可以实现蓝色/绿色应用程序两个版本同时运行并分割两者之间的流量。要实现此功能，需要设置`HAPROXY_DEPLOYMENT_NEW_INSTANCES`标签。

执行ZDD脚本时，通过参数`--new-instances`可以指定创建新版本应用程序的实例数并同时删除相同数量的旧版本应用程序实例（设定值等于旧应用程序实例数时，就是创建所有新应用程序实例并删除所有旧应用程序实例），以确保新应用程序和旧应用程序中的实例数之和等于`HAPROXY_DEPLOYMENT_TARGET_INSTANCES`的设定值。

除了上述方案，也可以通过其他工具实现此过程，如[Vamp](http://vamp.io/)，[Swan](https://github.com/Dataman-Cloud/swan)，详细信息请参考后续章节。



