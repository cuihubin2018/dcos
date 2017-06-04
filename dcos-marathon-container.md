## 容器运行管理

Marathon通过Docker引擎和通用容器运行时支持Docker和AppC容器镜像的运行。

Marathon支持两种运行Docker容器镜像的方式：

* Docker容器化，使用Docker引擎运行容器镜像。

* Mesos容器化，使用通用容器运行时\(UCR\)运行容器镜像。

### Docker容器化

Docker容器化依赖外部Docker引擎来运行Docker容器镜像。采用Docker引擎运行Docker容器镜像时，需要对Agent节点及Marathon的配置做一些调整。

#### 运行参数配置

注意，DCOS在部署时已针对Docker容器化做了参数调整，因此，下述配置仅适用于独立运行Marathon服务场景。下述参数在DCOS中同样可以找到，只是位置稍有差别。

1.调整Agent节点配置，提高Docker容器化的优先级。注意，此处配置的值顺序决定了容器运行时的选择，靠前的优先。

```
$ echo 'docker,mesos' > /etc/mesos-slave/containerizers
```

2.增加执行程序超时设置，以应对因网络带宽等因素将docker镜像拉取到Agent节点的潜在延迟。

```
$ echo '10mins' > /etc/mesos-slave/executor_registration_timeout
```

上述两项配置在DC/OS中对应的位置为\(Agent节点\)：`/opt/mesosphere/etc/mesos-slave-common。`

3.调整Marathon的启动参数`--task_launch_timeout`（在杀死一个任务之前等待其进入`TASK_RUNNING`的时间，毫秒，默认值为30000），使其匹配Agent节点执行程序超时设置。在**DC/OS**中，该启动参数通过变量`MARATHON_TASK_LAUNCH_TIMEOUT`进行设置，参数位于\(Master节点\)：`/opt/mesosphere/etc/marathon`。

说明：Marathon的命令行参数都对应一个添加前缀“`MARATHON_`”的变量。

#### 容器应用定义示例

为了让容器镜像采用Docker容器化运行，需要在应用定义JSON中添加container节点配置。如下示例：

```json
{ 
    "container": { 
        "type": "DOCKER", 
        "docker": { 
            "network": "HOST", 
            "image": "group/image" 
        }, 
        "volumes": [ 
            { "containerPath": "/etc/a", "hostPath": "/var/data/a", "mode": "RO" }, 
            { "containerPath": "/etc/b", "hostPath": "/var/data/b", "mode": "RW" } 
        ] 
    }
}
```

此处volumes和type为可选配置，type的默认值为`DOCKER`。承载容器镜像运行的Mesos沙箱的系统挂载点可以通过环境变量`$MESOS_SANDBOX`获取。

#### 桥接网络模式

桥接网络使得与容器内静态配置的端口进行绑定的应用的运行变得容易。Marathon负责Mesos管理的端口资源和Docker绑定的主机端口间的对接。

**动态端口绑定**

```json
{ 
    "id": "bridged-webapp", 
    "cmd": "python3 -m http.server 8080", 
    "cpus": 0.5, "mem": 64.0, "instances": 2, 
    "container": { 
        "type": "DOCKER", 
        "docker": { "image": "python:3", "network": "BRIDGE", 
            "portMappings": [ 
                { "containerPort": 8080, "hostPort": 0, "servicePort": 9000, "protocol": "tcp" }, 
                { "containerPort": 161, "hostPort": 0, "protocol": "udp"} 
            ] 
        } 
    }, 
    "healthChecks": [ 
        { "protocol": "HTTP", "portIndex": 0, 
          "path": "/", "gracePeriodSeconds": 5, 
          "intervalSeconds": 20, "maxConsecutiveFailures": 3 } 
    ] 
}
```

示例中的端口定义再简单总结一下，详细信息可参考[**服务端口配置**](/dcos-network-marathon-ports.md)章节。

1）**hostPort**，值为0（默认值）时，是一个在Agent主机上随机分配的端口，该端口属于Mesos管理的端口资源的一部分。该配置可选。

2）**containerPort**，是指应用在容器内监听的端口。该配置可选且默认值为0。取默认值0时，Marathon将该端口设置为与hostPort相同的值。这要求容器内应用通过获取传递进容器的环境变量$PORT0，$PORT1，...设定应用的启动端口。

3）**servicePort**，是一个用于服务发现的辅助配置，Marathon并不实际使用该端口配置，但它会确保该端口在整个集群内唯一。该配置可选且默认值为0。取默认值0时，端口被随机分配，范围介于`[local_port_min, local_port_max]`之间，默认值为`10000~20000`。

**静态端口绑定**

如果为hostPort设置了非默认值0，必须确保设定的端口值处于Mesos资源提供的范围内。默认情况下，Mesos的Agent节点提供的端口资源范围是`[31000-32000]`，该默认值可以通过参数调整：

```
--resources="ports(*):[8000-9000, 31000-32000]"
```

DCOS默认设置的端口资源范围\(`/opt/mesosphere/etc/mesos-slave` \)为：

```json
{
    "name":"ports",
    "type":"RANGES",
    "ranges": {
        "range": [
            {"begin": 1025, "end": 2180},
            {"begin": 2182, "end": 3887},
            {"begin": 3889, "end": 5049},
            {"begin": 5052, "end": 8079},
            {"begin": 8082, "end": 8180},
            {"begin": 8182, "end": 32000}
        ]
    }
}
```

### 访问私有Docker仓库

#### Registry 1.0 - Docker pre 1.6

在应用JSON定义的uris字段里添加一个**.dockercfg**文件路径配置。Docker的`$HOME`环境变量会设置为`$MESOS_SANDBOX`，因此可以自动获取docker配置文件。

