## Secrets管理

现代系统需要访问大量的敏感信息如：数据库访问凭证，用于外部服务的API密钥，面向服务架构的通信凭据等，这使得了解谁正在访问什么敏感信息变得非常困难，并且往往是特定于某个平台。如果没有定制的解决方案，为系统添加密钥滚动切换，安全存储和详细的审计日志几乎是不可能的。

### Vault

HashiCorp的[Vault](https://www.vaultproject.io)是一个安全访问Secrets的工具。Secrets是需要严格控制访问权限的任何内容，例如API密钥，密码，证书等。Vault为任何secret提供统一的接口，同时提供严格的访问控制并记录详细的审计日志。

其主要特性包括：

- 安全的Secrets存储

  任意键/值Secrets可以存储在Vault中。Vault在将这些Secrets写入永久存储器之前对其加密，因此访问原始存储器无法直接访问您的敏感信息。Vault可以将加密数据写入磁盘，[Consul](https://www.consul.io/)等等。

- 动态的Secrets

  Vault可以为某些系统（如AWS或SQL数据库）动态生成访问凭据。例如，当应用程序需要访问MySQL数据库时，它会向Vault询问凭据，Vault将根据需要生成具有有效权限的MySQL数据库访问密钥对。创建这些动态访问凭据之后，Vault还会在租期结束后自动撤销这些凭据。

- 数据加密

  Vault可以加密和解密数据而不存储。这允许安全团队定义加密参数，开发人员负责将加密的数据存储在诸如SQL数据库中而不必设计自己的加密方法。

- 租赁和续约（Leasing and Renewal）

  Vault中的所有Secrets都具有与其关联的租约。在租赁结束时，Vault会自动撤销该Secret。客户可以通过内置的更新API更新租约。

- 撤销（Revocation）

  Vault内置支持Secrets的撤销。Vault不仅可以撤销单个secret，而且可以撤销Secrets树，例如由特定用户读取的所有Secrets或特定类型的所有Secrets。撤销功能可以协助Secrets的滚动切换以及在入侵的情况下锁定系统。

#### Vault架构

![](/assets/vault-arch-overview.png)

在详解上述架构之前，先来看一下Vault架构所涉及的几个概念：

- 存储后端（Storage Backend）

  存储后端负责加密数据的持久存储。Vault不信任存储后端，并且仅期望其提供持久性存储功能。在启动Vault服务器时需要配置存储后端。

- 屏障（Barrier）

  屏障如同银行保险库的外层防护钢壳和围绕穹顶的混凝土。在Vault和存储后端之间流动的所有数据都通过屏障。该屏障确保仅输出加密数据，并且该数据在进入时被验证和解密。任何内部数据可以访问之前，障碍必须“开封”。

- Secret后端（Secret Backend）

  Secret后端负责管理Secrets。简单的Secret后端像“`generic`”后端在查询时简单地返回相同的secret。某些后端支持使用策略在每次查询时动态生成secret。这允许使用唯一的secret，可以让Vault进行细粒度的撤销和策略更新。例如，一个MySQL后端可以配置一个“web”策略。当读取“web”密码时，将生成一个新的MySQL用户/密码对，其仅具有Web服务器的有限权限集。

- 审计后端（Audit Backend）

  审计后端负责管理审计日志。来自Vault的每个请求和来自Vault的响应都将通过配置的审核后端。这提供了一种将Vault与不同类型的多个审核日志记录目标集成的简单方法。

- 凭据后端（Credential Backend）

  凭据后端用于验证连接到Vault的用户或应用程序。一旦验证，后端返回应用的适用策略列表。Vault接受经过身份验证的用户，并返回可用于将来请求的客户端令牌。例如，用户密码后端使用用户名和密码来认证用户。

- 客户端令牌（Client Token）

  客户端令牌在概念上类似于网站上的会话cookie。用户验证后，Vault会返回一个用于将来请求的客户端令牌。Vault使用令牌验证客户端的身份并强制实施适用的ACL策略。此令牌通过HTTP头传递。

- Secret

  Secret是Vault返回的包含机密或加密信息的任何内容。不是由Vault返回的所有内容都是Secret，例如系统配置，状态信息或后端策略不被视为Secrets。Secret总是有相关的租约，这意味着客户端不能假设Secret可以无限期地使用。Vault将在租赁结束时撤销Secret，并且维护人员可以在租赁结束之前进行干预以撤销Secret。

- 服务器（Server）

  Vault依赖于作为服务器运行的长时间运行的实例。 Vault服务器提供了一个API，客户端与之交互并管理所有后端之间的交互，ACL实施和Secret租用撤销。采用基于服务器的架构可以客户端与安全密钥及策略分离，实现集中式审计日志记录并简化操作员的管理。

再看上述Vault架构图，从图中可以看出组件被屏障Barrier清晰的隔离为内外两部分。只有存储后端和HTTP API位于外部，其他组件都位于屏障内部。存储后端是不可信的，被Vault用来存储加密后的数据。

Vault启动时处于**密封（“sealed”）**状态，在可以与其交互之前，Vault必须切换到**启封（“unsealed”）**状态。状态切换需要提供启封钥匙。Vault初始化时会生成一把加密密钥来保护所有的数据，这个加密密钥受**主钥匙**（master key）保护。默认情况下，Vault使用[Shamir的密钥共享算法](https://en.wikipedia.org/wiki/Shamir's_Secret_Sharing)将主钥匙拆分为5份，必须提供5个密钥中任意3个来重建主钥匙。

![](/assets/vault-master-keys-shares-arch.png)

主钥匙拆分的数量及用来重建主钥匙的最低份数都可以设置，也可以禁用Shamir的密钥共享算法，这时需要使用主钥匙直接启封Vault。一旦Vault检索到加密密钥，它就能够解密存储后端中的数据，并进入启封状态。取消密封后，Vault会加载所有配置的审计，凭据和Secrets后端。

当客户端首次连接到Vault时，需要进行身份验证。 Vault提供可配置的凭据后端（Credential Backend），为客户端提供灵活的身份验证机制，如操作人员可使用用户名/密码，GitHub验证，而应用可以使用公私钥或Token令牌验证。一个验证请求从Vault内核进入凭据后端，由其校验合法性并返回一个关联的策略列表。

策略只是一个命名的ACL规则，例如，“`root`”策略是一个内置策略，具有所有资源的访问权限。可以创建任意数量的命名策略来细粒度的控制访问路径。Vault以白名单模式运作，除非透过策略明确授予存取权限，否则不允许执行操作。由于用户可能具有多个关联的策略，因此如果任何一个策略允许，则允许操作。策略由内部策略存储库存储和管理，此内部存储通过系统后端操作，系统后端始终挂载在`sys/`下。

校验过程中，凭据后端通过提供的策略进行验证，如果验证通过会生成一个新的客户端令牌保存到令牌存储，并返回给客户端供后续请求使用。如果凭据后端的配置为客户端令牌设定了租约，则该客户端令牌必须周期性的生成以避免变得无效。

校验通过后，所有的请求必须提供令牌，通过令牌可以校验客户端是否得到授权并加载相关的策略，加载的策略用于校验用户特定操作是否得到授权。操作请求随后路由到Secrets后端，由其根据自身类型进行相应的处理。如果Secrets后端返回了一个secret，Vault内核会将其注册到过期管理服务并为其附加一个租用ID。客户端可以使用这个租用ID来撤销或重新生成其secret，如果客户端允许租约自动过期，则由过期管理服务负责secret的到期撤销。

Vault内核负责记录请求和响应并发送给审计总线，再由其存储到审计后端。在请求处理流程之外，Vault内核负责某些后台操作，如关键的租约管理可以撤销客户端令牌和自动撤销Secrets。此外，Vault通过使用回滚管理器使用预写写日志（write ahead logging）来处理某些部分故障情况的恢复。

#### Secret Backends

- AWS
- Cassandra
- Consul
- Cubbyhole
- Generic
- MongoDB
- MSSQL
- MySQL
- PKI (Certificates)
- PostgreSQL
- RabbitMQ
- SSH
- Transit
- Custom

#### Auth Backends

- App ID
- AppRole
- AWS EC2
- GitHub
- LDAP
- MFA
- Okta
- RADIUS
- TLS Certificates
- Tokens
- Username & Password

#### Audit Backends

- File
- Syslog
- Socket

### Keywhiz

[Keywhiz](https://github.com/square/keywhiz)是Square开发的用于管理和分发秘密的系统。它可以很好地适应面向服务的体系结构（SOA）。

关于Keywhiz与Vault的对比可以参考：[Vault vs. Keywhiz](https://www.vaultproject.io/intro/vs/keywhiz.html)。