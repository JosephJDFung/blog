# Tomcat

tomcat 主要有 连接器 Connector 和 容器  两大部分，连接器主要负责建立和解析tcp 连接，具体的业务逻辑有容器处理。

容器|作用
-|-
Server容器|一个StandardServer类实例就表示一个Server容器，server是tomcat的顶级构成容器
Service容器|一个StandardService类实例就表示一个Service容器，Tomcat的次顶级容器，Service是这样一个集合：它由一个或者多个Connector组成，以及一个Engine，负责处理所有Connector所获得的客户请求。
Engine容器|一个StandardEngine类实例就表示一个Engine容器。Engine下可以配置多个虚拟主机Virtual Host，每个虚拟主机都有一个域名。当Engine获得一个请求时，它把该请求匹配到某个Host上，然后把该请求交给该Host来处理，Engine有一个默认虚拟主机，当请求无法匹配到任何一个Host上的时候，将交给该默认Host来处理
Host容器|一个StandardHost类实例就表示一个Host容器，代表一个VirtualHost，虚拟主机，每个虚拟主机和某个网络域名Domain Name相匹配。每个虚拟主机下都可以部署(deploy)一个或者多个WebApp，每个Web App对应于一个Context，有一个Context path。当Host获得一个请求时，将把该请求匹配到某个Context上，然后把该请求交给该Context来处理。匹配的方法是“最长匹配”，所以一个path==”“的Context将成为该Host的默认Context。所有无法和其它Context的路径名匹配的请求都将最终和该默认Context匹配
Context容器|一个StandardContext类实例就表示一个Context容器。一个Context对应于一个Web Application，一个WebApplication由一个或者多个Servlet组成。Context在创建的时候将根据配置文件CATALINA_HOME/conf/web.xml和WEBAPP_HOME/WEB-INF/web.xml载入Servlet类。当Context获得请求时，将在自己的映射表(mappingtable)中寻找相匹配的Servlet类。如果找到，则执行该类，获得请求的回应，并返回
Wrapper容器|一个StandardWrapper类实例就表示一个Wrapper容器，Wrapper容器负责管理一个Servlet，包括Servlet的装载、初始化、资源回收。Wrapper是最底层的容器，其不能在添加子容器了。Wrapper是一个接口，其标准实现类是StandardWrapper

## tomcat四大容器

Tomcat提供了engine，host，context及wrapper四种容器。在总体结构中已经阐述了他们之间的包含关系。这四种容器继承了一个容器基类，因此可以定制化。当然，tomcat也提供了标准实现。

