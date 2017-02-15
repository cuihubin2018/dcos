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

在详解上述架构之前，先来看一下Vault所涉及的几个概念：

存储后端（Storage Backend）

存储后端负责加密数据的持久存储。Vault不信任存储后端，并且仅期望其提供持久性存储功能。在启动Vault服务器时需要配置存储后端。

屏障（Barrier）

屏障如同银行保险库的外层防护钢壳和围绕穹顶的混凝土。在Vault和存储后端之间流动的所有数据都通过屏障。该屏障确保仅输出加密数据，并且该数据在进入时被验证和解密。任何内部数据可以访问之前，障碍必须“开封”。

Secret后端（Secret Backend）

Secret后端负责管理Secrets。简单的Secret后端像“`generic`”后端在查询时简单地返回相同的secret。某些后端支持使用策略在每次查询时动态生成secret。这允许使用唯一的secret，可以让Vault进行细粒度的撤销和策略更新。例如，一个MySQL后端可以配置一个“web”策略。当读取“web”密码时，将生成一个新的MySQL用户/密码对，其仅具有Web服务器的有限权限集。

审计后端（Audit Backend）

审计后端负责管理审计日志。来自Vault的每个请求和来自Vault的响应都将通过配置的审核后端。这提供了一种将Vault与不同类型的多个审核日志记录目标集成的简单方法。

凭据后端（Credential Backend）

凭据后端用于验证连接到Vault的用户或应用程序。一旦验证，后端返回应用的适用策略列表。Vault接受经过身份验证的用户，并返回可用于将来请求的客户端令牌。例如，用户密码后端使用用户名和密码来认证用户。

客户端令牌（Client Token）

客户端令牌在概念上类似于网站上的会话cookie。用户验证后，Vault会返回一个用于将来请求的客户端令牌。Vault使用令牌验证客户端的身份并强制实施适用的ACL策略。此令牌通过HTTP头传递。

Secret

Secret是Vault返回的包含机密或加密信息的任何内容。不是由Vault返回的所有内容都是Secret，例如系统配置，状态信息或后端策略不被视为Secrets。Secret总是有相关的租约，这意味着客户端不能假设Secret可以无限期地使用。Vault将在租赁结束时撤销Secret，并且维护人员可以在租赁结束之前进行干预以撤销Secret。

服务器（Server）



Keywhiz