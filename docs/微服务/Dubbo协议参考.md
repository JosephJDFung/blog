# Dubbo协议参考

## dubbo://

Dubbo 缺省协议采用单一长连接和 NIO 异步通讯，适合于小数据量大并发的服务调用，以及服务消费者机器数远大于服务提供者机器数的情况。
反之，Dubbo 缺省协议不适合传送大数据量的服务，比如传文件，传视频等，除非请求量很低。

### 特性

缺省协议，使用基于 mina 1.1.7 和 hessian 3.2.1 的 tbremoting 交互。

- 连接个数：单连接
- 连接方式：长连接
- 传输协议：TCP
- 传输方式：NIO 异步传输
- 序列化：Hessian 二进制序列化


适用范围：传入传出参数数据包较小（建议小于100K），消费者比提供者个数多，单一消费者无法压满提供者，尽量不要用 dubbo 协议传输大文件或超大字符串。

- 适用场景：常规远程服务方法调用

### 约束
- 参数及返回值需实现 Serializable 接口
- 参数及返回值不能自定义实现 List, Map, Number, Date, Calendar 等接口，只能用 JDK 自带的实现，因为 hessian 会做特殊处理，自定义实现类中的属性值都会丢失。
Hessian 序列化，只传成员属性值和值的类型，不传方法或静态变量，兼容情况
- 服务器端和客户端对领域对象并不需要完全一致，而是按照最大匹配原则。

### 配置

配置协议：
```xml
<dubbo:protocol name="dubbo" port="20880" />
```
设置默认协议：
```xml
<dubbo:provider protocol="dubbo" />
```
设置服务协议：
```xml
<dubbo:service protocol="dubbo" />
```
多端口：
```xml
<dubbo:protocol id="dubbo1" name="dubbo" port="20880" />
<dubbo:protocol id="dubbo2" name="dubbo" port="20881" />
```
配置协议选项：
```xml
<dubbo:protocol name=“dubbo” port=“9090” server=“netty” client=“netty” codec=“dubbo” serialization=“hessian2” charset=“UTF-8” threadpool=“fixed” threads=“100” queues=“0” iothreads=“9” buffer=“8192” accepts=“1000” payload=“8388608” />
```
多连接配置：

Dubbo 协议缺省每服务每提供者每消费者使用单一长连接，如果数据量较大，可以使用多个连接。
```xml
<dubbo:service connections="1"/>
<dubbo:reference connections="1"/>
<dubbo:service connections="0"> 或 <dubbo:reference connections="0"> 表示该服务使用 JVM 共享长连接。缺省
<dubbo:service connections="1"> 或 <dubbo:reference connections="1"> 表示该服务使用独立长连接。
<dubbo:service connections="2"> 或<dubbo:reference connections="2"> 表示该服务使用独立两条长连接。
```
为防止被大量连接撑挂，可在服务提供方限制大接收连接数，以实现服务提供方自我保护。
```xml
<dubbo:protocol name="dubbo" accepts="1000" />
```
dubbo.properties 配置：
```
dubbo.service.protocol=dubbo
```

- 为什么采用异步单一长连接?

因为服务的现状大都是服务提供者少，通常只有几台机器，而服务的消费者多，可能整个网站都在访问该服务，比如 Morgan 的提供者只有 6 台提供者，却有上百台消费者，每天有 1.5 亿次调用，如果采用常规的 hessian 服务，服务提供者很容易就被压跨，通过单一连接，保证单一消费者不会压死提供者，长连接，减少连接握手验证等，并使用异步 IO，复用线程池，防止 C10K 问题。

## rmi://
RMI 协议采用 JDK 标准的 java.rmi.* 实现，采用阻塞式短连接和 JDK 标准序列化方式。

注意：如果正在使用 RMI 提供服务给外部访问 [1]，同时应用里依赖了老的 common-collections 包 [2] 的情况下，存在反序列化安全风险 [3]。

### 特性
- 连接个数：多连接
- 连接方式：短连接
- 传输协议：TCP
- 传输方式：同步传输
- 序列化：Java 标准二进制序列化
- 适用范围：传入传出参数数据包大小混合，消费者与提供者个数差不多，可传文件。
- 适用场景：常规远程服务方法调用，与原生RMI服务互操作

### 约束
参数及返回值需实现 Serializable 接口
dubbo 配置中的超时时间对 RMI 无效，需使用 java 启动参数-设置：`-Dsun.rmi.transport.tcp.responseTimeout=3000`，参见下面的 RMI 配置
- dubbo.properties 配置
```
dubbo.service.protocol=rmi
```

