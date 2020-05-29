## java8新特性

JDK1.8已经发布很久了，在很多企业中都已经在使用。并且Spring5、SpringBoot2.0都推荐使用JDK1.8以上版本。

Jdk8这个版本包含语言、编译器、库、工具和JVM等方面的十多个新特性：

- Lambda表达式
- 函数式接口
- 方法引用
- 接口的默认方法
- Optional
- Streams
- 并行数组

### Lambda 表达式
Lambda 表达式，也可称为闭包，它是推动 Java 8 发布的最重要新特性。Lambda 允许把函数作为一个方法的参数（函数作为参数传递进方法中）。可以使代码变的更加简洁紧凑。

```
(参数列表) -> {代码块}
```

需要注意：

- 参数类型可省略，编译器可以自己推断
- 如果只有一个参数，圆括号可以省略
- 代码块如果只是一行代码，大括号也可以省略
- 如果代码块是一行，且是有结果的表达式，`return`可以省略

**注意：**事实上，把Lambda表达式可以看做是匿名内部类的一种简写方式。当然，前提是这个匿名内部类对应的必须是接口，而且接口中必须只有一个函数！Lambda表达式就是直接编写函数的：参数列表、代码体、返回值等信息，**`用函数来代替完整的匿名内部类`**！

假设我们要对集合排序，我们先看JDK7的写法，需要通过匿名内部类来构造一个`Comparator`：

```java
// Jdk1.7写法
Collections.sort(list,new Comparator<Integer>() {
    @Override
    public int compare(Integer o1, Integer o2) {
        return o1 - o2;
    }
});
System.out.println(list);
```

如果是jdk8，我们可以使用新增的集合API：`sort(Comparator c)`方法，接收一个比较器，我们用Lambda来代替`Comparator` 的匿名内部类：

```java
// Jdk1.8写法，参数列表的数据类型可省略：
list.sort((i1,i2) -> { return i1 - i2;});

System.out.println(list);
```

```java
// Jdk8写法
// 因为代码块是一个有返回值的表达式，可以省略大括号以及return
list.sort((i1,i2) -> i1 - i2);
```

#### 把Lambda赋值给变量

Lambda表达式的实质其实还是匿名内部类，所以我们其实可以把Lambda表达式赋值给某个变量。

```java
// 将一个Lambda表达式赋值给某个接口：
Runnable task = () -> {
    // 这里其实是Runnable接口的匿名内部类，我们在编写run方法。
    System.out.println("hello lambda!");
};
new Thread(task).start();
```

#### 隐式final

Lambda表达式的实质其实还是匿名内部类，而匿名内部类在访问外部局部变量时，要求变量必须声明为`final`！不过我们在使用Lambda表达式时无需声明`final`，这并不是说违反了匿名内部类的规则，因为Lambda底层会隐式的把变量设置为`final`，在后续的操作中，一定不能修改该变量：

正确示范：

```java
// 定义一个局部变量
int num = -1;
Runnable r = () -> {
    // 在Lambda表达式中使用局部变量num，num会被隐式声明为final
    System.out.println(num);
};
new Thread(r).start();// -1
```

错误案例：

```java
// 定义一个局部变量
int num = -1;
Runnable r = () -> {
    // 在Lambda表达式中使用局部变量num，num会被隐式声明为final，不能进行任何修改操作
    System.out.println(num++);
};
new Thread(r).start();//报错
```

### 函数式接口

- Lambda表达式是接口的匿名内部类的简写形式
- 接口必须满足：内部只有一个函数

其实这样的接口，我们称为函数式接口，我们学过的`Runnable`、`Comparator`都是函数式接口的典型代表。但是在实践中，函数接口是非常脆弱的，只要有人在接口里添加多一个方法，那么这个接口就不是函数接口了，就会导致编译失败。Java 8提供了一个特殊的注解`@FunctionalInterface`来克服上面提到的脆弱性并且显示地表明函数接口。而且jdk8版本中，对很多已经存在的接口都添加了`@FunctionalInterface`注解，例如`Runnable`接口。

### 方法引用

方法引用使得开发者可以将已经存在的方法作为变量来传递使用。方法引用可以和Lambda表达式配合使用。

#### 语法：

总共有四类方法引用：

