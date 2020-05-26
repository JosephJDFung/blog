## spring框架概述
* Spring是一个开源框架，Spring是于2003 年兴起的一个轻量级的Java 开发框架，它是为了解决企业应用开发的复杂性而创建的。框架的主要优势之一就是其分层架构，分层架构允许使用者选择使用哪一个组件，同时为 J2EE 应用程序开发提供集成的框架。Spring使用基本的JavaBean来完成以前只可能由EJB完成的事情。然而，Spring的用途不仅限于服务器端的开发。从简单性、可测试性和松耦合的角度而言，任何Java应用都可以从Spring中受益。Spring的核心是控制反转（IoC）和面向切面（AOP）。简单来说，Spring是一个分层的JavaSE/EE full-stack(一站式) 轻量级开源框架。
* 轻量级：与EJB对比，依赖资源少，销毁的资源少。
* 分层： 一站式，每一个层都提供的解决方案
    * web层：struts，spring-MVC
	* service层：spring
	* dao层：hibernate，mybatis ， jdbcTemplate  ->  spring-data

### spring核心
Spring的核心是控制反转（IoC）和面向切面（AOP）
* 核心容器：beans、core、context、expression
### spring优点

- 方便解耦，简化开发  （高内聚低耦合）
    - Spring就是一个大工厂（容器），可以将所有对象创建和依赖关系维护，交给Spring管理
    - spring工厂是用于生成bean
- AOP编程的支持
    - Spring提供面向切面编程，可以方便的实现对程序进行权限拦截、运行监控等功能
- 声明式事务的支持
    - 只需要通过配置就可以完成对事务的管理，而无需手动编程
- 方便程序的测试
    - Spring对Junit4支持，可以通过注解方便的测试Spring程序
- 方便集成各种优秀框架
    - Spring不排斥各种优秀的开源框架，其内部提供了对各种优秀框架（如：Struts、Hibernate、MyBatis、Quartz等）的直接支持
- 降低JavaEE API的使用难度
    - Spring 对JavaEE开发中非常难用的一些API（JDBC、JavaMail、远程调用等），都提供了封装，使这些API应用难度大大降低


### ioc & di
```
IoC：<bean id="" class="" >
DI：<bean> <property name="" value="" | ref="">
实例化方式：
  默认构造
  静态工厂：<bean id="" class="工厂类" factory-method="静态方法">
  实例工厂：<bean id="工厂id" class="工厂类">  <bean id="" factory-bean="工厂id" factory-method="方法">
作用域：<bean id="" class="" scope="singleton | prototype">
生命周期：<bean id="" class="" init-method="" destroy-method="">
  后处理bean  BeanPostProcessor接口，<bean class="注册"> ，对容器中所有的bean都生效
属性注入
  构造方法注入：<bean><constructor-arg index="" type="" >
  setter方法注入：<bean><property>
  p命名空间：简化<property>   <bean p:属性名="普通值"  p:属性名-ref="引用值">  注意声明命名空间
  SpEL：<property name="" value="#{表达式}">
     #{123}  #{'abc'}
     #{beanId.propName?.methodName()}
     #{T(类).静态方法|字段}
  集合
     数组<array>
     List <list>
     Set <set>
     Map <map><entry key="" value="">
     Properties <props><prop key="">....
```
### 核心API

- BeanFactory ：这是一个工厂，用于生成任意bean。
	采取延迟加载，第一次getBean时才会初始化Bean
- ApplicationContext：是BeanFactory的子接口，功能更强大。（国际化处理、事件传递、Bean自动装配、各种不同应用层的Context实现）。当配置文件被加载，就进行对象实例化。
	- ClassPathXmlApplicationContext 用于加载classpath（类路径、src）下的xml加载xml运行时位置 --> /WEB-INF/classes/...xml
	- FileSystemXmlApplicationContext 用于加载指定盘符下的xml
		加载xml运行时位置 --> /WEB-INF/...xml
			通过java web ServletContext.getRealPath() 获得具体盘符
### Bean种类
- 普通bean：之前操作的都是普通bean。`<bean id="" class="A">` ，spring直接创建A实例，并返回
- FactoryBean：是一个特殊的bean，具有工厂生成对象能力，只能生成特定的对象。
	- bean必须使用 FactoryBean接口，此接口提供方法 getObject() 用于获得特定bean。
	`<bean id="" class="FB">` 先创建FB实例，使用调用getObject()方法，并返回方法的返回值
		FB fb = new FB();
		return fb.getObject();
- BeanFactory 和 FactoryBean 对比
	- BeanFactory：工厂，用于生成任意bean。
	- FactoryBean：特殊bean，用于生成另一个特定的bean。例如：ProxyFactoryBean ，此工厂bean用于生产代理。`<bean id="" class="....ProxyFactoryBean">` 获得代理对象实例。


### 属性依赖注入
- 依赖注入方式：手动装配 和 自动装配
- 手动装配：一般进行配置信息都采用手动
	- 基于xml装配：构造方法、setter方法
	- 基于注解装配
- 自动装配：struts和spring 整合可以自动装配
	- byType：按类型装配 
	- byName：按名称装配
	- constructor构造装配
	- auto： 不确定装配

### 基于注解装配Bean 

- 1.@Component取代`<bean class="">`
	- @Component("id") 取代 `<bean id="" class="">`
- 2.web开发，提供3个@Component注解衍生注解（功能一样）取代`<bean class="">`
	- @Repository ：dao层
	- @Service：service层
	- @Controller：web层
- 3.依赖注入	，给私有字段设置，也可以给setter方法设置
	- 普通值：@Value("")
	- 引用值：
		- 方式1：按照【类型】注入
			- @Autowired
		- 方式2：按照【名称】注入1
			- @Autowired
			- @Qualifier("名称")
		- 方式3：按照【名称】注入2
			- @Resource("名称")
- 4.生命周期
	- 初始化：@PostConstruct
	- 销毁：@PreDestroy
- 5.作用域
	- @Scope("prototype") 多例

### 注解和xml混合使用

- 1.将所有的bean都配置xml中
	- `<bean id="" class="">`
- 2.将所有的依赖都使用注解
	- @Autowired
	默认不生效。为了生效，需要在xml配置：`<context:annotation-config>`

- 注解1：`<context:component-scan base-package=" ">`
- 注解2：`<context:annotation-config>`
- 一般情况两个注解不一起使用。
    - “注解1”扫描含有注解（@Component 等）类，注入注解自动生效。
	- “注解2”只在xml和注解（注入）混合使用时，使注入注解生效。

