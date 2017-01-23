## FAQ

### DCOS集群的DNS域名服务器变更后，如何重新配置DCOS集群？

修改 `/opt/mesosphere/etc/dns_config` （最多只支持[__3个DNS地址__](https://docs.mesosphere.com/1.8/administration/installing/custom/configuration-parameters/#resolvers)）

重启 `dcos-spartan.service`

在所有节点上重复上述步骤



### DC/OS系统出现问题的排查思路

1. 首先确保`ip-detect`脚本的正确性并且在`config.yaml`中配置的DNS解析正常工作。安装过程中出现的问题有95%的占比是因为这两个配置项存在问题而导致的。

  可以在集群所有节点上手动执行`ip-detect`脚本或者在已安装了DC/OS的节点上通过`/opt/mesosphere/bin/detect_ip`命令检查返回的IP地址是否有效，例如，返回结果不应含有额外新的行，不能含有空格，不能含有其它特殊的或不可见的字符。

  尽量在[**`https://dcos.io/docs/1.8/administration/installing/custom/advanced/`**](https://dcos.io/docs/1.8/administration/installing/custom/advanced)示例的基础上构建自己的_`ip-detect`_。

  就DNS解析而言，理想情况下，FQDN，短主机名和IP地址的正向和反向查找都应该正常工作，虽然可以在没有有效DNS支持的环境中运行DC/OS，但以下内容**必须**工作（尤其适用于Spark和其他框架）：1)`hostname -f`必须返回FQDN；2）`hostname -s`必须返回短主机名；3）检查所有节点上的`hostnamectl`输出是正常的。

  通常情况下，在转向Agent节点排查问题之前，应首先按照以下流程检查Master节点上服务确保启动并且正常：

  ```
Exhibitor -> Mesos Master -> Mesos-DNS -> Spartan -> {Marathon, Metronome, Adminrouter}
```

2. 确保正在使用的防火墙或其它连接过滤机制没有干扰集群组件之间的通信。TCP，UDP和ICMP通信都被允许。

3. 检查Exhibitor服务是否启动并正确配置`http://<PICK_A_MASTER_IP>:8181/exhibitor，否则，检查Exhibitor服务日志`journalctl -flu dcos-exhibitor`。

  确保`/tmp`**没有**使用`noexec`标志挂载，否则，由于Java JNI因没有权限执行其在`/tmp`下创建的一个文件而导致Exhibitor无法启动Zookeeper（修复：`mount -o remount,exec /tmp`）。

  检查`/exhibitor/v1/cluster/status`接口的输出，确保Master节点的数量与预期一致，并且所有Master节点都处于“`serving`”状态而且只有一个Master节点的"`isLeader`"属性值为**true**，例如：

  SSH登录到其中的一个Master节点，获取下述类似信息：

  ```
curl -fsSL http://localhost:8181/exhibitor/v1/cluster/status | python -m json.tool
[ 
    { "code": 3, "description": "serving", "hostname": "10.0.6.70", "isLeader": false }, 
    { "code": 3, "description": "serving", "hostname": "10.0.6.69", "isLeader": false }, 
    { "code": 3, "description": "serving", "hostname": "10.0.6.68", "isLeader": true }
]
```

  注意，在采用多Master节点（3/5）部署时可能需要等待一段时间（大约10-15分钟）才能确认上述状态。如果过了这段时间仍无法确认，你需要通过`journalctl -flu dcos-exhibitor`仔细梳理Exhibitor的日志输出查找问题。

4. 假设Exhibitor正常启动，检查在集群节点上是否可以**ping**通`ready.spartan`，否则，通过`journalctl -flu dcos-spartan`检查**Spartan**服务的日志。

5. 假设Spartan服务正常启动而且可以ping通`ready.spartan`，检查是否可以ping通`leader.mesos` 和 `master.mesos`，否则，通过`journalctl -flu dcos-mesos-dns`检查**Mesos-DNS**服务的日志。

6. 如果可以ping通`ready.spartan`但无法ping通`leader.mesos`，应该检查 Mesos Master的服务日志：`journalctl -flu dcos-mesos-master`，因为在Mesos-DNS可以从`/state`生成Master节点的DNS记录之前，Mesos的Master节点必须先选举出Leader。

说明：上述步骤来自DC/OS在Slack上的讨论。





