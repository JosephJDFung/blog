## Spring Data MongoDB

### 入门

>转到文件→新建→Spring模板项目→Simple Spring Utility项目，然后在出现提示时按Yes。然后输入项目和程序包名称，例如org.spring.mongodb.example。将以下内容添加到pom.xml文件dependencies元素中：

```xml
<dependencies>

  <!-- other dependency elements omitted -->

  <dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-mongodb</artifactId>
    <version>3.0.1.RELEASE</version>
  </dependency>

</dependencies>
```

>设置日志记录级别DEBUG以查看一些其他信息。为此，请编辑log4j.properties文件以具有以下内容：

```
log4j.category.org.springframework.data.mongodb=DEBUG
log4j.appender.stdout.layout.ConversionPattern=%d{ABSOLUTE} %5p %40.40c:%4L - %m%n
```

创建一个Person要保留的类：

```java
package org.spring.mongodb.example;

public class Person {

  private String id;
  private String name;
  private int age;

  public Person(String name, int age) {
    this.name = name;
    this.age = age;
  }
}

```

主应用程序来运行：
```java
package org.spring.mongodb.example;

import static org.springframework.data.mongodb.core.query.Criteria.where;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.springframework.data.mongodb.core.MongoOperations;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.query.Query;

import com.mongodb.client.MongoClients;

public class MongoApp {

  private static final Log log = LogFactory.getLog(MongoApp.class);

  public static void main(String[] args) throws Exception {

    MongoOperations mongoOps = new MongoTemplate(MongoClients.create(), "database");
    mongoOps.insert(new Person("Joe", 34));

    log.info(mongoOps.findOne(new Query(where("name").is("Joe")), Person.class));

    mongoOps.dropCollection("person");
  }
}
```

### 使用Spring连接到MongoDB

使用MongoDB和Spring时，首要任务之一是com.mongodb.client.MongoClient使用IoC容器创建对象。使用基于Java的Bean元数据或使用基于XML的Bean元数据有两种主要方法。

#### 使用基于Java的元数据注册Mongo实例
```java
@Configuration
public class AppConfig {

  /*
   * Use the standard Mongo driver API to create a com.mongodb.client.MongoClient instance.
   */
   public @Bean MongoClient mongoClient() {
       return MongoClients.create("mongodb://localhost:27017");
   }
}

```
这种方法使您可以使用标准com.mongodb.client.MongoClient实例，而容器使用Spring的MongoClientFactoryBean。与com.mongodb.client.MongoClient直接实例化实例相比，它FactoryBean具有一个额外的优势，即还为容器提供了一个ExceptionTranslator实现，该实现将MongoDB异常转换为Spring的可移植DataAccessException层次结构中带@Repository注释的数据访问类的异常。Spring的DAO支持功能中@Repository描述了这种层次结构和用法。

#### 使用基于XML的元数据注册Mongo实例

虽然您可以使用Spring的传统`<beans/>XML`名称空间向com.mongodb.client.MongoClient容器注册一个实例，但是XML可能很冗长，因为它是通用的。XML名称空间是配置常用对象（如Mongo实例）的更好的选择。mongo命名空间使您可以创建Mongo实例服务器的位置，副本集和选项。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xmlns:mongo="http://www.springframework.org/schema/data/mongo"
          xsi:schemaLocation=
          "
          http://www.springframework.org/schema/data/mongo https://www.springframework.org/schema/data/mongo/spring-mongo.xsd
          http://www.springframework.org/schema/beans
          https://www.springframework.org/schema/beans/spring-beans.xsd">

    <!-- Default bean name is 'mongo' -->
    <mongo:mongo-client host="localhost" port="27017"/>

</beans>
```

### MongoDatabaseFactory接口

虽然这com.mongodb.client.MongoClient是MongoDB驱动程序API的入口点，但连接到特定的MongoDB数据库实例需要其他信息，例如数据库名称以及可选的用户名和密码。有了这些信息，您就可以获取com.mongodb.client.MongoDatabase对象并访问特定MongoDB数据库实例的所有功能。Spring提供了org.springframework.data.mongodb.core.MongoDatabaseFactory如下清单所示的接口，用于引导与数据库的连接：
```java
public interface MongoDatabaseFactory {

  MongoDatabase getDatabase() throws DataAccessException;