| 语法                   | 描述                             |
| ---------------------- | -------------------------------- |
| 类名::静态方法名       | 类的静态方法的引用               |
| 类名::非静态方法名     | 类的非静态方法的引用             |
| 实例对象::非静态方法名 | 类的指定实例对象的非静态方法引用 |
| 类名::new              | 类的构造方法引用                 |



### 默认方法

- 默认方法使得开发者可以在 不破坏二进制兼容性的前提下，往现存接口中添加新的方法，即不强制那些实现了该接口的类也同时实现这个新加的方法。

- 默认方法和抽象方法之间的区别在于抽象方法需要实现，而默认方法不需要。接口提供的默认方法会被接口的实现类继承或者覆写，例子代码如下：


```java
private interface Defaulable {
    // Interfaces now allow default methods, the implementer may or 
    // may not implement (override) them.
    default String notRequired() { 
        return "Default implementation"; 
    }        
}
```
### Optional

Java应用中最常见的bug就是空值异常。

`Optional`仅仅是一个容器，可以存放T类型的值或者`null`。它提供了一些有用的接口来避免显式的`null`检查，可以参考Java 8官方文档了解更多细节。

接下来看一点使用Optional的例子：可能为空的值或者某个类型的值：


```java
Optional< String > firstName = Optional.of( "Tom" );
System.out.println( "First Name is set? " + firstName.isPresent() );        
System.out.println( "First Name: " + firstName.orElseGet( () -> "[none]" ) ); 
System.out.println( firstName.map( s -> "Hey " + s + "!" ).orElse( "Hey Stranger!" ) );
System.out.println();
```

这个例子的输出是：


```
First Name is set? true
First Name: Tom
Hey Tom!
```

如果`Optional`实例持有一个非空值，则`isPresent()`方法返回`true`，否则返回`false`；如果`Optional`实例持有`null`，`orElseGet()`方法可以接受一个lambda表达式生成的默认值；`map()`方法可以将现有的`Optional`实例的值转换成新的值；`orElse()`方法与`orElseGet()`方法类似，但是在持有null的时候返回传入的默认值，而不是通过Lambda来生成。

### Streams

新增的Stream API（java.util.stream）将生成环境的函数式编程引入了Java库中。这是目前为止最大的一次对Java库的完善，以便开发者能够写出更加有效、更加简洁和紧凑的代码。Steam API极大得简化了集合操作。

首先看一个问题：在这个task集合中一共有多少个OPEN状态的？计算出它们的points属性和。在Java 8之前，要解决这个问题，则需要使用foreach循环遍历task集合；但是在Java 8中可以利用steams解决：包括一系列元素的列表，并且支持顺序和并行处理。


```java
// Calculate total points of all active tasks using sum()
final long totalPointsOfOpenTasks = tasks
    .stream()
    .filter( task -> task.getStatus() == Status.OPEN )
    .mapToInt( Task::getPoints )
    .sum();

System.out.println( "Total points: " + totalPointsOfOpenTasks );
```

运行这个方法的控制台输出是：


```
Total points: 18
```

这里有很多知识点值得说。首先，`tasks`集合被转换成`steam`表示；其次，在`steam`上的`filter`操作会过滤掉所有`CLOSED`的`task`；第三，`mapToInt`操作基于`tasks`集合中的每个`task`实例的`Task::getPoints`方法将`task`流转换成`Integer`集合；最后，通过`sum`方法计算总和，得出最后的结果。

在学习下一个例子之前，还需要记住一些steams（点此更多细节）的知识点。Steam之上的操作可分为中间操作和晚期操作。

中间操作会返回一个新的steam——执行一个中间操作（例如filter）并不会执行实际的过滤操作，而是创建一个新的steam，并将原steam中符合条件的元素放入新创建的steam。

晚期操作（例如forEach或者sum），会遍历steam并得出结果或者附带结果；在执行晚期操作之后，steam处理线已经处理完毕，就不能使用了。在几乎所有情况下，晚期操作都是立刻对steam进行遍历。

steam的另一个价值是创造性地支持并行处理（parallel processing）。对于上述的tasks集合，我们可以用下面的代码计算所有task的points之和：


```java
// Calculate total points of all tasks
final double totalPoints = tasks
   .stream()
   .parallel()
   .map( task -> task.getPoints() ) // or map( Task::getPoints ) 
   .reduce( 0, Integer::sum );

System.out.println( "Total points (all tasks): " + totalPoints );
```

