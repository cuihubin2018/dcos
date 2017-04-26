## 容器网络

容器网络需要面对的最重要的一个问题是容器的IP地址。DC/OS的容器网络在Mesos V0.25版本（2015年10月）之前没有明确的方案，需要手动维护创建。在Mesos V0.25到V1.0（2016年7月）这段时期，Docker容器技术推出了网络扩展功能，Docker容器有了原生的网络扩展支持，典型的第三方插件有Weave，Calico等，此阶段可以通过Marathon的API在创建容器的同时赋予一个IP地址。在Mesos V1.0之后，Mesos原生支持了CNI网络，无论是Docker容器还是AppC容器，**IP-per-container**变得非常简单。

总之，Mesos支持_Mesos容器化_和_Docker容器化_两种容器运行时方案，这两种运行时都支持**IP-per-container**，允许容器绑定到不同的IP网络，不同的是两种运行时支持的IP-per-container的实现机制。Mesos容器化使用network/cni隔离器实现的[CNI](https://github.com/containernetworking/cni/blob/master/SPEC.md)为容器提供网络支持；Docker容器化依赖Docker Daemon通过Docker的[容器网络模型](https://github.com/docker/libnetwork)提供网络支持。

**IP-per-container**仅是为容器隔离提供网络支持方案中的一种，Mesos容器化还可以通过其它方案为容器提供网络支持，如：[端口映射隔离器](https://github.com/apache/mesos/blob/master/docs/port-mapping-isolator.md)。

Mesos容器化和Docker容器化在Mesos中使用相同的网络配置描述：

```
message NetworkInfo {
  enum Protocol {
    IPv4 = 1;
    IPv6 = 2;
  }

  message IPAddress {
    optional Protocol protocol = 1;
    optional string ip_address = 2;
  }

  repeated IPAddress ip_addresses = 5;
  optional string name = 6;
  repeated string groups = 3;
  optional Labels labels = 4;
};
```

此外，Docker容器化在描述网络配置的同时还需要对容器进行定义：

```
message DockerInfo {
  // The docker image that is going to be passed to the registry.
  required string image = 1;

  // Network options.
  enum Network {
    HOST = 1;
    BRIDGE = 2;
    NONE = 3;
    USER = 4;
  }

  optional Network network = 2 [default = HOST];
 };
```

在DC/OS中，通过Marathon管理容器（AppC或Docker）时，当前支持三种网络模式配置：

* **BRIDGE**

> 适用于Docker容器服务。在该模式下，容器端口（容器内部定义的端口）被映射为容器宿主机上对应的端口。在该模式下，容器中的应用绑定到容器的端口，Docker网络服务负责把这些端口绑定到主机的端口。

* **USER**

> 适用于Docker容器服务。在该模式下，容器端口（容器内部定义的端口）被映射为容器宿主机上对应的端口。在该模式下，容器中的应用绑定到容器的端口，Docker网络服务负责把这些端口绑定到主机的端口。该模式主要用于适配Docker自定义网络（通过Docker插件扩展实现），在Mesos中主要通过CNI插件结合Mesos CNI网络隔离组件来实现。

* **HOST**

> 同时适用于AppC容器应用和Docker容器应用。在该模式下，应用直接绑定到宿主机的一个或多个端口。

### VIPs

DC/OS提供了一个名为Minuteman的服务器端内部微服务之间的东-西向（区分于Client-Server的南-北向）4层负载均衡。为了易于服务的配置和发现，DC/OS采用**命名（name-based）VIPs**来定位服务。因此，客户端访问服务时连接的是一个服务地址而不是具体的IP地址，同时DC/OS可以很容易的将指向一个命名VIP的调用请求映射到多个具体的IP地址和端口，从而实现负载调度。采用命名VIPs的另一个好处是可以避免与基于IP的VIP产生冲突，在服务安装时可以自动创建。

一个命名VIP包含3个组成部分：

* 私有的虚拟IP地址

* 端口（通过该端口提供服务）

* 服务名称


VIPs的命名遵循如下规则：

`<service-name>.marathon.l4lb.thisdcos.directory:<port>`

根据Marathon所运行的是Docker容器镜像还是AppC容器镜像的不同，在Marathon应用定义中分别用`portMappings`和`portDefinitions`两个不同的属性节点进行配置。

**AppC容器配置**

```
{
    "id": "myservice",
    "portDefinitions": [ 
        { 
            "protocol": "tcp", 
            "port": 6666, 
            "labels": { "VIP_0": "myservice:6666" }, 
            "name": "jmx" 
        }, 
        { 
            "protocol": "tcp", 
            "port": 7777, 
            "labels": { "VIP_1": "myservice:7777" }, 
            "name": "api" 
        } 
    ]
}
```

如上示例，该配置定义了两个VIP：

* `myservice.marathon.l4lb.thisdcos.directory:6666`

* `myservice.marathon.l4lb.thisdcos.directory:7777`


客户端（注：此处指DC/OS集群中的其他服务）可以通过调用端口为6666的VIP访问服务提供的JMX管理功能，可以通过调用端口为7777的VIP访问服务提供的业务API。

上述配置也可以通过DC/OS的WEB管理控制台进行配置：

![](/assets/dcos_network_vip_appc.png)

**Docker容器配置**

```
{ 
    "id": "myservice", 
    "container": { 
        "docker": { 
            "image": "chrisrc/myservice", 
            "forcePullImage": false, 
            "privileged": false, 
            "portMappings": [ 
                { 
                    "containerPort": 6666, 
                    "protocol": "tcp", 
                    "name": "jmx", 
                    "servicePort": 6666, 
                    "labels": { "VIP_0": "myservice:6666" } 
                }, 
                { 
                    "containerPort": 7777, 
                    "protocol": "tcp", 
                    "name": "api", 
                    "servicePort": 7777, 
                    "labels": { "VIP_1": "myservice:7777" } 
                } 
            ], 
            "network": "BRIDGE" 
        } 
    }
}
```

如上示例，该配置定义了一个名为myservice的服务，该服务通过Docker镜像`chrisrc/myservice`提供。与上例类似，该服务也定义了两个客户端（注：此处指DC/OS集群中的其他服务）可以直接访问的VIP：

* `myservice.marathon.l4lb.thisdcos.directory:6666`

* `myservice.marathon.l4lb.thisdcos.directory:7777`


上述配置也可以通过DC/OS的WEB管理控制台进行配置：

![](/assets/dcos_network_vip_docker.png)

