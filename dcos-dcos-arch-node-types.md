## 节点类型

DC/OS节点是运行DC/OS系统组件的虚拟或物理主机，这些节点通过网络连接在一起以形成DC/OS集群。

![](/assets/dcos-node-types.png)

### Master节点

DC/OS的Master节点与其它Master节点一起协同管理集群的其它所有节点。Master节点运行着大量的DC/OS系统组件，包括Mesos Master进程。

- 安全

  尽管Master节点可以被配置为公开访问，但为了安全考虑，通常运行在一个受控区域，仅允许内部访问。

- 高可靠性

  多个Master节点协同工作，以提供高可用性和容错。
  
- Leader选举

  Mesos执行Leader选举并将输入请求路由到当前Leader以确保一致性。
  
  与Mesos类似，DC/OS的Master节点也采用独立的Leader选举。这也意味着位于Master节点上的Marathon和Zookeeper服务的Leader与Master节点的Leader可能不在同一个Master节点上。
  
- Quorum

 为保持一致性，必须确保一半加一个Master节点同时在线。例如，如果有三个Master，仅允许一个掉线；如果有五个Master，仅允许两个同时掉线。 
 
 当前仅支持安装时设定Master节点的个数，安装后不能改变。在未来版本中这种限制将会被消除。
 
### Agent节点

DC/OS的Agent节点是承载用户服务运行的节点。Agent节点仅包含部分DC/OS系统组件，其中包括Mesos Agent进程。

根据网络配置的不同，Agent节点可以是公有或私有的。

#### 公有Agent节点

默认情况下，公有Agent节点上的资源配置为仅分配给指定了`slave_public`角色的任务。公有Agent节点上的Mesos Agent服务的`public_ip`属性参数被设置为：**true**。

公有Agent节点通常位于**DMZ**区域。用于反向代理及负载均衡器职能的Marathon-LB通常位于公有Agent节点上。

#### 私有Agent节点

DC/OS集群中的私有Agent节点不能被DC/OS集群外的网络访问。默认情况下，私有Agent节点上的资源配置为仅分配给指定了`*`角色的任务。
