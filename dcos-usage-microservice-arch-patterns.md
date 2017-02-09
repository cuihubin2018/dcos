## 架构与模式

微服务架构的模式语言在[Chris Richardson 的微服务架构网站](http://microservices.io/index.html)有详细的介绍。

![](http://microservices.io/i/MicroservicePatternLanguage.jpg)

前述我们提到了当前主流的微服务基础框架Spring Boot/Cloud体系，Lagom，Vert.x及Dropwizard，还有许多其它的商业和开源的微服务基础框架在不断出现，Oracle和IBM等厂商也为J2EE 8提出了支持微服务的[Micro Profile](http://microprofile.io/)。

这些微服务基础框架在秉承自己体系的突出风格的同时，也在不断的将微服务架构关注的核心及周边基础设施进行开发或整合，以期望提供完善的基础微服务平台。而与此同时，由云计算发展而来的IaaS，PaaS也在不断演进，以期望为上层云原生应用提供更加丰富和完善的基础服务。随着这两大阵营的不断推进，两者之间的边界已经变得模糊，特别是Docker容器化的出现，加速了这一进程，Dev与Ops之间的“鸿沟”也变成了“坦途”。

下图（本图参考了Bilgin Ibryam的[文章](https://developers.redhat.com/blog/2016/12/09/spring-cloud-for-microservices-compared-to-kubernetes/)）展示了微服务架构与IaaS/PaaS各自的关注层面，从图中可以看出，微服务与IaaS/PaaS相辅相成，两者的结合可以为用户带来完善的云应用解决方案。

![](/assets/msa-tech-stacks.png)

微服务架构的关注点主要包括：配置管理（Config Management），API管理（API Management），服务发现（Service Discovery），负载均衡（Load Balance），数据管理（Data Management），日志聚合（Centralized ），指标聚合（Centralized Metrics），分布式追踪（Distributed Tracking），服务安全（Service Security），弹性和容错（Resilience & Fault Tolerance），调度部署（Deployment & Scheduling），自动伸缩与修复（Auto Scaling & Self Healing）等。



