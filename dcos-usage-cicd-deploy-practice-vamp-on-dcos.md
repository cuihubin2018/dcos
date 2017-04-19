## Vamp on DC/OS

Vamp对DC/OS具有良好的支持。部署完成后类似如下图所示：

![](/assets/vamp-on-dcos-deploy.png)

### 应用场景

Vamp官方为多种场景提供了示例。通过这些场景示例，我们可以很好的理解Vamp提供的强大的功能特性，并以此为模板整合处理实际需求。

- 部署单体应用
- 执行金丝雀部署
- 拆分单体应用并整合部署
- 服务的合并与删除
- 通过Wordpress示例演示Vamp如何从工件构建部署并与网关Gateways协同
- 如何自动化金丝雀部署并支持回滚
- 如何构建一个产生events的workflow

上述场景示例的详细信息请参考[官网文档](http://vamp.io/documentation/tutorials/overview/)。

在实际场景需求中，Vamp往往需要配合其他系统一起实现完整的软件开发流程控制，下述简要介绍其与Jenkins及其它一些服务的整合。

### 与Jenkins整合

### 与Marathon-LB整合

### 参考

- https://www.youtube.com/watch?v=RgTE1Y74A9o