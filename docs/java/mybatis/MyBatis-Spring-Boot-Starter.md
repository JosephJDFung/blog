## MyBatis-Spring-Boot-Starter

MyBatis-Spring-Boot-Starter是在Spring Boot之上快速构建MyBatis应用程序。

- 通过使用此模块，将实现：
    - 构建独立的应用程序
    - 将样板减少到几乎为零
    - 较少的XML配置

* MyBatis-Spring-Boot-Starter需要以下版本：

MyBatis-Spring-Boot-Starter | MyBatis-Spring | Spring Boot | Java
-|-|-|-
2.1	2.0 (need 2.0.2+ for enable all features) | 2.1 |or higher | 8 or higher
1.3 | 1.3 | 1.5 | 6 or higher

### 安装
要使用MyBatis-Spring-Boot-Starter模块，您只需在类路径中包含mybatis-spring-boot-autoconfigure.jar文件及其依赖项（mybatis.jar，mybatis-spring.jar等）。

* 如果使用的是Maven，只需将以下依赖项添加到pom.xml中：
```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.1.1</version>
</dependency>
```

### 快速设置

要在Spring上使用MyBatis，至少需要一个SqlSessionFactory和至少一个mapper接口。

- MyBatis-Spring-Boot-Starter将：

    - 自动检测现有的`DataSource`
    - 通过`DataSource`作为输入使用的`SqlSessionFactoryBean`将创建并注册一个实例的`SqlSessionFactory`
    - 将创建并注册从`SqlSessionFactory`中获取的`SqlSessionTemplate`的实例
    - 自动扫描您的映射器，将它们链接到`SqlSessionTemplate`并将它们注册到Spring上下文，以便可以将它们注入到您的bean中

假设我们有以下映射器：
```java
@Mapper
public interface CityMapper {

  @Select("SELECT * FROM CITY WHERE state = #{state}")
  City findByState(@Param("state") String state);

}
```

只需要创建一个普通的Spring引导应用程序，然后按如下所示注入映射器即可（在Spring 4.3+上可用）：

```java
@SpringBootApplication
public class SampleMybatisApplication implements CommandLineRunner {

  private final CityMapper cityMapper;

  public SampleMybatisApplication(CityMapper cityMapper) {
    this.cityMapper = cityMapper;
  }

  public static void main(String[] args) {
    SpringApplication.run(SampleMybatisApplication.class, args);
  }

  @Override
  public void run(String... args) throws Exception {
    System.out.println(this.cityMapper.findByState("CA"));
  }

}
```

### 进阶scanning

* 默认情况下，MyBatis-Spring-Boot-Starter将搜索带有@Mapper批注的映射器。

* 需要指定自定义注解进行扫描。使用@MapperScan注解。

* 如果MyBatis-Spring-Boot-Starter 在Spring的上下文中找到至少一个MapperFactoryBean，则它将不会开始扫描过程，因此，如果想完全停止扫描，则应使用@Bean方法显式注册映射器。

### 使用SqlSession

创建了SqlSessionTemplate的实例并将其添加到Spring上下文中，因此可以使用MyBatis API使其像下面那样注入到bean中（在Spring 4.3+上可用）：

```java
@Component
public class CityDao {

  private final SqlSession sqlSession;

  public CityDao(SqlSession sqlSession) {
    this.sqlSession = sqlSession;
  }

  public City selectCityById(long id) {
    return this.sqlSession.selectOne("selectCityById", id);
  }

}
```

### Configuration

与其他任何Spring Boot应用程序一样，MyBatis-Spring-Boot-Application配置参数存储在application.properties（或application.yml）中。

MyBatis使用前缀mybatis作为其属性。

可用属性为：

