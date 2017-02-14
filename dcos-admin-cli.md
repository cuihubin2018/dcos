## DC/OS CLI

### 管理多个集群

通常我们会有开发、测试、生产环境（不同机房）等多个DC/OS集群，通过以下命令可以查看当前正在管理的集群：

```
dcos config show core.dcos_url
```

通过以下命令可以切换到其他集群：

```
$ dcos config set core.dcos_url http://192.168.83.11

[core.dcos_url]: changed from 'http://192.168.1.69' to 'http://192.168.83.11'
```


