## Marathon事件总线

Marathon有一个内部事件总线，用于捕获所有API请求并发布对应的事件。通过订阅这个事件总线，应用可以在事件触发时立即被通知而无需主动拉取。这个事件总线对于所有依赖Marathon的状态变化而采取相应处理的应用非常有用，如负载均衡可以根据Marathon发布的应用健康检查信息事件而在服务实例之间动态调度。

事件可以由可插拔（pluggable）的订阅者进行订阅。

事件总线有两个API：

- 事件流。有关详细信息，请参阅[Marathon REST API](https://mesosphere.github.io/marathon/docs/generated/api.html)参考中的`/v2 /events`条目。

-回调接口，以JSON格式将事件POST到一个或多个端点。

在实际使用时，推荐使用事件流而不是回调接口，因为：

- 更容易设置。

- 因为没有请求/响应周期，所以

- 事件是按顺序传播的。

### 参考

https:\/\/mesosphere.github.io\/marathon\/docs\/event-bus.html