属性 | 描述
-|-
config-location | MyBatis xml配置文件的位置。
check-config-location | 指示是否执行MyBatis xml配置文件的存在性检查。
mapper-locations | Mapper xml配置文件的位置。
type-aliases-package | 用于搜索类型别名的软件包。（包定界符为“ ，; \ t \ n ”）
type-aliases-super-type | 用于过滤类型别名的超类。如果未指定，则MyBatis将所有从type-aliases-package搜索的类都当作类型别名。
type-handlers-package | 用于搜索类型处理程序的软件包。（包定界符为“ ，; \ t \ n ”）
executor-type | 执行程序类型：SIMPLE，REUSE，BATCH
default-scripting-language-driver | 默认脚本语言驱动程序类。此功能需要与mybatis-spring 2.0.2+一起使用。
configuration-properties | MyBatis配置的外部化属性。可以将指定的属性用作MyBatis配置文件和Mapper文件上的占位符。
lazy-initialization | 是否启用映射器bean的延迟初始化。设置为true以启用延迟初始化。此功能需要与mybatis-spring 2.0.2+一起使用。
configuration.* | MyBatis Core提供的Configuration bean的属性键。关于可用的嵌套属性，此属性不能与config-location一起使用。
scripting-language-driver.thymeleaf.* | MyBatis Thymeleaf提供的ThymeleafLanguageDriverConfig bean的属性键。
scripting-language-driver.freemarker.* | MyBatis FreeMarker提供的FreeMarkerLanguageDriverConfig bean的属性键。此功能需要与mybatis-freemarker 1.2.0+一起使用。
scripting-language-driver.velocity.* | MyBatis Velocity提供的VelocityLanguageDriverConfig bean的属性键。此功能需要与mybatis-velocity 2.1.0+一起使用。


```yaml
# application.yml
mybatis:
    type-aliases-package: com.example.domain.model
    type-handlers-package: com.example.typehandler
    configuration:
        map-underscore-to-camel-case: true
        default-fetch-size: 100
        default-statement-timeout: 30
...
```

### 使用ConfigurationCustomizer

MyBatis-Spring-Boot-Starter提供了自定义使用Java Config自动配置生成的MyBatis配置的机会。MyBatis-Spring-Boot-Starter将自动搜索实现ConfigurationCustomizer接口的bean ，并调用自定义MyBatis配置的方法。（自1.2.1或更高版本开始可用）

```java
@Configuration
public class MyBatisConfig {
  @Bean
  ConfigurationCustomizer mybatisConfigurationCustomizer() {
    return new ConfigurationCustomizer() {
      @Override
      public void customize(Configuration configuration) {
        // customize ...
      }
    };
  }
}
```


### 检测MyBatis组件

- MyBatis-Spring-Boot-Starter将检测实现MyBatis提供的以下接口的bean。
    - Interceptor
    - TypeHandler
    - LanguageDriver (需要与mybatis-spring 2.0.2+一起使用)
    - DatabaseIdProvider 

```java
@Configuration
public class MyBatisConfig {
  @Bean
  MyInterceptor myInterceptor() {
    return MyInterceptor();
  }
  @Bean
  MyTypeHandler myTypeHandler() {
    return MyTypeHandler();
  }
  @Bean
  MyLanguageDriver myLanguageDriver() {
    return MyLanguageDriver();
  }
  @Bean
  VendorDatabaseIdProvider databaseIdProvider() {
    VendorDatabaseIdProvider databaseIdProvider = new VendorDatabaseIdProvider();
    Properties properties = new Properties();
    properties.put("SQL Server", "sqlserver");
    properties.put("DB2", "db2");
    properties.put("H2", "h2");
    databaseIdProvider.setProperties(properties);
    return databaseIdProvider;
  }  
}
```

`注意`:如果检测到LangaugeDriver的计数为1，则会自动设置为默认脚本语言。

#### 自定义LanguageDriver

如果要自定义通过自动配置创建的LanguageDriver，请注册用户定义的bean。

* ThymeleafLanguageDriver
* FreeMarkerLanguageDriverConfig
* ThymeleafLanguageDriver
* [案例](http://mybatis.org/spring-boot-starter/mybatis-spring-boot-autoconfigure/#)

### 项目依赖
[官网介绍连接](http://mybatis.org/spring-boot-starter/mybatis-spring-boot-autoconfigure/dependencies.html)