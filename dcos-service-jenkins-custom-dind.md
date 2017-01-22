## Jenkins Slave环境维护

Jenkins on DC/OS 支持同时维护多种不同的Slave镜像环境，因此可以为Java，Node.js，Ruby等等构建不同的Slave主机环境镜像。这些镜像可以通过任务标签进行选择。

![](/assets/jenkins-dnd-image-labels.png)

### Jenkins Slave DnD镜像

- Maven构建环境

 https://github.com/christtrc/docker-jenkins-gitbook-agent

- Gitbook构建环境

 https://github.com/christtrc/docker-jenkins-gitbook-agent


### Docker私有仓库配置

Jenkins Slave主机以容器运行时，有些任务需要拉取私有仓库中的基础镜像并将编译后的镜像推送到私有仓库。因DnD模式的容器是动态构建的，如果私有仓库配置的是自签名证书，为了能够正确访问容器私有仓库，通常需要挂载Agent主机上的Docker配置。

![](/assets/jenkins-dnd-docker-ssl.png)

否则，在应用过程中可能会碰到`x509: certificate signed by unknown authority`的异常信息。

### Maven环境

当Jenkins的Job任务中存在Maven构建时，Maven会自动从中央仓库下载项目的依赖。由于DnD模式的容器是动态构建的，Maven环境位于DnD容器中，每次任务执行都需要下载Maven的仓库依赖，这会严重影响带宽及项目构建速度，因此必须将Maven仓库挂载在DnD容器运行的Agent宿主机的磁盘上。

另一个问题是DnD容器在DC/OS集群中是动态漂移的，所以选取的磁盘存储必须能够在所有Agent节点之间都能够共享访问，基本的方案是使用NFS存储，也可以选择采用GlusterFS，Ceph及其它SAN/SNS方案，参考：[存储策略与方案](/dcos-storage.md)。

![](/assets/jenkins-dnd-maven-repo-external.png)

### SBT环境

与Jenkins Maven编译环境类似，因为执行Job任务时，每次都重新构建（或重用未释放的）Jenkins Slave环境，对于项目依赖的一些构件，如果不采用外部存储，构建任务往往会每次全量下载所有的依赖，导致很长的构建延时。为了解决此问题，需要对Jenkins DnD镜像设置添加SBT构件缓存路径的外部卷挂载。

![](/assets/jenkins-dnd-sbt-volume.png)

此时，可以将SBT仓库的配置文件repositories添加到Agent主机的/data/sbt目录下，根据需要随时调整，如：

```
[repositories]
  local
  aliyun:http://maven.aliyun.com/nexus/content/groups/public
  maven-local
  typesafe-ivy:https://dl.bintray.com/typesafe/ivy-releases/, [organization]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)[revision]/[type]s/[artifact](-[classifier]).[ext]
  clever-nexus-proxy-releases: http://192.168.1.54:8081/nexus/content/groups/public
```


