## cfg4j

[cfg4j](http://www.cfg4j.org/)是一个开源的为分布式系统设计的轻量的简单易用的配置管理工具，可以与Spring，Guice及其它DI无缝集成，支持JSON，YAML，Java Properties等配置格式，支持Git仓库，Consul，文件及系统环境变量等多种配置存储机制。

cfg4j主要包括以下特性：

- 配置自动重新加载：每个服务每隔特定时间可以重新加载配置并自动重新配置自身。

- 多环境支持：适用于用户定义的各种环境，如：测试，预发布，金丝雀部署，各种生产环境等。

- 本地缓存：在推送缓存或配置库关闭时防止服务中断，并提高获取配置的性能。

- 回退和合并策略：简化本地开发，同时支持多个配置存储库。

- 多DI框架支持：支持Spring，Guice等多种DI。


### 功能示例

#### 添加依赖

```xml
<dependencies>
  <dependency>
    <groupId>org.cfg4j</groupId>
    <artifactId>cfg4j-core</artifactId>
    <version>4.4.0</version>
  </dependency>
  
  <!-- Optional plug-ins -->
  
  <!-- Consul integration -->
  <dependency> 
    <groupId>org.cfg4j</groupId>
    <artifactId>cfg4j-consul</artifactId>
    <version>4.4.0</version>
  </dependency>
  
  <!-- Git integration -->
  <dependency>
    <groupId>org.cfg4j</groupId>
    <artifactId>cfg4j-git</artifactId>
    <version>4.4.0</version>
  </dependency>
</dependencies>
```

#### 初始化配置服务

```java

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

应用通过配置对象注入配置，而不是直接提供配置。这里也有两种方案：

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

#### 访问配置

直接获取原始类型的配置：

```java
Boolean property = provider.getProperty("some.property", Boolean.class); // some.property=false

Integer property = provider.getProperty("some.other.property", Integer.class); // some.other.property=1234

float[] floats = provider.getProperty("my.floatsArray", float[].class); // my.floatsArray=1.2,99.999,0.15

URL url = provider.getProperty("my.url", URL.class); // my.url=http://www.cfg4j.org
```

直接获取集合类型的配置：

```java
List<String> stringList = provider.getProperty("some.string.list", new GenericType<List<String>>() {}); // some.string.list=how,are,you

Map<String, Integer> pairs = provider.getProperty("some.map", new GenericType<Map<String, Integer>>() {}); // some.map=a=1,b=33,c=-10
```

直接获取配置对象：

```java
// Define your configuration interface
public interface MyConfiguration {
  Boolean featureToggle();
  File[] someFiles();
  Map<String, Integer> prices();
}

// someFeature.featureToggle=true
// someFeature.someFiles=/temp/fileA.txt,/temp/fileB.txt
// someFeature.prices=apple=1,candy=33,car=10
MyConfiguration config = simpleConfigurationProvider.bind("someFeature", MyConfiguration.class);

// Use it (when configuration changes your object will be auto-updated!)
if(config.featureToggle()) {
  ...
}
```

#### 配置刷新

定时刷新配置：

```java
// Reload every second
ConfigurationProvider provider = new ConfigurationProviderBuilder()
        .withConfigurationSource(...)
        .withReloadStrategy(new PeriodicalReloadStrategy(1, TimeUnit.SECONDS))
        .build();
```

自定义触发刷新配置：

```java
// Implement your strategy
public class MyReloadStrategy implements ReloadStrategy {

  public void register(Reloadable resource) {
      ...
      resource.reload();
      ...
  }

}

ConfigurationProvider provider = new ConfigurationProviderBuilder()
        .withReloadStrategy(new MyReloadStrategy())
        .build();
```

#### 环境切换

cfg4j使用Environment告诉配置服务配置文件的存储位置。通过Environment可以切换不同的环境获取各自对应的配置。

```java
ConfigurationProvider provider = new ConfigurationProviderBuilder()
        .withConfigurationSource(new ConsulConfigurationSourceBuilder().build())
        .withEnvironment(new ImmutableEnvironment("myApplication/prod"))  // each conf variable will be prefixed with this value when fetching from Consul
        .build();
```

#### 不同配置文件的支持

cfg4j使用ConfigurationSource读取配置文件的内容时，能够通过文件的扩展名(.properties 或.yaml)自动识别是Properties文件还是YAML文件。

#### 从多个配置源中合并配置

```java
ConfigurationSource source1 = ...
ConfigurationSource source2 = ...

ConfigurationSource mergedSource = new MergeConfigurationSource(source1, source2);

```

#### 故障情况下回退

在分布式环境中工作时，故障很难避免，因此，可以考虑使用多个来源，当一个失败时，另一个将被用作后备。

```java
ConfigurationSource source1 = ...
ConfigurationSource source2 = ...

ConfigurationSource mergedSource = new FallbackConfigurationSource(source1, source2);

```


更多详细示例请参考[cfg4j官方文档](http://www.cfg4j.org/releases/latest/)。

### 快速开发

在本地开发时，可以通过以下方式加快开发速度：

向项目中添加一个临时配置文件，并使用cfg4j的MergeConfigurationSource从配置存储和文件读取配置。通过将本地文件设置为主配置源，cfg4j可以使用覆盖机制，如果在文件中找到该属性，将使用它，否则，将回退到使用配置存储中的值。

```java
@Bean
ConfigurationProvider configurationProvider(ConfigurationSource configurationSource,
                                           @Value("${configuration.environment}") String configurationEnvironment) {

 ConfigurationSource localOverrideSource = new FilesConfigurationSource(() -> Collections.singletonList(Paths.get("application.properties")));
 MergeConfigurationSource mergeConfigurationSource = new MergeConfigurationSource(localOverrideSource, configurationSource);

 return new ConfigurationProviderBuilder()
     .withConfigurationSource(mergeConfigurationSource)
     .withEnvironment(new ImmutableEnvironment("app/" + configurationEnvironment + "/feature"))
     .withReloadStrategy(new PeriodicalReloadStrategy(30, TimeUnit.SECONDS))
     .build();
}
```

可以Fork一个配置库，对fork库的配置进行更改，并使用cfg4j的GitConfigurationSource直接访问它（不需要推送缓存）。

```java
@Bean
ConfigurationProvider configurationProvider(@Value("${configuration.environment}") String configurationEnvironment) {
 ConfigurationSource GitSource = new GitConfigurationSourceBuilder()
     .withRepositoryURI("https://Github.com/myUser/my-config-fork.Git")
     .build();

 return new ConfigurationProviderBuilder()
     .withConfigurationSource(GitSource)
     .withEnvironment(new ImmutableEnvironment("app/" + configurationEnvironment + "/feature"))
     .withReloadStrategy(new PeriodicalReloadStrategy(30, TimeUnit.SECONDS))
     .build();
}
```

设置一个私有的推送缓存，将开发的服务指向该缓存，并直接在缓存修改配置。

### 参考

- http://code.flickr.net/2016/03/24/configuration-management-for-distributed-systems-using-github-and-cfg4j/




