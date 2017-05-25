## 节点维护

在日常运维过程中，运维人员经常需要对DC/OS集群中运行的Agent节点进行维护，大多数情况，这些工作不会对正在运行的任务产生影响，但仍然存在一些场景会影响正在运行的任务，如：硬件修复，系统内核升级，Agent节点升级（调整Agent节点的[属性和资源](/dcos-mesos-attributes-and-resources.md)）。



- Example Maintenance Schedule JSON: 
https://github.com/vishnu2kmohan/dcos-toolbox/blob/master/mesos/agent-maintenance-schedule-example.json 

- Put agents into maintenance: 
https://github.com/vishnu2kmohan/dcos-toolbox/blob/master/mesos/maintain-agents.sh

- Example Machine Down JSON: 
https://github.com/vishnu2kmohan/dcos-toolbox/blob/master/mesos/agent-up-down-example.json

- Bring agents down: 
https://github.com/vishnu2kmohan/dcos-toolbox/blob/master/mesos/down-agents.sh 

- Bring agents back up: 
https://github.com/vishnu2kmohan/dcos-toolbox/blob/master/mesos/up-agents.sh

### 参考

- https://mesos.apache.org/documentation/latest/maintenance/