- RMI配置
java -Dsun.rmi.transport.tcp.responseTimeout=3000


### 接口

如果服务接口继承了 java.rmi.Remote 接口，可以和原生 RMI 互操作，即：

提供者用 Dubbo 的 RMI 协议暴露服务，消费者直接用标准 RMI 接口调用，
或者提供方用标准 RMI 暴露服务，消费方用 Dubbo 的 RMI 协议调用。

- 如果服务接口没有继承 java.rmi.Remote 接口：

缺省 Dubbo 将自动生成一个 com.xxx.XxxService$Remote 的接口，并继承 java.rmi.Remote 接口，并以此接口暴露服务，
但如果设置了` <dubbo:protocol name="rmi" codec="spring" />`，将不生成 $Remote 接口，而使用 Spring 的 RmiInvocationHandler 接口暴露服务，和 Spring 兼容。

### 配置
定义 RMI 协议：
```xml
<dubbo:protocol name="rmi" port="1099" />
```
设置默认协议：
```xml
<dubbo:provider protocol="rmi" />
```
设置服务协议：
```xml
<dubbo:service protocol="rmi" />
```
多端口：
```xml
<dubbo:protocol id="rmi1" name="rmi" port="1099" />
<dubbo:protocol id="rmi2" name="rmi" port="2099" />
 
<dubbo:service protocol="rmi1" />
```
Spring 兼容性：
```xml
<dubbo:protocol name="rmi" codec="spring" />
```


## hessian://

- Hessian 是 Caucho 开源的一个 RPC 框架，其通讯效率高于 WebService 和 Java 自带的序列化。
- Hessian  协议用于集成 Hessian 的服务，Hessian 底层采用 Http 通讯，采用 Servlet 暴露服务，Dubbo 缺省内嵌 Jetty 作为服务器实现。

- Dubbo 的 Hessian 协议可以和原生 Hessian 服务互操作，即：

提供者用 Dubbo 的 Hessian 协议暴露服务，消费者直接用标准 Hessian 接口调用
或者提供方用标准 Hessian 暴露服务，消费方用 Dubbo 的 Hessian 协议调用。

### 特性
- 连接个数：多连接
- 连接方式：短连接
- 传输协议：HTTP
- 传输方式：同步传输
- 序列化：Hessian二进制序列化
- 适用范围：传入传出参数数据包较大，提供者比消费者个数多，提供者压力较大，可传文件。
- 适用场景：页面传输，文件传输，或与原生hessian服务互操作

### 依赖
```xml
<dependency>
    <groupId>com.caucho</groupId>
    <artifactId>hessian</artifactId>
    <version>4.0.7</version>
</dependency>
```

### 约束
参数及返回值需实现 Serializable 接口
参数及返回值不能自定义实现 List, Map, Number, Date, Calendar 等接口，只能用 JDK 自带的实现，因为 hessian 会做特殊处理，自定义实现类中的属性值都会丢失。
### 配置
定义 hessian 协议：
```xml
<dubbo:protocol name="hessian" port="8080" 
server="jetty" />
```
设置默认协议：
```xml
<dubbo:provider protocol="hessian" />
```
设置 service 协议：
```xml
<dubbo:service protocol="hessian" />
```
多端口：
```xml
<dubbo:protocol id="hessian1" name="hessian" port="8080" />
<dubbo:protocol id="hessian2" name="hessian" port="8081" />
```
直连：
```xml
<dubbo:reference id="helloService" interface="HelloWorld" url="hessian://10.20.153.10:8080/helloWorld" />
```

## http://

基于 HTTP 表单的远程调用协议，采用 Spring 的 HttpInvoker 实现 (2.3.0 以上版本支持)

### 特性
- 连接个数：多连接
- 连接方式：短连接
- 传输协议：HTTP
- 传输方式：同步传输
- 序列化：表单序列化
- 适用范围：传入传出参数数据包大小混合，提供者比消费者个数多，可用浏览器查看，可用表单或URL传入参数，暂不支持传文件。
- 适用场景：需同时给应用程序和浏览器 JS 使用的服务。

### 配置
配置协议：
```xml
<dubbo:protocol name="http" port="8080" />
```
配置 Jetty Server (默认)：
```xml
<dubbo:protocol ... server="jetty" />
```
配置 Servlet Bridge Server (推荐使用)：
```xml
<dubbo:protocol ... server="servlet" />
```
配置 DispatcherServlet：
```xml
<servlet>
         <servlet-name>dubbo</servlet-name>
         <servlet-class>org.apache.dubbo.remoting.http.servlet.DispatcherServlet</servlet-class>
         <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
         <servlet-name>dubbo</servlet-name>
         <url-pattern>/*</url-pattern>
</servlet-mapping>
```
`注意`，如果使用 servlet 派发请求：