  MongoDatabase getDatabase(String dbName) throws DataAccessException;
}
```

### MongoTemplate

MongoTemplate是中央级的Spring的MongoDB的支持，并提供了与数据库交互的丰富的功能集。该模板提供了创建，更新，删除和查询MongoDB文档的便捷操作，并提供了域对象和MongoDB文档之间的映射。

>配置后，它MongoTemplate是线程安全的，并且可以在多个实例之间重用。

MongoTemplate类实现了接口MongoOperations。尽可能MongoOperations以MongoDB驱动程序Collection对象上可用的方法命名方法，以使熟悉该驱动程序API的现有MongoDB开发人员熟悉该API。例如，你可以找到方法，如find，findAndModify，findAndReplace，findOne，insert，remove，save，update，和updateMulti。设计目标是使在基本MongoDB驱动程序和NET的使用之间的转换尽可能容易MongoOperations。这两个API之间的主要区别是MongoOperations可以传递域对象而不是Document。此外，MongoOperations有流利的API Query，Criteria以及Update操作，而不是填充一个Document 为这些操作指定参数。

>引用MongoTemplate实例操作的首选方法是通过其接口MongoOperations。

- 所使用的默认转换器实现MongoTemplate是MappingMongoConverter。尽管MappingMongoConverter可以使用其他元数据来指定对象到文档的映射，但是它也可以通过使用一些ID和集合名称映射的约定来转换不包含其他元数据的对象。

- MongoTemplate提供了许多便利的方法来帮助您轻松执行常见任务。但是，如果您需要直接访问MongoDB驱动程序API，则可以使用几种Execute回调方法之一。execute回调为您提供对com.mongodb.client.MongoCollection或com.mongodb.client.MongoDatabase对象的引用。

#### 实例化MongoTemplate

注册一个com.mongodb.client.MongoClient对象并启用Spring的异常转换支持
```java
@Configuration
public class AppConfig {

  public @Bean MongoClient mongoClient() {
      return MongoClients.create("mongodb://localhost:27017");
  }

  public @Bean MongoTemplate mongoTemplate() {
      return new MongoTemplate(mongoClient(), "mydatabase");
  }
}
```

- 有几个重载的构造函数MongoTemplate：

    - MongoTemplate(MongoClient mongo, String databaseName)：使用MongoClient对象和默认数据库名称进行操作。

    - MongoTemplate(MongoDatabaseFactory mongoDbFactory)：采用一个MongoDbFactory对象，该MongoClient对象封装了该对象，数据库名称以及用户名和密码。

    - MongoTemplate(MongoDatabaseFactory mongoDbFactory, MongoConverter mongoConverter)：添加一个MongoConverter用于映射。

### 保存，更新和删除文档
```java
public class Person {

  private String id;
  private String name;
  private int age;

  public Person(String name, int age) {
    this.name = name;
    this.age = age;
  }
}
```

```java
package org.spring.example;

import static org.springframework.data.mongodb.core.query.Criteria.where;
import static org.springframework.data.mongodb.core.query.Update.update;
import static org.springframework.data.mongodb.core.query.Query.query;

import java.util.List;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.springframework.data.mongodb.core.MongoOperations;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.SimpleMongoClientDbFactory;

import com.mongodb.client.MongoClients;

public class MongoApp {

  private static final Log log = LogFactory.getLog(MongoApp.class);

