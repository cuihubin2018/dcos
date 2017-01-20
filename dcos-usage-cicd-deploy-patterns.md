## 部署模式

### 蓝-绿部署（Blue-Green Deployment）

__蓝-绿部署__是通过创建两个版本的应用程序（蓝色和绿色），通过在两个版本之间安全切换，确保应用提供的服务实时在线的一种方式。

![](/assets/blue_green_deployments.png)
如上图所示，应用程序的新版本（绿色）部署后，如果通过功能及性能测试，可以在当前版本（蓝色）应用处理完所有的流量、请求及待处理操作后通过路由切换的方式将新的流量、请求调度给新的版本（绿色）。如果新版应用系统（绿色）出现问题，可以及时回滚到原有版本（蓝色）；如果一切正常可以停止原有版本（蓝色），回收资源。在发布新版本时，继续重复上述操作。

关于蓝-绿部署的详细过程，请参考[Martin Fowler的文章](https://martinfowler.com/bliki/BlueGreenDeployment.html)。

蓝-绿部署的优点包括：1）上线过程对用户透明，不影响用户的体验；2）可以在生产环境对新版本进行功能和性能进行测试，便于试点；3）在新系统出现故障时，可以及时降级回滚。

在生产环境中，通常使用脚本将上述过程集成到部署系统中实现过程的自动化控制。在DC/OS中，用户可以通过CLI命令实现蓝-绿部署过程。

#### 通过DCOS CLI实现蓝-绿部署

__前提条件__

- 应用通过[Marathon管理部署](/dcos-marathon-app-deployments.md)，并且实现[健康检查](/dcos-marathon-health-checks.md)。

- 应用程序必须提供一个指标度量接口，以确定应用是否有任何待处理操作。例如，应用程序可以用一个全局[Gauge](/dcos-admin-monitoring-prometheus-concepts.md)暴露当前排队的DB事务的数量。
- 安装JSON命令行处理工具-[jq](https://stedolan.github.io/jq/)。

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

5. 将新增的绿色应用服务实例添加到负载均衡池。

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

除了上述方案，也可以通过其他工具实现此过程，如[Vamp](http://vamp.io/)，[Swan](https://github.com/Dataman-Cloud/swan)，详细信息请参考后续章节。

### 金丝雀部署（Canary Deployment）

十七世纪的矿工，工作时会带一只金丝雀进入矿场，当矿井中二氧化碳浓度升高时，人类不易察觉，金丝雀会先死亡，矿工们以此作为监测二氧化碳浓度的指标。Google的Chrome浏览器提供金丝雀版本（Canary Version），并特别注明“胆小者请勿轻易尝试”。

![](/assets/canary-release-1.png)

__金丝雀部署__作为一种测试策略，一小部分服务器被升级到一个新版本或者新配置，随后保持一定的观察期。

![](/assets/canary-release-2.png)

如果没有任何异常出现，发布过程会继续，剩余的服务器也会升级到新版本。

![](/assets/canary-release-3.png)

如果出现异常，这部分单独修改过的应用服务可以很快被回退到原来的状态。

### 参考