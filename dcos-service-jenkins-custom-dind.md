## Jenkins Slave环境维护

修改Jenkins Slave镜像

添加`VOLUME "/etc/docker"`,让动态创建的Jenkins Slave可以使用Agent主机上配置的私有仓库的SSL证书信息。

解决问题：`x509: certificate signed by unknown authority`

修改Jenkins Slave主机配置


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

