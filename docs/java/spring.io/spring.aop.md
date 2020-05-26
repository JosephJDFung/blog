## AOP介绍

### 什么是AOP

- 在软件业，AOP为Aspect Oriented Programming的缩写，意为：面向切面编程，通过`预编译方式`和`运行期动态代理`实现程序功能的统一维护的一种技术。`AOP是OOP（面向对象编程）的延续`，是软件开发中的一个热点，也是Spring框架中的一个重要内容，是函数式编程的一种衍生范型。`利用AOP可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合度降低，提高程序的可重用性，同时提高了开发的效率。`
- AOP采取横向抽取机制，取代了传统纵向继承体系重复性代码
- 经典应用：事务管理、性能监视、安全检查、缓存 、日志等
- Spring AOP使用纯Java实现，不需要专门的编译过程和类加载器，在运行期通过代理方式向目标类织入增强代码
- AspectJ是一个基于Java语言的AOP框架，Spring2.0开始，Spring AOP引入对Aspect的支持，AspectJ扩展了Java语言，提供了一个专门的编译器，在编译时提供横向代码的织入

### AOP实现原理

- aop底层将采用代理机制进行实现。
- 接口 + 实现类 ：spring采用 jdk 的动态代理Proxy。
- 实现类：spring 采用 cglib字节码增强。

### AOP术语

- 1.target：目标类，需要被代理的类。例如：UserService
- 2.Joinpoint(连接点):所谓连接点是指那些可能被拦截到的方法。例如：所有的方法
- 3.PointCut 切入点：已经被增强的连接点。例如：addUser()
- 4.advice 通知/增强，增强代码。例如：after、before
- 5.Weaving(织入):是指把增强advice应用到目标对象target来创建新的代理对象proxy的过程.
- 6.proxy 代理类
- 7.Aspect(切面): 是切入点pointcut和通知advice的结合
	- 一个线是一个特殊的面。
	- 一个切入点和一个通知，组成成一个特殊的面。

### AspectJ

- AspectJ是一个基于Java语言的AOP框架
- Spring2.0以后新增了对AspectJ切点表达式支持
- @AspectJ 是AspectJ1.5新增功能，通过JDK5注解技术，允许直接在Bean类中定义切面
新版本Spring框架，建议使用AspectJ方式来开发AOP
- 主要用途：自定义开发

#### 切入点表达式
* execution()  用于描述方法
```
语法：execution(修饰符  返回值  包.类.方法名(参数) throws异常)
		修饰符，一般省略
			public		公共方法
			*			任意
		返回值，不能省略
			void			返回没有值
			String		返回值字符串
			* 			任意
		包，[省略]
			com.itheima.crm			固定包
			com.itheima.crm.*.service	crm包下面子包任意 （例如：com.itheima.crm.staff.service）
			com.itheima.crm..			crm包下面的所有子包（含自己）
			com.itheima.crm.*.service..	crm包下面任意子包，固定目录service，service目录任意包
		类，[省略]
			UserServiceImpl			指定类
			*Impl					以Impl结尾
			User*					以User开头
			*						任意
		方法名，不能省略
			addUser					固定方法
			add*						以add开头
			*Do						以Do结尾
			*						任意
		(参数)
			()						无参
			(int)						一个整型
			(int ,int)					两个
			(..)						参数任意
		throws ,可省略，一般不写。
```

#### AspectJ 通知类型

- aop联盟定义通知类型，具有特性接口，必须实现，从而确定方法名称。
- aspectj 通知类型，只定义类型名称。已经方法格式。
- 个数：6种
	- before:前置通知(应用：各种校验)
		在方法执行前执行，如果通知抛出异常，阻止方法运行
	- afterReturning:后置通知(应用：常规数据处理)
		方法正常返回后执行，如果方法中抛出异常，通知无法执行
		必须在方法执行后才执行，所以可以获得方法的返回值。
	- around:环绕通知(应用：十分强大，可以做任何事情)
		方法执行前后分别执行，可以阻止方法的执行
		必须手动执行目标方法
	- afterThrowing:抛出异常通知(应用：包装异常信息)
		方法抛出异常后执行，如果方法没有抛出异常，无法执行
	- after:最终通知(应用：清理现场)
		方法执行完毕后执行，无论方法中是否出现异常