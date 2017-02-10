## API网关

API网关在网络拓扑中位于客户端和后端服务之间，当前互联网架构中，在这个位置扮演重要角色的通常是反向代理和负载平衡系统如Nginx，因此，扩展Nginx（结合LUA）实现API网关的职能也是一种常见的方案，Mashape公司开发的[KONG](https://getkong.org/)项目正是基于这种思路实现的一个开源产品。

### KONG

KONG是一个通过[lua-nginx-module](https://github.com/openresty/lua-nginx-module)实现的在Nginx中运行的Lua应用程序。KONG没有使用lua-nginx-module直接编译Nginx，而是基于[OpenResty](https://openresty.org/)，与已经包含了lua-nginx-module的OpenResty一起分发。

KONG支持用Cassandra和PostgreSQL作为数据存储服务器，负载存储来自Kong操作的数据，同时，借助于Nginx，OpenResty和Lua插件体系，KONG作为API网关具备了高性能，高扩展性和灵活性。

![](/assets/kong-architecture.png)

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

#### 配置管理

Mashape官方为KONG提供了商业化的在线监控分析工具Galileo和在线API开发工具Gelato。

Github上也存在一些第三方开发的工具，涉及图形化配置管理的有[Django Kong Admin](https://github.com/vikingco/django-kong-admin)，[Jungle](https://github.com/rsdevigo/jungle)，[Kong Dashboard](https://github.com/PGBI/kong-dashboard)等，下面简要介绍**Kong Dashboard**的功能。

Kong Dashboard是用Javascript实现的，可以通过NPM或Docker等方式方便的安装和启动：

**NPM方式：**

```
# Install Kong Dashboard
npm install -g kong-dashboard

# Start Kong Dashboard
kong-dashboard start

# To start Kong Dashboard on a custom port
kong-dashboard start -p [port]

# To start Kong Dashboard with basic auth
kong-dashboard start -a user=password

# You can set basic auth user with environment variables
# Do not set -a parameter or this will be overwritten
set kong-dashboard-name=admin && set kong-dashboard-pass=password && kong-dashboard start
```

**Docker方式：**

```
# Start Kong Dashboard
docker run -d -p 8080:8080 pgbi/kong-dashboard

# Start Kong Dashboard on a custom port
docker run -d -p [port]:8080 pgbi/kong-dashboard

# Start Kong Dashboard with basic auth
docker run -d -p 8080:8080 pgbi/kong-dashboard npm start -- -a user=password
```
通过浏览器打开，可以看到如下界面：

![](/assets/kong-dashboard-home.png)

通过界面可以方便的管理API，用户和插件。

管理API：

![](/assets/kong-dashboard-apis-mgt.png)

添加API：

![](/assets/kong-dashboard-apis-add.png)

管理用户：

![](/assets/kong-dashboard-consumers.png)

添加用户：

![](/assets/kong-dashboard-consumer-edit.png)

管理插件：

![](/assets/kong-dashboard-apis-plugin.png)


### API网关 和 DC/OS

![](/assets/dcos-api-gateway-deployments.png)

如上图所示，API网关与DC/OS的整合存在两种模式，在模式1中，API网关独立于DC/OS集群之外，Marathon-LB部署于公开节点上作为外部负载均衡服务；在模式2中，API网关位于在DC/OS集群内部，部署在**公开节点**上（也可以部署在**私有节点**上，此时需要额外的Marathon-LB作为**外部**负载均衡服务），Marathon-LB作为**内部**负载均衡服务。

#### KONG in DC/OS

KONG作为API网关与DC/OS集群的整合既可以按上述模式1方式部署也可以按模式2进行。按模式2部署时，API网关是单实例覆盖全业务还是按业务进行实例拆分也可以根据实际需求进行调整。

下述步骤按单实例覆盖全业务的模式进行部署实践，其他场景可以根据实际需要调整。





#### 服务自动注册

通过订阅Marathon的事件，配合微服务部署时设定的LABEL信息，微服务实现可以将服务的REST接口描述（通过Swagger提供）注册到KONG。