这里我们使用parallel方法并行处理所有的task，并使用reduce方法计算最终的结果。控制台输出如下：


```
Total points（all tasks）: 26.0
```

对于一个集合，经常需要根据某些条件对其中的元素分组。利用steam提供的API可以很快完成这类任务，代码如下：


```java
// Group tasks by their status
final Map< Status, List< Task > > map = tasks
    .stream()
    .collect( Collectors.groupingBy( Task::getStatus ) );
System.out.println( map );
```

控制台的输出如下：


```java
{CLOSED=[[CLOSED, 8]], OPEN=[[OPEN, 5], [OPEN, 13]]}
```

最后一个关于tasks集合的例子问题是：如何计算集合中每个任务的点数在集合中所占的比重，具体处理的代码如下：


```java
// Calculate the weight of each tasks (as percent of total points) 
final Collection< String > result = tasks
    .stream()                                        // Stream< String >
    .mapToInt( Task::getPoints )                     // IntStream
    .asLongStream()                                  // LongStream
    .mapToDouble( points -> points / totalPoints )   // DoubleStream
    .boxed()                                         // Stream< Double >
    .mapToLong( weigth -> ( long )( weigth * 100 ) ) // LongStream
    .mapToObj( percentage -> percentage + "%" )      // Stream< String> 
    .collect( Collectors.toList() );                 // List< String > 

System.out.println( result );
```

控制台输出结果如下：


```
[19%, 50%, 30%]

```

最后，正如之前所说，Steam API不仅可以作用于Java集合，传统的IO操作（从文件或者网络一行一行得读取数据）可以受益于steam处理，这里有一个小例子：


```java
final Path path = new File( filename ).toPath();
try( Stream< String > lines = Files.lines( path, StandardCharsets.UTF_8 ) ) {
    lines.onClose( () -> System.out.println("Done!") ).forEach( System.out::println );
}
```

Stream的方法`onClose()` 返回一个等价的有额外句柄的Stream，当Stream的`close()`方法被调用的时候这个句柄会被执行。Stream API、Lambda表达式还有接口默认方法和静态方法支持的方法引用，是Java 8对软件开发的现代范式的响应。

### 并行数组
Java8版本新增了很多新的方法，用于支持并行数组处理。最重要的方法是`parallelSort()`，可以显著加快多核机器上的数组排序。下面的例子论证了parallexXxx系列的方法：


```java
package com.javacodegeeks.java8.parallel.arrays;

import java.util.Arrays;
import java.util.concurrent.ThreadLocalRandom;

public class ParallelArrays {
    public static void main( String[] args ) {
        long[] arrayOfLong = new long [ 20000 ];        

        Arrays.parallelSetAll( arrayOfLong, 
            index -> ThreadLocalRandom.current().nextInt( 1000000 ) );
        Arrays.stream( arrayOfLong ).limit( 10 ).forEach( 
            i -> System.out.print( i + " " ) );
        System.out.println();

        Arrays.parallelSort( arrayOfLong );        
        Arrays.stream( arrayOfLong ).limit( 10 ).forEach( 
            i -> System.out.print( i + " " ) );
        System.out.println();
    }
}
```

上述这些代码使用parallelSetAll()方法生成20000个随机数，然后使用parallelSort()方法进行排序。这个程序会输出乱序数组和排序数组的前10个元素。

### 日期、时间操作
在Java8之前，日期时间API一直被开发者诟病，包括：java.util.Date是可变类型，SimpleDateFormat非线程安全等问题。故此，Java8引入了一套全新的日期时间处理API，新的API基于ISO标准日历系统。

#### SimpleDateFormat线程不安全原因
- parse 方法为什么不线程安全
    - 1.有一个共享变量calendar，而这个共享变量的访问没有做到线程安全
    - 2.parse方法生成CalendarBuilder，然后通过CalendarBuilder 设值到calendar，最后calendar.getTime();

```java
private StringBuffer format(Date date, StringBuffer toAppendTo,
                            FieldDelegate delegate) {
    // 这里已经彻底毁坏线程的安全性
    calendar.setTime(date);

    boolean useDateFormatSymbols = useDateFormatSymbols();

    for (int i = 0; i < compiledPattern.length; ) {
        int tag = compiledPattern[i] >>> 8;
        int count = compiledPattern[i++] & 0xff;
        if (count == 255) {
            count = compiledPattern[i++] << 16;
            count |= compiledPattern[i++];
        }
...
```
> 相关类说明

