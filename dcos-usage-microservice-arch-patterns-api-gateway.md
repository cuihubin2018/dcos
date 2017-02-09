## API网关

API网关在网络拓扑中位于客户端和后端服务之间，当前互联网架构中，在这个位置扮演重要角色的通常是反向代理和负载平衡系统如Nginx，因此，扩展Nginx（结合LUA）实现API网关的职能也是一种常见的方案，项目[KONG](https://getkong.org/)正是基于这种思路实现的一个开源产品。

### KONG

### 整合微服务，Marathon-LB 和 KONG

通过订阅Marathon的事件，配合微服务部署时设定的LABEL信息，微服务实现可以将服务的REST接口描述（通过Swagger提供）注册到KONG。