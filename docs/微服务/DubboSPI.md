# Dubbo SPI

## JavaSPI

>SPI（Service Provider Interface），是JDK内置的一种 服务提供发现机制，可以用来启用框架扩展和替换组件

>JavaSPI是“基于接口的编程＋策略模式＋配置文件”组合实现的动态加载机制
- 步骤
    - 定义一组接口
    - 写出接口的一个或多个实现
    - 在 src/main/resources/ 下建立 /META-INF/services 目录， 新增一个以接口命名的文件, 内容是要应用的实现类
    - 使用 ServiceLoader 来加载配置文件中指定的实现。 ,调用方通过ServiceLoader.load方法加载接口的实现类实例

ServiceLoader 的实现： 
1. 读取配置文件，获取实现类的全名称字符串； 
2. 使用 Java 反射机制来构造服务实现类的实例。可以使用泛型方法，避免获取的时候做类型转换。

JDK 自带的 java.util.ServiceLoader 使用了 ClassLoader 来加载类，并使用迭代器来获取服务实现类。
```java
public final class ServiceLoader<S> implements Iterable<S> {


    //扫描目录前缀
    private static final String PREFIX = "META-INF/services/";

    // 被加载的类或接口
    private final Class<S> service;

    // 用于定位、加载和实例化实现方实现的类的类加载器
    private final ClassLoader loader;

    // 上下文对象
    private final AccessControlContext acc;

    // 按照实例化的顺序缓存已经实例化的类
    private LinkedHashMap<String, S> providers = new LinkedHashMap<>();

    // 懒查找迭代器
    private java.util.ServiceLoader.LazyIterator lookupIterator;

    // 私有内部类，提供对所有的service的类的加载与实例化
    private class LazyIterator implements Iterator<S> {
        Class<S> service;
        ClassLoader loader;
        Enumeration<URL> configs = null;
        String nextName = null;

        private boolean hasNextService() {
            if (configs == null) {
                try {
                    //获取目录下所有的类
                    String fullName = PREFIX + service.getName();
                    if (loader == null)
                        configs = ClassLoader.getSystemResources(fullName);
                    else
                        configs = loader.getResources(fullName);
                } catch (IOException x) {
                   
                }
            }
        }

        private S nextService() {
            String cn = nextName;
            nextName = null;
            Class<?> c = null;
            try {
                //反射加载类
                c = Class.forName(cn, false, loader);
            } catch (ClassNotFoundException x) {
            }
            try {
                //实例化
                S p = service.cast(c.newInstance());
                //放进缓存
                providers.put(cn, p);
                return p;
            } catch (Throwable x) {
                
            }
            
        }
    }
}
```

SPI 的应用之一是可替换的插件机制。比如JDBC 数据库驱动包 mysql-connector-java-5.1.18.jar 就有一个 /META-INF/services/java.sql.Driver 里面内容是 com.mysql.jdbc.Driver 。

>Java SPI优缺点

- 优点：使用Java SPI机制的优势是实现了解耦，使第三方模块的装配逻辑与业务代码分离。应用程序可以根据实际业务情况使用新的框架拓展或者替换原有组件。

- 缺点：ServiceLoader在加载实现类的时候会全部加载并实例化，假如不想使用某些实现类，它也会被加载示例化的，这就造成了浪费。另外获取某个实现类只能通过迭代器迭代获取，不能根据某个参数来获取，使用方式上不够灵活;多个并发多线程使用 ServiceLoader 类的实例是`不安全`的。