协议的端口 `<dubbo:protocol port="8080" /> `必须与 servlet 容器的端口相同，
协议的上下文路径` <dubbo:protocol contextpath="foo" /> `必须与 servlet 应用的上下文路径相同。

## webservice://
基于 WebService 的远程调用协议，基于 Apache CXF (RPC 框架，由 Xfire 和 Celtix 合并) 的 frontend-simple 和 transports-http 实现(2.3.0 以上版本支持)。

可以和原生 WebService 服务互操作，即：

提供者用 Dubbo 的 WebService 协议暴露服务，消费者直接用标准 WebService 接口调用，
或者提供方用标准 WebService 暴露服务，消费方用 Dubbo 的 WebService 协议调用。

### 依赖

```
<dependency>
    <groupId>org.apache.cxf</groupId>
    <artifactId>cxf-rt-frontend-simple</artifactId>
    <version>2.6.1</version>
</dependency>
<dependency>
    <groupId>org.apache.cxf</groupId>
    <artifactId>cxf-rt-transports-http</artifactId>
    <version>2.6.1</version>
</dependency>
```
### 特性

- 连接个数：多连接
- 连接方式：短连接
- 传输协议：HTTP
- 传输方式：同步传输
- 序列化：SOAP 文本序列化
- 适用场景：系统集成，跨语言调用
### 约束
- 参数及返回值需实现 Serializable 接口
- 参数尽量使用基本类型和 POJO
### 配置
配置协议：
```
<dubbo:protocol name="webservice" port="8080" server="jetty" />
```
配置默认协议：
```
<dubbo:provider protocol="webservice" />
```
配置服务协议：
```
<dubbo:service protocol="webservice" />
```
多端口：
```
<dubbo:protocol id="webservice1" name="webservice" port="8080" />
<dubbo:protocol id="webservice2" name="webservice" port="8081" />
```
直连：
```
<dubbo:reference id="helloService" interface="HelloWorld" url="webservice://10.20.153.10:8080/com.foo.HelloWorld" />
```
WSDL：
```
http://10.20.153.10:8080/com.foo.HelloWorld?wsdl
```
Jetty Server (默认)：
```
<dubbo:protocol ... server="jetty" />
```
Servlet Bridge Server (推荐)：
```
<dubbo:protocol ... server="servlet" />
```
配置 DispatcherServlet：
```
<servlet>
         <servlet-name>dubbo</servlet-name>
         <servlet-class>org.apache.dubbo.remoting.http.servlet.DispatcherServlet</servlet-class>
         <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
         <servlet-name>dubbo</servlet-name>
         <url-pattern>/*</url-pattern>
</servlet-mapping>
```
- 注意，如果使用 servlet 派发请求：
    - 协议的端口 `<dubbo:protocol port="8080" />` 必须与 servlet 容器的端口相同，
    - 协议的上下文路径 `<dubbo:protocol contextpath="foo" />`必须与 servlet 应用的上下文路径相同。

## thrift://
- 当前 dubbo 支持的 thrift 协议是对 thrift 原生协议 的扩展，在原生协议的基础上添加了一些额外的头信息，比如 service name，magic number 等。

- 使用 dubbo thrift 协议同样需要使用 thrift 的 idl compiler 编译生成相应的 java 代码，后续版本中会在这方面做一些增强。

### 依赖
```xml
<dependency>
    <groupId>org.apache.thrift</groupId>
    <artifactId>libthrift</artifactId>
    <version>0.8.0</version>
</dependency>
```
### 配置
所有服务共用一个端口 ：

`<dubbo:protocol name="thrift" port="3030" />`

### 常见问题
Thrift 不支持 null 值，即：不能在协议中传递 null 值

## grpc://

Dubbo 自 2.7.5 版本开始支持 gRPC 协议，对于计划使用 HTTP/2 通信，或者想利用 gRPC 带来的 Stream、反压、Reactive 编程等能力的开发者来说， 都可以考虑启用 gRPC 协议。

### 支持 gRPC 的好处

为期望使用 gRPC 协议的用户带来服务治理能力，方便接入 Dubbo 体系

用户可以使用 Dubbo 风格的，基于接口的编程风格来定义和使用远程服务
### 如何在 Dubbo 中使用 gRPC
- 使用 IDL 定义服务
- 配置 compiler 插件，本地预编译
- 配置暴露/引用 Dubbo 服务
