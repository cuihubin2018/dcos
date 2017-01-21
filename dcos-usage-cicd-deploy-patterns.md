## 部署模式

### 蓝-绿部署（Blue-Green Deployment）

__蓝-绿部署__是通过创建两个版本的应用程序（蓝色和绿色），通过在两个版本之间安全切换，确保应用提供的服务实时在线的一种方式。

![](/assets/blue_green_deployments.png)
如上图所示，应用程序的新版本（绿色）部署后，如果通过功能及性能测试，可以在当前版本（蓝色）应用处理完所有的流量、请求及待处理操作后通过路由切换的方式将新的流量、请求调度给新的版本（绿色）。如果新版应用系统（绿色）出现问题，可以及时回滚到原有版本（蓝色）；如果一切正常可以停止原有版本（蓝色），回收资源。在发布新版本时，继续重复上述操作。

关于蓝-绿部署的详细过程，请参考[Martin Fowler的文章](https://martinfowler.com/bliki/BlueGreenDeployment.html)。

蓝-绿部署的优点包括：1）上线过程对用户透明，不影响用户的体验；2）可以在生产环境对新版本进行功能和性能进行测试，便于试点；3）在新系统出现故障时，可以及时降级回滚。


### 金丝雀部署（Canary Deployment）

十七世纪的矿工，工作时会带一只金丝雀进入矿场，当矿井中二氧化碳浓度升高时，人类不易察觉，金丝雀会先死亡，矿工们以此作为监测二氧化碳浓度的指标。Google的Chrome浏览器提供金丝雀版本（Canary Version），并特别注明“胆小者请勿轻易尝试”。

![](/assets/canary-release-1.png)

__金丝雀部署__作为一种测试策略，一小部分服务器被升级到一个新版本或者新配置，随后保持一定的观察期。

![](/assets/canary-release-2.png)

如果没有任何异常出现，发布过程会继续，剩余的服务器也会升级到新版本。

![](/assets/canary-release-3.png)

如果出现异常，这部分单独修改过的应用服务可以很快被回退到原来的状态。

### 参考

https://martinfowler.com/bliki/BlueGreenDeployment.html

https://martinfowler.com/bliki/CanaryRelease.html