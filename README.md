# DC\/OS step by step

A book about DC\/OS to record my own experience

安装前常见问题

Bootstrap节点是否必须？

在安装参数“”未设置成“”时，Bootstrap节点不是必须的。可以在安装完成后，将该节点纳入集群作为Agent节点。但是该节点在安装过程中生成集群的安装包，需要提前做好备份，该安装包可用于增加新的Agent节点。

Master节点需要部署几个？

开发测试环境中，1个master即可，master节点故障仅对新的服务部署及现有环境配置修改产生影响，不对当前正在运行的集群及服务造成影响。如果想确保开发测试环境的可靠性，推荐3个节点。

生成环境中，根据规模，3个节点或5个节点即可。Master节点数不是越多越好，Master节点数量与集群性能成反比。







