## Marathon事件总线

Marathon有一个内部事件总线，用于捕获所有API请求并发布对应的事件。通过订阅这个事件总线，应用可以在事件触发时立即被通知而无需主动拉取。这个事件总线对于所有依赖Marathon的状态变化而采取相应处理的应用非常有用，如负载均衡可以根据Marathon发布的应用健康检查信息事件而在服务实例之间动态调度。

事件可以由可插拔（pluggable）的订阅者进行订阅。

事件总线有两个API：

- 事件流。有关详细信息，请参阅[Marathon REST API](https://mesosphere.github.io/marathon/docs/generated/api.html)参考中的`/v2 /events`条目。

-回调接口，以JSON格式将事件POST到一个或多个端点。

在实际使用时，推荐使用事件流而不是回调接口，因为：

- 更容易设置。

- 因为没有请求/响应周期，所以速度更快。

- 事件是按顺序传播的。

### 通过回调接口订阅事件

```
./bin/start --master ... --event_subscriber http_callback --http_endpoints http://host1/foo,http://host2/bar
```
在上述代码示例中，通过命令行参数配置事件订阅的类型为`http_callback`，并通过`http_endpoints`指定两个回调接口，则主机host1和host2都能够接订阅到事件。

### 事件类型

#### API Request

每次Marathon接收到修改应用程序的API请求（创建，更新，删除）时触发：

```json
{
  "eventType": "api_post_event",
  "timestamp": "2014-03-01T23:29:30.158Z",
  "clientIp": "0:0:0:0:0:0:0:1",
  "uri": "/v2/apps/my-app",
  "appDefinition": {
    "args": [],
    "backoffFactor": 1.15,
    "backoffSeconds": 1,
    "cmd": "sleep 30",
    "constraints": [],
    "container": null,
    "cpus": 0.2,
    "id": "/my-app",
    "instances": 2,
    "mem": 32.0,
    "ports": [10001],
    "requirePorts": false,
    "storeUrls": [],
    "upgradeStrategy": {
        "minimumHealthCapacity": 1.0
    },
    "uris": [],
    "version": "2014-09-09T05:57:50.866Z"
  }
}
```
#### Status Update

每次任务的状态更改时触发：

```json
{
  "eventType": "status_update_event",
  "timestamp": "2014-03-01T23:29:30.158Z",
  "slaveId": "20140909-054127-177048842-5050-1494-0",
  "taskId": "my-app_0-1396592784349",
  "taskStatus": "TASK_RUNNING",
  "appId": "/my-app",
  "host": "slave-1234.acme.org",
  "ports": [31372],
  "version": "2014-04-04T06:26:23.051Z"
}
```

属性**`taskStatus`**的值为以下几种（最后四个状态是终止状态）：

- TASK_STAGING
- TASK_STARTING
- TASK_RUNNING
- TASK_FINISHED
- TASK_FAILED
- TASK_KILLED
- TASK_LOST

#### Framework Message

```json
{
  "eventType": "framework_message_event",
  "timestamp": "2014-03-01T23:29:30.158Z",
  "slaveId": "20140909-054127-177048842-5050-1494-0",
  "executorId": "my-app.3f80d17a-37e6-11e4-b05e-56847afe9799",
  "message": "aGVsbG8gd29ybGQh"
}
```

#### Event Subscription

添加或删除新的HTTP回调订阅时触发：

```json
{
  "eventType": "subscribe_event",
  "timestamp": "2014-03-01T23:29:30.158Z",
  "clientIp": "1.2.3.4",
  "callbackUrl": "http://subscriber.acme.org/callbacks"
}
```

### 参考

https:\/\/mesosphere.github.io\/marathon\/docs\/event-bus.html

