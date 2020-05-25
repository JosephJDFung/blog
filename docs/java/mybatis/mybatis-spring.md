## mybatis-spring

MyBatis-Spring 会帮助你将 MyBatis 代码无缝地整合到 Spring 中。它将允许 MyBatis 参与到 Spring 的事务管理之中，创建映射器 mapper 和 SqlSession 并注入到 bean 中，以及将 Mybatis 的异常转换为 Spring 的 DataAccessException。最终，可以做到应用代码不依赖于 MyBatis，Spring 或 MyBatis-Spring。

版本：


MyBatis-Spring | MyBatis | Spring 框架 | Spring Batch | Java |
-|-|-|-|-
2.0 | 3.5+ | 5.0+ | 4.0+ | Java 8+ |
1.3 | 3.4+ | 3.2.2+ | 2.1+ | Java 6+ |

### 快速上手
* 要和 Spring 一起使用 MyBatis，需要在 Spring 应用上下文中定义至少两样东西：一个 SqlSessionFactory 和至少一个数据映射器类。

* 在 MyBatis-Spring 中，可使用 SqlSessionFactoryBean来创建 SqlSessionFactory。 要配置这个工厂 bean，只需要把下面代码放在 Spring 的 XML 配置文件中：

```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
  <property name="dataSource" ref="dataSource" />
</bean>
```
* 需要注意的是 SqlSessionFactoryBean 实现了 Spring 的 FactoryBean 接口（参见 Spring 官方文档 3.8 节 通过工厂 bean 自定义实例化逻辑）。这意味着由 Spring 最终创建的 bean 并不是 SqlSessionFactoryBean 本身，而是工厂类（SqlSessionFactoryBean）的 getObject() 方法的返回结果。这种情况下，Spring 将会在应用启动时为你创建 SqlSessionFactory，并使用 sqlSessionFactory 这个名字存储起来。

等效的 Java 代码如下：
```java
@Bean
public SqlSessionFactory sqlSessionFactory() throws Exception {
  SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
  factoryBean.setDataSource(dataSource());
  return factoryBean.getObject();
}
```
`注意`：SqlSessionFactory 需要一个 DataSource（数据源）。 这可以是任意的 DataSource，只需要和配置其它 Spring 数据库连接一样配置它就可以了。

* 通常，在 MyBatis-Spring 中，你`不需要直接使用 SqlSessionFactoryBean 或对应的 SqlSessionFactory`。相反，session 的工厂 bean 将会被注入到 MapperFactoryBean 或其它继承于 SqlSessionDaoSupport 的 DAO（Data Access Object，数据访问对象）中。

### 属性

* SqlSessionFactory 有一个唯一的必要属性：用于 JDBC 的 `DataSource `。这可以是任意的 DataSource 对象，它的配置方法和其它 Spring 数据库连接是一样的。

* 一个常用的属性是 `configLocation`，它用来指定 MyBatis 的 XML 配置文件路径。它在需要修改 MyBatis 的基础配置非常有用。通常，基础配置指的是` <settings> `或 `<typeAliases> `元素。

* 需要注意的是，这个配置文件并不需要是一个完整的 MyBatis 配置。确切地说，任何环境配置（`<environments>`），数据源（`<DataSource>`）和 MyBatis 的事务管理器（`<transactionManager>`）都会被忽略。SqlSessionFactoryBean 会创建它自有的 MyBatis 环境配置（Environment），并按要求设置自定义环境的值。

- 如果 MyBatis 在映射器类对应的路径下找不到与之相对应的映射器 XML 文件，那么也需要配置文件。这时有两种解决办法：
    - 第一种是手动在 MyBatis 的 XML 配置文件中的 `<mappers>` 部分中指定 XML 文件的类路径；
    - 第二种是设置工厂 bean 的 mapperLocations 属性。

* mapperLocations 属性接受多个资源位置。这个属性可以用来指定 MyBatis 的映射器 XML 配置文件的位置。属性的值是一个 Ant 风格的字符串，可以指定加载一个目录中的所有文件，或者从一个目录开始递归搜索所有目录。

* 如果你使用了多个数据库，那么需要设置 databaseIdProvider 属性：

```xml
<bean id="databaseIdProvider" class="org.apache.ibatis.mapping.VendorDatabaseIdProvider">
  <property name="properties">
    <props>
      <prop key="SQL Server">sqlserver</prop>
      <prop key="DB2">db2</prop>
      <prop key="Oracle">oracle</prop>
      <prop key="MySQL">mysql</prop>
    </props>
  </property>
</bean>

<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
  <property name="dataSource" ref="dataSource" />
  <property name="mapperLocations" value="classpath*:sample/config/mappers/**/*.xml" />
  <property name="databaseIdProvider" ref="databaseIdProvider"/>
</bean>
```

