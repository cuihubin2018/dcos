## 高级安装

采用高级安装方式时，所有待装节点（Masters或Agents）到Bootstrap节点的网络必须是通畅的且Bootstrap节点的HTTP(s)端口必须对待装节点开放。完成DC/OS的安装后，待装节点上会出现下述列表中的目录或文件：

| 目录 | 说明 |
| --- | --- |
| **/opt/mesosphere** | 包含DC/OS系统的运行脚本，依赖库和集群配置。不建议修改。 |
| **/etc/systemd/system/dcos.target.wants** | 包含了用于启动组成DC/OS系统的所有Systemd服务。 |
| **/etc/systemd/system/dcos.[units]** | 这组文件是/etc/systemd/system/dcos.target.wants目录下的文件拷贝（链接），这些文件必须同时存在于Systemd的system目录和dcos.target.wants目录中。 |
|**/var/lib/zookeeper** | 包含ZooKeeper运行数据(默认情况下位于/var/lib/dcos/exhibitor/zookeeper下)。|
| **/var/lib/docker** | 包含容器镜像等Docker相关数据 |
| **/var/lib/dcos** | 包含DC/OS相关的数据 |
| **/var/lib/mesos** | 包含Mesos相关的数据|
| **/run/dcos** | 包含Pkgpanda及Metronome等服务相关的数据|

### 安装前准备

#### 生成genconf目录

在Bootstrap节点上创建名为“`genconf`”的目录：

```
$ mkdir -p genconf
```

#### 设置自定义安装配置

在genconf目录下创建一个配置文件：`genconf/config.yaml`。

在这个YAML配置文件中根据实际环境自定义DC/OS集群的安装配置参数。下述是具有三个Master节点的DC/OS集群，Master节点使用内部管理的Exhibitor/ZooKeeper处理Quorum，集群内部配置了两个DC/OS虚拟网络，两个私有Agent节点并使用Google的DNS解析域名。

```yaml
agent_list:
    - <agent-private-ip-1>
    - <agent-private-ip-2>
    - <agent-private-ip-3>
# 如果待装节点通过HTTP下载安装包，则“/”指向genconf/serve目录.
bootstrap_url: http://192.168.88.10:8080
cluster_name: <cluster-name>
master_discovery: static
master_list:
    - <master-private-ip-1>
    - <master-private-ip-2>
    - <master-private-ip-3>
resolvers:
    # 此处可以配置内部DNS解析器，最多支持3个.
    - 8.8.4.4
    - 8.8.8.8
ssh_port: 22
ssh_user: centos
dcos_overlay_enable: true
dcos_overlay_mtu: 9001
dcos_overlay_config_attempts: 6
dcos_overlay_network:
      vtep_subnet: 44.128.0.0/20
      vtep_mac_oui: 70:B3:D5:00:00:00
      overlays:
        - name: dcos
          subnet: 9.0.0.0/8
          prefix: 26
        - name: dcos-1
          subnet: 192.168.0.0/16
          prefix: 24
```

关于安装模板自定义参数的详细信息，请参考[高级安装配置](/dcos-install-by-advanced-mode/dcos-install-by-advanced-mode-config.md)

#### 构建IP Detect脚本

```
echo $\(ip addr \| grep "inet &lt;IP网段，如：192.168.83&gt;" \| awk '{print $2}' \| awk -F '\/' '{print $1}'\)
```

**注意**：在节点上安装DC/OS后，节点的IP地址不得更改。例如，重新启动节点或更新DHCP时，IP地址不得更改。如果节点的IP地址更改，则必须移除并重新安装节点。

### 安装DC/OS

下述步骤会在Bootstrap节点上构建自定义的DC/OS编译文件，然后登录集群中的各个节点安装DC/OS。

提示：如果安装过程中出现问题，可以遵循集[群节点管理](/dcos-install-nodes-management.md)中的步骤移除节点然后重新尝试安装。

#### 下载DC/OS Installer

```
curl -O https://downloads.dcos.io/dcos/stable/dcos_generate_config.sh
```
#### 生成自定义集群安装包

在Bootstrap节点上执行下述命令生成自定义的安装文件。该命令从一个通用的DC/OS安装镜像中提取安装文件并根据前述准备的YAML配置及IP侦测脚本生成自定义的集群安装文件，生成的文件位于`./genconf/serve/`。

```
sudo bash dcos_generate_config.sh
```

命令执行完成后，目录结构如下：

```
├── dcos-genconf.<HASH>.tar
├── dcos_generate_config.sh
├── genconf
│   ├── config.yaml
│   ├── ip-detect
```

#### 通过HTTP发布serve目录

通过Nginx容器镜像或其它手段将`./genconf/serve/`目录通过HTTP开放给待装节点访问。

```
sudo docker run -d -p <your-port>:80 -v $PWD/genconf/serve:/usr/share/nginx/html:ro nginx
```
#### 安装Master节点

在所有Master节点上执行下述命令。

1. 登录到Master节点

  ```
ssh <master-ip>
```

2. 创建一个新目录并跳转到该目录下

  ```
  mkdir /tmp/dcos && cd /tmp/dcos
  ```

3. 从Bootstrap节点下载生成的安装文件，此处`http://<bootstrap-ip>:<your_port>`与YAML文件中的`bootstrap_url`的配置相同。

  ```
  curl -O http://<bootstrap-ip>:<your_port>/dcos_install.sh
  ```

4. 执行下述命令安装Master。

  ```
  sudo bash dcos_install.sh master
  ```

#### 安装Agent节点

1. 登录到Master节点

```
ssh <agent-ip>
```

2. 创建一个新目录并跳转到该目录下

```
mkdir /tmp/dcos && cd /tmp/dcos
```

3. 从Bootstrap节点下载生成的安装文件，此处`http://<bootstrap-ip>:<your_port>`与YAML文件中的`bootstrap_url`的配置相同。

```
curl -O http://<bootstrap-ip>:<your_port>/dcos_install.sh
```

4. 执行下述命令安装私有Agent或公有Agent。

  Private Agents:
  
  ```
sudo bash dcos_install.sh slave
```

  Public Agents:
  
 ```
sudo bash dcos_install.sh slave_public
```

#### 登录Exhibitor控制台确认ZK服务正常

```
http://<master-ip>:8181/exhibitor/v1/ui/index.html
```

#### 登录DC/OS WEB UI查看集群状态

```
http://<master-node-public-ip>/
```
