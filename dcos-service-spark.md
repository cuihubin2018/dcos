## Spark on DC/OS

### 限定Spark Job在特定Agent节点上运行

首先为一组特定的节点[设置Agent节点的属性](/dcos-mesos-attributes-and-resources.md)，在执行Spark任务提交时，通过`spark.mesos.constraints`参数设置属性约束，如下示例：

```
dcos spark run --submit-args='-Dspark.mesos.constraints="worker:true" --driver-cores 1 --driver-memory 1024M --class org.apache.spark.examples.SparkPi https://downloads.mesosphere.com/spark/assets/spark-examples_2.10-1.4.0-SNAPSHOT.jar 100'
```