* 自 1.3.0 版本开始，新增的 configuration 属性能够在没有对应的 MyBatis XML 配置文件的情况下，直接设置 Configuration 实例。例如：

```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
  <property name="dataSource" ref="dataSource" />
  <property name="configuration">
    <bean class="org.apache.ibatis.session.Configuration">
      <property name="mapUnderscoreToCamelCase" value="true"/>
    </bean>
  </property>
</bean>
```

#### 事务
* 一个使用 MyBatis-Spring 的其中一个主要原因是它允许 MyBatis 参与到 Spring 的事务管理中。而不是给 MyBatis 创建一个新的专用事务管理器，MyBatis-Spring 借助了 Spring 中的 DataSourceTransactionManager 来实现事务管理。

* 一旦配置好了 Spring 的事务管理器，你就可以在 Spring 中按你平时的方式来配置事务。并且支持 @Transactional 注解和 AOP 风格的配置。在事务处理期间，一个单独的 SqlSession 对象将会被创建和使用。当事务完成时，这个 session 会以合适的方式提交或回滚。

* 事务配置好了以后，MyBatis-Spring 将会透明地管理事务。这样在你的 DAO 类中就不需要额外的代码了。

### 使用 SqlSession

在 MyBatis 中，你可以使用 SqlSessionFactory 来创建 SqlSession。一旦你获得一个 session 之后，你可以使用它来执行映射了的语句，提交或回滚连接，最后，当不再需要它的时候，你可以关闭 session。使用 MyBatis-Spring 之后，你不再需要直接使用 SqlSessionFactory 了，因为你的 bean 可以被注入一个线程安全的 SqlSession，它能基于 Spring 的事务配置来自动提交、回滚、关闭 session。

#### SqlSessionTemplate

* SqlSessionTemplate 是 MyBatis-Spring 的核心。作为 SqlSession 的一个实现，这意味着可以使用它无缝代替你代码中已经在使用的 SqlSession。SqlSessionTemplate 是线程安全的，可以被多个 DAO 或映射器所共享使用。

* 当调用 SQL 方法时（包括由 getMapper() 方法返回的映射器中的方法），SqlSessionTemplate 将会保证使用的 SqlSession 与当前 Spring 的事务相关。此外，它管理 session 的生命周期，包含必要的关闭、提交或回滚操作。另外，它也负责将 MyBatis 的异常翻译成 Spring 中的 DataAccessExceptions。

* 由于模板可以参与到 Spring 的事务管理中，并且由于其是线程安全的，可以供多个映射器类使用，你应该总是用 SqlSessionTemplate 来替换 MyBatis 默认的 DefaultSqlSession 实现。在同一应用程序中的不同类之间混杂使用可能会引起数据一致性的问题。

* 可以使用 SqlSessionFactory 作为构造方法的参数来创建 SqlSessionTemplate 对象。

```xml
<bean id="sqlSession" class="org.mybatis.spring.SqlSessionTemplate">
  <constructor-arg index="0" ref="sqlSessionFactory" />
</bean>
```

```java
@Bean
public SqlSessionTemplate sqlSession() throws Exception {
  return new SqlSessionTemplate(sqlSessionFactory());
}

// 现在，这个 bean 就可以直接注入到你的 DAO bean 中了。你需要在你的 bean 中添加一个 SqlSession 属性

public class UserDaoImpl implements UserDao {

  private SqlSession sqlSession;

  public void setSqlSession(SqlSession sqlSession) {
    this.sqlSession = sqlSession;
  }

  public User getUser(String userId) {
    return sqlSession.selectOne("org.mybatis.spring.sample.mapper.UserMapper.getUser", userId);
  }
}
```

按下面这样，注入 SqlSessionTemplate：

```xml
<bean id="userDao" class="org.mybatis.spring.sample.dao.UserDaoImpl">
  <property name="sqlSession" ref="sqlSession" />
</bean>

<!-- SqlSessionTemplate 还有一个接收 ExecutorType 参数的构造方法。这允许你使用如下 Spring 配置来批量创建对象，例如批量创建一些 SqlSession -->
<bean id="sqlSession" class="org.mybatis.spring.SqlSessionTemplate">
  <constructor-arg index="0" ref="sqlSessionFactory" />
  <constructor-arg index="1" value="BATCH" />
</bean>

```

```java
@Bean
public SqlSessionTemplate sqlSession() throws Exception {
  return new SqlSessionTemplate(sqlSessionFactory(), ExecutorType.BATCH);
}

// 现在所有的映射语句可以进行批量操作了，可以在 DAO 中编写如下的代码

public void insertUsers(List<User> users) {
  for (User user : users) {
    sqlSession.insert("org.mybatis.spring.sample.mapper.UserMapper.insertUser", user);
  }
}
```

