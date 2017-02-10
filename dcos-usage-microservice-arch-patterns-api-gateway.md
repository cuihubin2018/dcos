## API网关

API网关在网络拓扑中位于客户端和后端服务之间，当前互联网架构中，在这个位置扮演重要角色的通常是反向代理和负载平衡系统如Nginx，因此，扩展Nginx（结合LUA）实现API网关的职能也是一种常见的方案，项目[KONG](https://getkong.org/)正是基于这种思路实现的一个开源产品。

### KONG

KONG是一个通过[lua-nginx-module](https://github.com/openresty/lua-nginx-module)实现的在Nginx中运行的Lua应用程序。KONG没有使用lua-nginx-module直接编译Nginx，而是基于[OpenResty](https://openresty.org/)，与已经包含了lua-nginx-module的OpenResty一起分发。

KONG支持用Cassandra和PostgreSQL作为数据存储服务器，负载存储来自Kong操作的数据，同时，借助于Nginx，OpenResty和Lua插件体系，KONG作为API网关具备了高性能，高扩展性和灵活性。

KONG提供了API支持用户自定义插件扩展，并实现了众多的插件支持API网关的职能：

- 认证
  HTTP基本认证，密钥认证，OAuth，JWT，HMAC和LDAP

- 安全
  ACL，CORS（ Cross-origin Resource Sharing，跨域资源共享），动态SSL，IP限定和机器人侦测
  
- 流量控制
  请求限流，请求响应限流，请求载荷限定
  
- 分析与监控
  Galileo，Datadog，Runscope
  
- 数据转换
  请求转换，响应转换，请求/响应关联
  
- 日志记录
  TCP，UDP，HTTP，文件日志，Syslog，StatsD和Loggly。
  
#### 安装部署

KONG提供了多种部署方式，支持Docker容器部署，AWS部署，CentOS/RedHat，Debian/Ubuntu，Heroku，OSX，Vagrant和源代码编译部署。

以Docker容器部署为例：

1. 启动数据库（这里使用Cassandra，使用PostgreSQL时请参考官方文档）：

  ```
$ docker run -d --name kong-database \
              -p 9042:9042 \
              cassandra:2.2
```

2. 启动KONG服务实例：

  ```
$ docker run -d --name kong \
              --link kong-database:kong-database \
              -e "KONG_DATABASE=cassandra" \
              -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
              -p 8000:8000 \
              -p 8443:8443 \
              -p 8001:8001 \
              -p 7946:7946 \
              -p 7946:7946/udp \
              kong
```

3. 检查KONG是否正常启动：

  ```
$ curl http://127.0.0.1:8001
```

4. 使用KONG，可参考官方**[5分钟快速上手示例](https://getkong.org/docs/latest/getting-started/quickstart)**。

### API网关 和 DC/OS

![](/assets/dcos-api-gateway-deployments.png)

如上图所示，API网关与DC/OS的整合存在两种模式，在模式1中，API网关独立于DC/OS集群之外，Marathon-LB部署于公开节点上作为外部负载均衡服务；在模式2中，API网关位于在DC/OS集群内部，部署在**公开节点**上（也可以部署在**私有节点**上，此时需要额外的Marathon-LB作为**外部**负载均衡服务），Marathon-LB作为**内部**负载均衡服务。

#### KONG in DC/OS

KONG作为API网关与DC/OS集群的整合既可以按上述模式1方式部署也可以按模式2进行。按模式2部署时，API网关是单实例覆盖全业务还是按业务进行实例拆分也可以根据实际需求进行调整。

下述步骤按单实例覆盖全业务的模式进行部署实践，其他场景可以根据实际需要调整。





#### 服务自动注册

通过订阅Marathon的事件，配合微服务部署时设定的LABEL信息，微服务实现可以将服务的REST接口描述（通过Swagger提供）注册到KONG。