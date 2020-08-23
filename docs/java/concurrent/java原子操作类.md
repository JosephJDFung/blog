##  Java原子操作类

- Atomic包提供了以下3个类。
    - AtomicBoolean：原子更新布尔类型。 
    - AtomicInteger：原子更新整型。 
    - AtomicLong：原子更新长整型。

- 原子更新数组
    - AtomicIntegerArray：原子更新整型数组里的元素。 
    - AtomicLongArray：原子更新长整型数组里的元素。 
    - AtomicReferenceArray：原子更新引用类型数组里的元素。 
    - AtomicIntegerArray类主要是提供原子的方式更新数组里的整型，其常用方法如下。 
        - int addAndGet（int i，int delta）：以原子方式将输入值与数组中索引i的元素相加。 
        - boolean compareAndSet（int i，int expect，int update）：如果当前值等于预期值，则以原子 方式将数组位置i的元素设置成update值。

#### 原子更新引用类型
* 原子更新基本类型的AtomicInteger，只能更新一个变量，如果要原子更新多个变量，就需 要使用这个原子更新引用类型提供的类。Atomic包提供了以下3个类。

- AtomicReference：原子更新引用类型。 
- AtomicReferenceFieldUpdater：原子更新引用类型里的字段。 
- AtomicMarkableReference：原子更新带有标记位的引用类型。可以原子更新一个布尔类 型的标记位和引用类型。构造方法是AtomicMarkableReference（V initialRef，boolean initialMark）。

#### 原子更新字段类

- 如果需原子地更新某个类里的某个字段时，就需要使用原子更新字段类，Atomic包提供 了以下3个类进行原子字段更新。 
    - AtomicIntegerFieldUpdater：原子更新整型的字段的更新器。 
    - AtomicLongFieldUpdater：原子更新长整型字段的更新器
    - AtomicStampedReference：原子更新带有版本号的引用类型。该类将整数值与引用关联起来，可用于原子的更新数据和数据的版本号，可以解决使用CAS进行原子更新时可能出现的 ABA问题。

* 要想原子地更新字段类需要两步。第一步，因为原子更新字段类都是抽象类，每次使用的时候必须使用静态方法newUpdater()创建一个更新器，并且需要设置想要更新的类和属性。第二步，更新类的字段（属性）必须使用public volatile修饰符。