* 注意，只需要在希望语句执行的方法与 SqlSessionTemplate 中的默认设置不同时使用这种配置。

* 这种配置的弊端在于，当调用这个方法时，不能存在使用不同 ExecutorType 的进行中的事务。要么确保对不同 ExecutorType 的 SqlSessionTemplate 的调用处在不同的事务中，要么完全不使用事务。

#### SqlSessionDaoSupport

SqlSessionDaoSupport 是一个抽象的支持类，用来为你提供 SqlSession。调用 getSqlSession() 方法你会得到一个 SqlSessionTemplate，之后可以用于执行 SQL 方法，就像下面这样:

```java
public class UserDaoImpl extends SqlSessionDaoSupport implements UserDao {
  public User getUser(String userId) {
    return getSqlSession().selectOne("org.mybatis.spring.sample.mapper.UserMapper.getUser", userId);
  }
}
```

* 在这个类里面，通常更倾向于使用 MapperFactoryBean，因为它不需要额外的代码。但是，如果你需要在 DAO 中做其它非 MyBatis 的工作或需要一个非抽象的实现类，那么这个类就很有用了。

* SqlSessionDaoSupport 需要通过属性设置一个 sqlSessionFactory 或 SqlSessionTemplate。如果两个属性都被设置了，那么 SqlSessionFactory 将被忽略。

* 假设类 UserMapperImpl 是 SqlSessionDaoSupport 的子类，可以编写如下的 Spring 配置来执行设置：

```xml
<bean id="userDao" class="org.mybatis.spring.sample.dao.UserDaoImpl">
  <property name="sqlSessionFactory" ref="sqlSessionFactory" />
</bean>
```
### 注入映射器

与其在数据访问对象（DAO）中手工编写使用 SqlSessionDaoSupport 或 SqlSessionTemplate 的代码，还不如让 Mybatis-Spring 为你创建一个线程安全的映射器，这样你就可以直接注入到其它的 bean 中了：

```xml
<bean id="fooService" class="org.mybatis.spring.sample.service.FooServiceImpl">
  <constructor-arg ref="userMapper" />
</bean>
```

注入完毕后，映射器就可以在你的应用逻辑代码中使用了：

```java
public class FooServiceImpl implements FooService {

  private final UserMapper userMapper;

  public FooServiceImpl(UserMapper userMapper) {
    this.userMapper = userMapper;
  }

  public User doSomeBusinessStuff(String userId) {
    return this.userMapper.getUser(userId);
  }
}
```
* 注意代码中并没有任何的对 SqlSession 或 MyBatis 的引用。你也不需要担心创建、打开、关闭 session，MyBatis-Spring 将为你打理好一切。

#### 注册映射器

注册映射器的方法根据你的配置方法，即经典的 XML 配置或新的 3.0 以上版本的 Java 配置（也就是常说的 @Configuration），而有所不同。

* XML 配置

```xml
<bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
  <property name="mapperInterface" value="org.mybatis.spring.sample.mapper.UserMapper" />
  <property name="sqlSessionFactory" ref="sqlSessionFactory" />
</bean>
```

* 如果映射器接口 UserMapper 在相同的类路径下有对应的 MyBatis XML 映射器配置文件，将会被 MapperFactoryBean 自动解析。不需要在 MyBatis 配置文件中显式配置映射器，除非映射器配置文件与接口类不在同一个类路径下。参考 SqlSessionFactoryBean 的 configLocation 属性以获取更多信息。

* 注意 MapperFactoryBean 需要配置一个 SqlSessionFactory 或 SqlSessionTemplate。它们可以分别通过 sqlSessionFactory 和 sqlSessionTemplate 属性来进行设置。如果两者都被设置，SqlSessionFactory 将被忽略。由于 SqlSessionTemplate 已经设置了一个 session 工厂，MapperFactoryBean 将使用那个工厂。


* java 配置
```java
@Bean
public MapperFactoryBean<UserMapper> userMapper() throws Exception {
  MapperFactoryBean<UserMapper> factoryBean = new MapperFactoryBean<>(UserMapper.class);
  factoryBean.setSqlSessionFactory(sqlSessionFactory());
  return factoryBean;
}
```

#### 发现映射器

不需要一个个地注册你的所有映射器。你可以让 MyBatis-Spring 对类路径进行扫描来发现它们。

- 有几种办法来发现映射器：

  - 使用 `<mybatis:scan/>` 元素
  - 使用 @MapperScan 注解
  - 在经典 Spring XML 配置文件中注册一个 MapperScannerConfigurer

<mybatis:scan/> 和 @MapperScan 都在 MyBatis-Spring 1.2.0 中被引入。@MapperScan 需要使用 Spring 3.1+。


#### mybatis-spring示例代码
[示例代码](http://mybatis.org/spring/zh/sample.html)