#### Registry 2.0 - Docker 1.6 and up

在应用JSON定义的uris字段里添加一个`docker.tar.gz`文件路径配置。`docker.tar.gz`文件需要包含`.docker`目录及该目录下的`.docker/config.json`。

**步骤1：制作docker.tar.gz**

1）登录到容器镜像仓库，登录后自动在用户目录下创建了一个`.docker`和`.docker/config.json`文件。

```
$ docker login some.docker.host.com 
    Username: foo 
    Password: 
    Email: foo@bar.com
```

2）压缩`.docker`目录

```
$ cd ~ 
$ tar czf docker.tar.gz .docker
```

3）检查压缩包的正确性

```
$ tar -tvf ~/docker.tar.gz 
    drwx------ root/root 0 2015-07-28 02:54 .docker/ 
    -rw------- root/root 114 2015-07-28 01:31 .docker/config.json
```

4）将`.docker.tar.gz`放置到一个Mesos/Marathon可访问的位置（该位置必须被所有节点访问到）

```
$ cp docker.tar.gz /etc/
```

**步骤2：在应用JSON中设置.docker.tar.gz的文件路径**

```json
{ 
    "id": "/some/name/or/id", 
    "cpus": 1, 
    "mem": 1024, 
    "instances": 1, 
    "container": { 
        "type": "DOCKER", 
        "docker": { 
            "image": "some.docker.host.com/namespace/repo", 
            "network": "HOST" 
        } 
    }, 
    "uris": [ "file:///etc/docker.tar.gz" ] 
}
```

应用部署时，Docker会根据文件提供的身份凭据登录镜像仓库拉取镜像。

### 高级参数配置

#### 强制拉取镜像

Marathon 0.8.2 和 Mesos 0.22.0版本开始支持在启动任务时强制拉取（即使本地已存在）容器镜像：

```json
{ "type": "DOCKER", "docker": { "image": "group/image", "forcePullImage": true } }
```

#### **Command vs Args**

Marathon自0.7.0版本开始在应用的JSON定义中支持args字段。在同一应用JSON定义中不能同时出现cmd和args。cmd字段在任务启动时以“`/bin/sh -c '${app.cmd}'`”执行。  
args字段提供了一种与容器定义进行交互的方式，如自定义Docker容器中的**ENTRYPOINT**：

```
FROM busybox 
MAINTAINER support@mesosphere.io 
CMD ["inky"] 
ENTRYPOINT ["echo"]
```

假定需要在上述定义的容器启动时向其传递额外的自定义参数，则可以如下定义JSON：

```json
{ 
    "id": "inky", 
    "container": { 
        "docker": { 
            "image": "mesosphere/inky" 
        }, 
        "type": "DOCKER", 
        "volumes": [] 
    }, 
    "args": ["hello"], 
    "cpus": 0.2, 
    "mem": 32.0, 
    "instances": 1 
}
```

也可以向容器传递复杂的参数：

```json
"args": [ "--name", "etcd0", "--initial-cluster-state", "new" ]
```

#### 特权模式和自定义Docker选项

Marathon 自0.7.6版本开始，新增了两个字段：`privileged` 和 `parameters`。privileged字段允许用户以特权模式运行容器，默认值为false。parameters字段允许用户为`docker run`提供自定义的命令行参数，使用该特性时要注意，随着Mesos容器化的发展，当Mesos与Docker引擎的交互不通过CLI方式时，该特性可能将不再被支持。

```json
{ 
    "id": "privileged-job", 
    "container": { 
        "docker": { 
            "image": "mesosphere/inky", 
            "privileged": true, 
            "parameters": [ 
                { "key": "hostname", "value": "a.corp.org" }, 
                { "key": "volumes-from", "value": "another-container" }, 
                { "key": "lxc-conf", "value": "..." }     
            ] 
        }, 
        "type": "DOCKER", 
        "volumes": [] 
    }, 
    "args": ["hello"], 
    "cpus": 0.2, "mem": 32.0, "instances": 1 
}
```

### Mesos容器化

Marathon自1.3.0版本开始不再依赖原生的Docker引擎来运行docker容器镜像。Mesos容器化使用通用容器运行时UCR（2016年7月发布的Mesos 1.0版本）来运行docker容器镜像。通用容器运行时UCR直接使用原生的操作系统特性配置和启动Docker或[AppC](https://github.com/appc/spec)容器镜像并提供隔离。

Mesos容器化提供了新的属性字段“credential”定义访问容器镜像仓库的身份凭据：

```json
{ 
    "id": "mesos-docker", 
    "container": { 
        "docker": { 
            "image": "mesosphere/inky", 
            "credential": { "principal": "alice", "secret": "wonderland" } 
        }, 
        "type": "MESOS" 
    }, 
    "args": ["hello"], 
    "cpus": 0.2, 
    "mem": 16.0, 
    "instances": 1 
}
```

在Mesos容器化（**type:MESOS**）中，Mesos使用CURL从容器仓库拉取Docker镜像。CURL默认会校验服务器端SSL证书，DC/OS中的CURL使用CAfile（而不是CApath）来查找信任的证书列表。默认CAfile的位置为：

```
/opt/mesosphere/active/python-requests/lib/python3.5/site-packages/requests/cacert.pem
```

如果在实际环境中使用了自签名的证书构建的私有容器仓库，则需要将签名证书添加到所有Agent节点上CURL的证书信任列表中。cacert.pem存放的证书是PEM格式，对于crt格式的证书，可通过下面的命令进行转换：

```
openssl x509 -in ca.crt -out ca.pem -outform PEM
```



