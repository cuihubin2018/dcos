### API管理

当前互联网特别是移动互联网，设备与平台之间的交互的基础是服务API接口。以API驱动的开发是团队之间最常用的协作方式，而作为交互的基石，API的准确性，完整性和及时性是影响开发效率的关键。

在生产环境中，创建、发布、维护、监控和保护任意规模的API，接收和处理成千上万个并发API的调用，管理流量、授权和访问控制、监控以及API版本也是采用微服务架构所必须解决的问题。

解决上述问题的一种方案是[API网关（API Gateway）](http://microservices.io/patterns/apigateway.html)，API网关的主要职责是提供统一的服务入口，让微服务对前端透明，并提供路由，缓存，安全，过滤，流量控制等管理功能，其具体的实现大家可能首先会想到Netflix的[Zuul](https://github.com/Netflix/zuul)以及亚马逊的Amazon API Gateway。

当前，微服务的API接口根据不同的业务需求有不同的技术实现选择，常用的有HTTP/REST，RPC/gRPC，Thrift等，而对于一些领域需要特定的协议如流媒体的RTMP, HLS, 和 HDS等，API网关如何兼顾这些协议也是一个很大的挑战。