  public static void main(String[] args) {

    MongoOperations mongoOps = new MongoTemplate(new SimpleMongoClientDbFactory(MongoClients.create(), "database"));

    Person p = new Person("Joe", 34);

    // Insert is used to initially store the object into the database.
    mongoOps.insert(p);
    log.info("Insert: " + p);

    // Find
    p = mongoOps.findById(p.getId(), Person.class);
    log.info("Found: " + p);

    // Update
    mongoOps.updateFirst(query(where("name").is("Joe")), update("age", 35), Person.class);
    p = mongoOps.findOne(query(where("name").is("Joe")), Person.class);
    log.info("Updated: " + p);

    // Delete
    mongoOps.remove(p);

    // Check that deletion worked
    List<Person> people =  mongoOps.findAll(Person.class);
    log.info("Number of people = : " + people.size());


    mongoOps.dropCollection(Person.class);
  }
}
```
>MongoOperations是MongoTemplate实现的接口。

前面的示例将产生以下日志输出（包括来自的调试消息MongoTemplate）：

```
DEBUG apping.MongoPersistentEntityIndexCreator:  80 - Analyzing class class org.spring.example.Person for index information.
DEBUG work.data.mongodb.core.MongoTemplate: 632 - insert Document containing fields: [_class, age, name] in collection: person
INFO               org.spring.example.MongoApp:  30 - Insert: Person [id=4ddc6e784ce5b1eba3ceaf5c, name=Joe, age=34]
DEBUG work.data.mongodb.core.MongoTemplate:1246 - findOne using query: { "_id" : { "$oid" : "4ddc6e784ce5b1eba3ceaf5c"}} in db.collection: database.person
INFO               org.spring.example.MongoApp:  34 - Found: Person [id=4ddc6e784ce5b1eba3ceaf5c, name=Joe, age=34]
DEBUG work.data.mongodb.core.MongoTemplate: 778 - calling update using query: { "name" : "Joe"} and update: { "$set" : { "age" : 35}} in collection: person
DEBUG work.data.mongodb.core.MongoTemplate:1246 - findOne using query: { "name" : "Joe"} in db.collection: database.person
INFO               org.spring.example.MongoApp:  39 - Updated: Person [id=4ddc6e784ce5b1eba3ceaf5c, name=Joe, age=35]
DEBUG work.data.mongodb.core.MongoTemplate: 823 - remove using query: { "id" : "4ddc6e784ce5b1eba3ceaf5c"} in collection: person
INFO               org.spring.example.MongoApp:  46 - Number of people = : 0
DEBUG work.data.mongodb.core.MongoTemplate: 376 - Dropped collection [database.person]
```

### _id在映射层中处理字段

MongoDB要求您有一个_id用于所有文档的字段。如果不提供，驱动程序将为分配ObjectId一个生成的值。当您使用时MappingMongoConverter，某些规则控制Java类中的属性如何映射到此_id字段：

- 用@Id（org.springframework.data.annotation.Id）注释的属性或字段映射到该_id字段。

- 没有注释但已命名的属性或字段id映射到该_id字段。

以下概述了_id使用时MappingMongoConverter（默认为MongoTemplate）对映射到文档字段的属性执行的类型转换（如果有）。

- 如果可能，通过使用Spring id将String在Java类中声明为的属性或字段转换为，并存储为。有效的转换规则委托给MongoDB Java驱动程序。如果不能将其转换为，则该值将作为字符串存储在数据库中。`ObjectIdConverter<String, ObjectId>ObjectId`

- 使用Spring 将在Java类中id声明为的属性或字段BigInteger转换为，并存储为。`ObjectIdConverter<BigInteger, ObjectId>`

```java
public class PlainStringId {
  @MongoId String id; //该id被视为String未经进一步转换。
}

public class PlainObjectId {
  @MongoId ObjectId id; //该ID被视为ObjectId。
}

public class StringToObjectId {
    //将id视为ObjectId给定String的有效ObjectId十六进制，否则视为String。对@Id应用法。
  @MongoId(FieldType.OBJECT_ID) String id; 
}
```

### 地理空间查询
MongoDB的支持通过使用等运营商的地理空间查询$near，$within，geoWithin，和$nearSphere。Criteria该类提供了特定于地理空间查询的方法。还有一些形状类（Box，Circle和Point）与地理空间相关Criteria方法结合使用。

- MongoDB 4.2删除geoNear了以前用于运行的命令的支持 NearQuery。

- Spring Data MongoDB 2.2 MongoOperations#geoNear使用$geoNear 聚合 而不是geoNear命令来运行NearQuery。

- dis现在，以前在包装器类型内返回的计算距离（使用geoNear命令时）将被嵌入到生成的文档中。如果给定的域类型已经包含具有该名称的属性，则calculated-distance使用潜在的随机后缀来命名计算的距离。

- 目标类型可能包含一个以返回的距离命名的属性，以（另外）将其直接读回域类型，如下所示。

```java
GeoResults<VenueWithDisField> = template.query(Venue.class) //用于标识目标集合和潜在查询映射的域类型。
    .as(VenueWithDisField.class)   //	包含类型为的dis字段的目标类型Number。                         
    .near(NearQuery.near(new GeoJsonPoint(-73.99, 40.73), KILOMETERS))
    .all();
```

MongoDB支持在数据库中查询地理位置，并同时计算到给定起点的距离。使用地理附近的查询，您可以表达查询，例如“查找周围10英里内的所有餐馆”。为此，请MongoOperations提供geoNear(…)以a NearQuery作为参数的方法（以及已经熟悉的实体类型和集合），如以下示例所示：

```java
Point location = new Point(-73.99171, 40.738868);
NearQuery query = NearQuery.near(location).maxDistance(new Distance(10, Metrics.MILES));

GeoResults<Restaurant> = operations.geoNear(query, Restaurant.class);
```