![](https://dl.iteye.com/upload/picture/pic/124653/861d6c1e-174c-3afb-bd7b-48bd82cc7f62.jpg)

### Enigne

Engine是最顶层的容器，它是host容器的组合。

engine有四大组件：

- Cluster: 实现tomcat集群，例如session共享等功能，通过配置server.xml可以实现，对其包含的所有host里的应用有效，该模块是可选的。其实现方式是基于pipeline+valve模式的，有时间会专门整理一个pipeline+valve模式应用系列；
- Realm：实现用户权限管理模块，例如用户登录，访问控制等，通过通过配置server.xml可以实现，对其包含的所有host里的应用有效，该模块是可选的；
- Pipeline：这里简单介绍下，之后会有专门文档说明。每个容器对象都有一个pipeline，它不是通过server.xml配置产生的，是必须有的。它就是容器对象实现逻辑操作的骨架，在pipeline上配置不同的valve，当需要调用此容器实现逻辑时，就会按照顺序将此pipeline上的所有valve调用一遍，这里可以参考责任链模式；
- Valve：实现具体业务逻辑单元。可以定制化valve（实现特定接口），然后配置在server.xml里。对其包含的所有host里的应用有效。定制化的valve是可选的，但是每个容器有一个缺省的valve，例如engine的StandardEngineValve，是在StandardEngine里自带的，它主要实现了对其子host对象的StandardHostValve的调用，以此类推。

```xml

<Engine name=“Catalina” defaultHost=“localhost”>     
  <Valve className=“MyValve0”/>     
  <Valve className=“MyValve1”/>     
  <Valve className=“MyValve2”/>     
  ……     
  <Host name=“localhost” appBase=“webapps”>   
  </Host>  
</Engine> 

```

### Host

Host是engine的子容器，它是context容器的集合。

StandardHost的核心模块与StandardEngine差不多。只是作用域不一样，它的模块只对其包含的子context有效。除此，还有一些特殊的逻辑，例如context的部署。

### Context

Context是host的子容器，它是wrapper容器的集合。

主要结构：


![](https://dl.iteye.com/upload/picture/pic/124655/23566eaf-9a09-354e-9c44-f1a4a9211053.jpg)

>Manager

它主要是应用的session管理模块。其主要功能是session的创建，session的维护，session的持久化(persistence)，以及跨context的session的管理等。

- manager模块是必须要有的，可以在server.xml中配置，如果没有配置的话，会在程序里生成一个manager对象。

- Resources: 它是每个web app对应的部署结构的封装，比如，有的app是tomcat的webapps目录下的某个子目录或是在context节点配置的其他目录，或者是war文件部署的结构等。它对于每个web app是必须的。
- Loader：它是对每个web app的自有的classloader的封装。具体内容涉及到tomcat的classloader体系，会在一篇文档中单独说明。Tomcat正是有一套完整的classloader体系，才能保证每个web app或是独立运营，或是共享某些对象等等。它对于每个web app是必须的。
- Mapper：它封装了请求资源URI与每个相对应的处理wrapper容器的映射关系。

web.xml配置:

```xml
<servlet>  
  <servlet-name>httpserver</servlet-name>  
  <servlet-class>com.gearever.servlet.TestServlet</servlet-class>  
</servlet>  
   
<servlet-mapping>  
  <servlet-name>httpserver</servlet-name>  
  <url-pattern>/*.do</url-pattern>  
</servlet-mapping>  
```

mapper对象，可以抽象的理解成一个map结构，其key是某个访问资源，例如`/*.do`，那么其value就是封装了处理这个资源TestServlet的某个wrapper对象。当访问`/*.do`资源时，TestServlet就会在mapper对象中定位到。这里需要特别说明的是，通过这个mapper对象定位特定的wrapper对象的方式，只有一种情况，那就是在servlet或jsp中通过forward方式访问资源时用到。

### Wrapper

Wrapper是context的子容器，它封装的处理资源的每个具体的servlet。

>servlet对象与servlet stack对象

这两个对象在wrapper容器中只存在其中之一，也就是说只有其中一个不为空。当以servlet对象存在时，说明此servlet是支持多线程并发访问的，也就是说不存在线程同步的过程，此wrapper容器中只包含一个servlet对象（这是我们常用的模式）；

当以servlet stack对象存在时，说明servlet是不支持多线程并发访问的，每个servlet对象任一时刻只有一个线程可以调用，这样servlet stack实现的就是个简易的线程池，此wrapper容器中只包含一组servlet对象，它的基本原型是worker thread模式实现的。 


那么，怎么来决定是以servlet对象方式存储还是servlet stack方式存储呢？其实，只要在开发servlet类时，实现一个SingleThreadModel接口即可。

需要线程同步的servlet类

```java
public class LoginServlet extends HttpServlet implements javax.servlet.SingleThreadModel{
    
}  
```

>这种同步机制只是从servlet规范的角度来说提供的一种功能，在实际应用中并不能完全解决线程安全问题

## Web应用的初始化

WEB应用的初始化工作是在ContextConfig的configureStart方法中实现的，应用的初始化工作主要是解析web.xml文件，这个文件是一个WEB应用的入口。

1. Tomcat首先会找globalWebXml，这个文件的搜索路径是engine的工作目录下的org/apache/catalina/startup/NO-DEFAULT_XML或conf/web.xml。

2. 接着会找hostWebXml，这个文件可能会在System.getProperty(“catalina.base”)/conf/$ {EngineName}/${HostName}/web.xml.default中。

3. 接着寻找应用的配置文件examples/WEB-INF/web.xml，web.xml文件中的各个配置项将会被解析成相应的属性保存在WebXml对象中。

4. 接下来会讲WebXml对象中的属性设置到context容器中，这里包括创建servlet对象，filter，listerner等，这些在WebXml的configureContext方法中。


## SpringBoot 内置 Tomcat 参数配置

```yml
server:
  tomcat:
    accesslog:
      enabled: false #打开tomcat访问日志
      directory: logs #访问日志所在的目录
    accept-count: #允许HTTP请求缓存到请求队列的最大个数，默认不限制
    max-connections: #最大连接数，默认10000
    max-http-post-size: #HTTP POST内容最大长度，默认不限制
    max-threads: #最大工作线程数,默认200
  compression: #压缩：能大大减少网络传输量，页面加载速度加快，但压缩会有一定的CPU性能损耗。
    enabled: true #开启压缩
    min-response-size: 2048 #单位是byte，转换下2048byte=2kb,所以数据大于等于2kb才压缩，官方默认也是2kb。
    mime-types: #进行压缩的文件类型
      - text/html
      - application/json
```

这些参数最终在ServerProperties.Tomcat类中实现，包含以下属性：

```
maxThreads 最大工作线程数
minSpareThreads 最小工作线程数
maxHttpPostSize HTTP POST内容最大长度
internalProxies 受信任IP校验正则表达式
protocolHeader 协议头，通常设置为X-Forwarded-Proto
protocolHeaderHttpsValue 协议头的内容，判断是否使用了SSL，默认值是https
portHeader 用于覆盖原始端口值的HTTP头的名称，默认为X-Forwarded-Port
redirectContextRoot 对上下文根的请求是否应该通过附加/到路径来重定向
useRelativeRedirects 设置通过调用sendRedirect生成的HTTP 1.1和后面的位置头是使用相对重定向还是使用绝对重定向
remoteIpHeader 提取远程IP的HTTP头的名称。例如X-FORWARDED-FOR
maxConnections 最大连接数，如果一旦连接数到达，剩下的连接将会保存到请求缓存队列里，也就是accept-count指定队列
maxHttpHeaderSize HTTP消息头的最大值(以字节为单位)
acceptCount 当所有可能的请求处理线程都在使用时，传入连接请求的最大队列长度
```

配置Tomcat访问日志:

```
enabled 是否启用访问日志
pattern 访问日志的格式化模式，默认为common
directory 创建日志文件的目录。可以是绝对的或相对于Tomcat的基目录，默认是logs
prefix 日志文件名称前缀，默认为access_log
suffix 日志文件名称后缀，默认为.log
rotate 是否启用访问日志旋转，默认为true
renameOnRotate 是否推迟将日期戳包含在文件名中直到旋转时间。
fileDateFormat 日志文件名称中的日期格式，默认为.yyyy-MM-dd。
requestAttributesEnabled 为请求使用的IP地址、主机名、协议和端口设置请求属性。
buffered 是否缓冲输出，使其只定期刷新，默认为true
```

>远程Debug

服务端远程Debug模式开启,端口号为8888,在服务器上将启动参数修改为：
```shell
 java -Djavax.net.debug=ssl -Xdebug -Xnoagent -Djava.compiler=NONE -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=8888 -jar springboot-1.0.jar
```

其他调优参考：

[Springboot Tomcat APR模式详解和实践](https://www.jianshu.com/p/f716726ba340)