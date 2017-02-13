## cfg4j

[cfg4j](http://www.cfg4j.org/)是一个开源的为分布式系统设计的轻量的简单易用的配置管理工具，可以与Spring，Guice及其它DI无缝集成，支持JSON，YAML，Java Properties等配置格式，支持Git仓库，Consul，文件及系统环境变量等多种配置存储机制。

cfg4j主要包括以下特性：

- 配置自动重新加载：每个服务每隔特定时间可以重新加载配置并自动重新配置自身。

- 多环境支持：适用于用户定义的各种环境，如：测试，预发布，金丝雀部署，各种生产环境等。

- 本地缓存：在推送缓存或配置库关闭时防止服务中断，并提高获取配置的性能。

- 回退和合并策略：简化本地开发，同时支持多个配置存储库。

- 多DI框架支持：支持Spring，Guice等多种DI。


### 功能示例

下述代码使用依赖注入在Spring框架中启用cfg4j配置服务：

```java
import org.cfg4j.provider.ConfigurationProvider;
import org.cfg4j.provider.ConfigurationProviderBuilder;
import org.cfg4j.source.ConfigurationSource;
import org.cfg4j.source.consul.ConsulConfigurationSourceBuilder;
import org.cfg4j.source.context.environment.ImmutableEnvironment;
import org.cfg4j.source.reload.strategy.PeriodicalReloadStrategy;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.net.MalformedURLException;
import java.util.concurrent.TimeUnit;

@Configuration
class Cfg4jBeansConfig {

  @Value("${configuration.endpoint.protocol:http}")
  private String configurationEndpointProtocol;

  @Value("${configuration.endpoint.hostname:youForgotToSpecifyHost}")
  private String configurationEndpoint;

  @Value("${configuration.endpoint.port:8500}")
  private int configurationEndpointPort;

  @Bean
  ConfigurationSource globalConfigurationSource() throws MalformedURLException {
    return new ConsulConfigurationSourceBuilder()
        .withHost(configurationEndpoint)
        .withPort(configurationEndpointPort)
        .build();
  }

  @Bean
  ConfigurationProvider configurationProvider(ConfigurationSource configurationSource,
                                              @Value("${configuration.environment}") String configurationEnvironment) {
    return new ConfigurationProviderBuilder()
        .withConfigurationSource(configurationSource)
        .withEnvironment(new ImmutableEnvironment("app/" + configurationEnvironment + "/feature"))
        .withReloadStrategy(new PeriodicalReloadStrategy(30, TimeUnit.SECONDS))
        .build();
  }

  public interface SearchConfig {
    String defaultQuery();
  }

  @Bean
  SearchConfig searchConfig(ConfigurationProvider configurationProvider) {
    return configurationProvider.bind("search", SearchConfig.class);
  }
}
```

通过配置对象注入配置，而不是直接提供配置。这里也有两种方案：

- 直接注入配置（配置变更时不会自动更新）

```java
String defaultQuery = configurationProvider.getProperty("search.defaultQuery", String.class);

if(queryEmpty()) {
 return defaultQuery;
}
```

- 通过接口绑定注入配置（配置变更时会自动更新）

```java
SearchConfig config = configurationProvider.bind("search", SearchConfig.class);

// in search method
if(queryEmpty()) {
 return config.defaultQuery();
}
```


详细示例请参考[cfg4j官方文档](http://www.cfg4j.org/releases/latest/)。

### 参考

- http://code.flickr.net/2016/03/24/configuration-management-for-distributed-systems-using-github-and-cfg4j/