```
Instant         时间戳
Duration        持续时间、时间差
LocalDate       只包含日期，比如：2018-09-24
LocalTime       只包含时间，比如：10:32:10
LocalDateTime   包含日期和时间，比如：2018-09-24 10:32:10
Peroid          时间段
ZoneOffset      时区偏移量，比如：+8:00
ZonedDateTime   带时区的日期时间
Clock           时钟，可用于获取当前时间戳
java.time.format.DateTimeFormatter      时间格式化类
```

　Java 8中的 LocalDate 用于表示当天日期。和java.util.Date不同，它只有日期，不包含时间。

```java
public static void main(String[] args) {
　　LocalDate now = LocalDate.now();
　　System.out.println("当前日期=" + date);
    LocalDate date = LocalDate.of(2000, 1, 1);
    // 获取年月日信息
    System.out.printf("年=%d， 月=%d， 日=%d", date.getYear(), date.getMonthValue(), date.getDayOfMonth());
    // 比较两个日期是否相等
    System.out.println("日期是否相等=" + now.equals(date));
}
```

* 获取当前时间

> Java 8中的 LocalTime 用于表示当天时间。和java.util.Date不同，它只有时间，不包含日期。

> Java8提供了新的plusXxx()方法用于计算日期时间增量值，替代了原来的add()方法。新的API将返回一个全新的日期时间示例，需要使用新的对象进行接收。

```java
public static void main(String[] args) {
        
　　　　 // 时间增量
        LocalTime time = LocalTime.now();
        LocalTime newTime = time.plusHours(2);
        System.out.println("newTime=" + newTime);
        
　　　　　// 日期增量
        LocalDate date = LocalDate.now();
        LocalDate newDate = date.plus(1, ChronoUnit.WEEKS);
        System.out.println("newDate=" + newDate);
        
}
```

> Java8提供了isAfter()、isBefore()用于判断当前日期时间和指定日期时间的比较

```java
    public static void main(String[] args) {
        
        LocalDate now = LocalDate.now();
        
        LocalDate date1 = LocalDate.of(2000, 1, 1);
        if (now.isAfter(date1)) {
            System.out.println("千禧年已经过去了");
        }
        
        LocalDate date2 = LocalDate.of(2020, 1, 1);
        if (now.isBefore(date2)) {
            System.out.println("2020年还未到来");
        }
        
    }
```

> Java 8不仅分离了日期和时间，也把时区分离出来了。现在有一系列单独的类如ZoneId来处理特定时区，ZoneDateTime类来表示某时区下的时间。

```java
    public static void main(String[] args) {
        
        // 上海时间
        ZoneId shanghaiZoneId = ZoneId.of("Asia/Shanghai");
        ZonedDateTime shanghaiZonedDateTime = ZonedDateTime.now(shanghaiZoneId);
        
        // 东京时间
        ZoneId tokyoZoneId = ZoneId.of("Asia/Tokyo");
        ZonedDateTime tokyoZonedDateTime = ZonedDateTime.now(tokyoZoneId);
        
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
        System.out.println("上海时间: " + shanghaiZonedDateTime.format(formatter));
        System.out.println("东京时间: " + tokyoZonedDateTime.format(formatter));
        
    }

```

#### 使用预定义格式解析与格式化日期

```java
public static void main(String[] args) {
        
        // 解析日期
        String dateText = "20180924";
        LocalDate date = LocalDate.parse(dateText, DateTimeFormatter.BASIC_ISO_DATE);
        System.out.println("格式化之后的日期=" + date);
        
        // 格式化日期
        dateText = date.format(DateTimeFormatter.ISO_DATE);
        System.out.println("dateText=" + dateText);
        
    }
```

#### 日期和字符串的相互转换

```java
    public static void main(String[] args) {
        
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
        
        // 日期时间转字符串
        LocalDateTime now = LocalDateTime.now();
        String nowText = now.format(formatter);
        System.out.println("nowText=" + nowText);
        
        // 字符串转日期时间
        String datetimeText = "1999-12-31 23:59:59";
        LocalDateTime datetime = LocalDateTime.parse(datetimeText, formatter);
        System.out.println(datetime);
        
    }
```



