## 高级安装

采用高级安装方式时，所有待装节点（Masters或Agents）到Bootstrap节点的网络必须是通畅的且Bootstrap节点的HTTP(s)端口必须对待装节点开放。完成DC/OS的安装后，待装节点上会出现下述列表中的目录或文件：

| 目录 | 说明 |
| --- | --- |
| **/opt/mesosphere** | 包含DC/OS系统的运行脚本，依赖库和集群配置。不建议修改。 |
| **/etc/systemd/system/dcos.target.wants** | 包含了用于启动组成DC/OS系统的所有Systemd服务。 |
| **/etc/systemd/system/dcos.<units>** | 这组文件是/etc/systemd/system/dcos.target.wants目录下的文件拷贝（链接），这些文件必须同时存在于Systemd的system目录和dcos.target.wants目录中。 |
|**/var/lib/zookeeper** | 包含ZooKeeper运行数据(默认情况下位于/var/lib/dcos/exhibitor/zookeeper下)。|
| **/var/lib/docker** | 包含容器镜像等Docker相关数据 |
| **/var/lib/dcos** | 包含DC/OS相关的数据 |
| **/var/lib/mesos** | 包含Mesos相关的数据|
| **/run/dcos** | 包含Pkgpanda及Metronome等服务相关的数据|



IP Detect

echo $\(ip addr \| grep "inet &lt;IP网段，如：192.168.83&gt;" \| awk '{print $2}' \| awk -F '\/' '{print $1}'\)

