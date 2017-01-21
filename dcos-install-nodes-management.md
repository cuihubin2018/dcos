## DC/OS集群节点管理

### 新增节点

#### 新增Master节点

当前版本（1.8.x）DC/OS在集群部署完成后不支持添加新的Master节点，因此，在构建集群时，必须根据业务规划提前确定好Master节点的数量，推荐至少3个节点，如果集群规模足够庞大，则可以增至5个节点。

#### 新增Agent节点

**准备工作**

1. 确保DC/OS集群在第一次完成安装时，[备份了集群安装文件](/dcos-install-backup-installer-file.md)。

2. 确保执行了DC/OS集群[安装环境准备](/dcos-install-default.md)工作中所有涉及Agent节点的操作。

3. 将Bootstrap节点上的SSH公钥拷贝到新增Agent节点主机的`/root/.ssh/authorized_keys文件。`


**操作步骤**

* 使用scp将集群安装文件拷贝的新增Agent节点主机
  ```
  scp dcos-install.tar  <username>@<new-agent-node-ip>:~/dcos-install.tar
  ```


* 登录到新增节点主机
  ```
  ssh <username>@<new-agent-node-ip>
  ```


* 在新增节点上创建DCOS临时安装目录
  ```
  sudo mkdir -p /opt/dcos_install_tmp
  ```


* 解压上传的集群安装文件
  ```
  sudo tar -xf dcos-install.tar -C /opt/dcos_install_tmp
  ```


* 执行下述命令添加节点  
  添加私有Agent节点：

  ```
  sudo bash /opt/dcos_install_tmp/dcos_install.sh slave
  ```

  添加公有Agent节点：

  ```
  sudo bash /opt/dcos_install_tmp/dcos_install.sh slave_public
  ```


* 通过下述两个命令可以分别查询当前集群中私有节点和公有节点的数量  
  私有：

  ```
  dcos node --json | jq --raw-output '.[] | select(.reserved_resources.slave_public == null) | .id' | wc -l
  ```

  公有：

  ```
  dcos node --json | jq --raw-output '.[] | select(.reserved_resources.slave_public != null) | .id' | wc -l
  ```

**注意：** 如果环境还对其它做了配置，必须要进行相应的初始化工作。如Docker服务对内部私有仓库的信任配置，具体可参考[私有容器仓库](/dcos-service-pre-private-docker-registry.md)。

**提示：** 如果新增节点系统环境与初始安装节点存在差异，如网卡名称与初始节点的名称不同（eth1 <--> em1)，导致IP侦测脚本无法获取到Agent节点的正确IP地址，可以创建新网卡并桥接到实际网卡的模式（仅推荐测试环境）。

### 删除节点

移除Agent节点前，首先SSH登录到Agent节点主机，执行下述命令从Mesos注销并清理停止的任务：

```
systemctl kill -s SIGUSR1 dcos-mesos-slave
```

然后按下述步骤删除该节点，具体操作如下：

```
sudo -i  /opt/mesosphere/bin/pkgpanda uninstall

sudo reboot

sudo rm -rf /opt/mesosphere /etc/mesosphere
```

如果节点删除过程中出现异常，要手工检查`/etc/systemd/system`目录下是否存在以“dcos”开头的服务，需要手动清除。

如果想完全清除节点上的所有集群相关的数据，需要再执行下述命令清理：

```
sudo rm -rf /var/lib/dcos /var/lib/mesos
```

### 变更节点类型



