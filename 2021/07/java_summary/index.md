# 一张思维导图看 Java 【持续迭代】

## 概述
总结自己的 java 知识，按照编码 -> 运行，画了一张图，xmind 导出的图比较大，后续持续更新，迭代这部分的内容。

![java](/img/java/java_summary.png)

<p style="text-align:center;color:#8EC0E4;font-size:1.5em;font-weight: bold;">
下面是一些常见的知识,将会慢慢补充进思维导图内
</p>

## HashMap 原理
可参考的文章
- [HashMap原理，循环链表是如何产生的](https://zhuanlan.zhihu.com/p/200997545)
-  [HashMap? ConcurrentHashMap? 相信看完这篇没人能难住你！](https://crossoverjie.top/2018/07/23/java-senior/ConcurrentHashMap/)
- [转换红黑树的阈值为何设置为8](https://zhuanlan.zhihu.com/p/263523069)
- [官方comment](https://hg.openjdk.org/jdk8/jdk8/jdk/file/687fd7c7986d/src/share/classes/java/util/HashMap.java#l174)


### 结构
- Java 8 之前：数组 + 链表
- Java 8 之后：数组 + 链表；

#### 存储的对象
#### Java7
Entry<K,V>[] table，Entry 是 HashMap 中的一个静态内部类，它有key、value、next、hash（key的hashcode）成员变量。
#### Java8
- TREEIFY_THRESHOLD 用于判断是否需要将链表转换为红黑树的阈值。
- HashEntry 修改为 Node。

### 负载因子：
- 给定的默认容量为 16，负载因子为 0.75。Map 在使用过程中不断的往里面存放数据，当数量达到了 16 * 0.75 = 12 就需要将当前 16 的容量进行扩容，而扩容这个过程涉及到 rehash、复制数据等操作，所以非常消耗性能。
- 因此通常建议能提前预估 HashMap 的大小最好，尽量的减少扩容带来的性能损耗。

### put 操作
#### Java7
-   判断当前数组是否需要初始化。
-   如果 key 为空，则 put 一个空值进去。
-   根据 key 计算出 hashcode。
-   根据计算出的 hashcode 定位出所在桶。
-   如果桶是一个链表则需要遍历判断里面的 hashcode、key 是否和传入 key 相等，如果相等则进行覆盖，并返回原来的值。
-  如果桶是空的，说明当前位置没有数据存入，新增一个 Entry 对象写入当前位置。
- 当调用 addEntry 写入 Entry 时需要判断是否需要扩容。如果需要就进行两倍扩充，并将当前的 key 重新 hash 并定位。而在 createEntry 中会将当前位置的桶传入到新建的桶中，如果当前桶有值就会在位置形成链表。

#### Java8
> 当 Hash 冲突严重时，在桶上形成的链表会变的越来越长，这样在查询时的效率就会越来越低；时间复杂度为 O(N)，因此 1.8 中重点优化了这个查询效率。
-   判断当前桶是否为空，空的就需要初始化（在resize方法 中会判断是否进行初始化）。
-   根据当前 key 的 hashcode 定位到具体的桶中并判断是否为空，为空表明没有 Hash 冲突就直接在当前位置创建一个新桶即可。
-   如果当前桶有值（ Hash 冲突），那么就要比较当前桶中的 key、key 的 hashcode 与写入的 key 是否相等，相等就赋值给 e,在第 8 步的时候会统一进行赋值及返回。
-   如果当前桶为红黑树，那就要按照红黑树的方式写入数据。
-   如果是个链表，就需要将当前的 key、value 封装成一个新节点写入到当前桶的后面（形成链表）。
-   接着判断当前链表的大小是否大于预设的阈值，大于时就要转换为红黑树。
-   如果在遍历过程中找到 key 相同时直接退出遍历。
-   如果 e != null 就相当于存在相同的 key,那就需要将值覆盖。
-   最后判断是否需要进行扩容。


### get 操作
#### Java7
-   首先也是根据 key 计算出 hashcode，然后定位到具体的桶中。
-   判断该位置是否为链表。
-   不是链表就根据 key、key 的 hashcode 是否相等来返回值。
-   为链表则需要遍历直到 key 及 hashcode 相等时候就返回值。
-   啥都没取到就直接返回 null 。
#### Java8
-   首先将 key hash 之后取得所定位的桶。
-   如果桶为空则直接返回 null 。
-   否则判断桶的第一个位置(有可能是链表、红黑树)的 key 是否为查询的 key，是就直接返回 value。
-   如果第一个不匹配，则判断它的下一个是红黑树还是链表。
-   红黑树就按照树的查找方式返回值。
-   不然就按照链表的方式遍历匹配返回值。

### HashMap 自身的缺陷

#### 循环链表

修改为红黑树之后查询效率直接提高到了 O(logn)。但是 HashMap 原有的问题也都存在，比如在并发场景下使用时容易出现死循环：
在 HashMap 扩容的时候会调用 resize() 方法，就是这里的并发操作容易在一个桶上形成环形链表；这样当获取一个不存在的 key 时，计算出的 index 正好是环形链表的下标就会出现死循环：在 1.7 中 hash 冲突采用的头插法形成的链表，在并发条件下会形成循环链表，一旦有查询落到了这个链表上，当获取不到值时就会死循环。

#### 死循环
形成循环链表后，一旦有查询落到了这个链表上，当获取不到值时就会死循环。

#### 非线程安全
就如上所说的两个问题，都是并发导致的问题。



### 扩容相关

#### 何时扩容
当向容器添加元素的时候，会判断当前容器的元素个数，如果大于等于阈值---即大于当前数组的长度乘以加载因子的值的时候，就要自动扩容。

#### 扩容的算法是什么
扩容(resize)就是重新计算容量，向HashMap对象里不停的添加元素，而HashMap对象内部的数组无法装载更多的元素时，对象就需要扩大数组的长度，以便能装入更多的元素。当然Java里的数组是无法自动扩容的，方法是使用一个新的数组代替已有的容量小的数组。

### Hashmap如何解决散列碰撞？
Java中HashMap是利用“拉链法”处理HashCode的碰撞问题。在调用HashMap的put方法或get方法时，都会首先调用hashcode方法，去查找相关的key，当有冲突时，再调用equals方法。hashMap基于hasing原理，我们通过put和get方法存取对象。当我们将键值对传递给put方法时，他调用键对象的hashCode()方法来计算hashCode，然后找到bucket（哈希桶）位置来存储对象。当获取对象时，通过键对象的equals()方法找到正确的键值对，然后返回值对象。HashMap使用链表来解决碰撞问题，当碰撞发生了，对象将会存储在链表的下一个节点中。hashMap在每个链表节点存储键值对对象。当两个不同的键却有相同的hashCode时，他们会存储在同一个bucket位置的链表中。键对象的equals()来找到键值对。

> 额外补充，其他解决办法是，
> 开放地址法，但是这个方法浪费存储空间，空间利用率很低。适用于规模很小的场景；
> 再hash法，进行二次、三次hash；


### 数组大小为何是 2 的倍数
1. 与运算比或运算效率高，提高速度；
2. 减少hash冲突；

整体为了提高散列的均匀性。


### 红黑树
#### 为何需要转换成红黑树
每次遍历一个链表，平均查找的时间复杂度是 O(n)，n 是链表的长度。红黑树有和链表不一样的查找性能，由于红黑树有自平衡的特点，可以防止不平衡情况的发生，所以可以始终将查找的时间复杂度控制在 O(log(n))。最初链表还不是很长，所以可能 O(n) 和 O(log(n)) 的区别不大，但是如果链表越来越长，那么这种区别便会有所体现。所以为了提升查找性能，需要把链表转化为红黑树的形式。

#### 那为什么不一开始就用红黑树，反而要经历一个转换的过程呢？

因为树节点(TreeNodes)所占的空间是普通节点Node的两倍，所以我们只有在桶中包含足够的节点时才使用树节点(请参阅TREEIFY_THRESHOLD)(只有在同一个哈希桶中的节点数量大于等于TREEIFY_THRESHOLD时，才会将该桶中原来的链式存储的节点转化为红黑树的树节点)。并且当桶中的节点数过少时 (由于移除或调整)，树节点又会被转换回普通节点(当桶中的节点数量过少时，原来的红黑树树节点又会转化为链式存储的普通节点)，以便节省空间。　　


#### 阈值的选取
>体现了时间和空间平衡的思想

如果 hashCode 分布良好，也就是 hash 计算的结果离散好的话，那么红黑树这种形式是很少会被用到的，因为各个值都均匀分布，很少出现链表很长的情况。在理想情况下，桶(bins)中的节点数概率(链表长度)符合泊松分布，当桶中节点数(链表长度)为 8 的时候，概率仅为 0.00000006。这是一个小于千万分之一的概率，通常我们的 Map 里面是不会存储这么多的数据的，所以通常情况下，并不会发生从链表向红黑树的转换。

阈值为何是 8 和 6，这个是一个概率问题，是用泊松分布计算出来。


## ConcurrentHashMap
- [可以参考的文章](https://mp.weixin.qq.com/s/My4P_BBXDnAGX1gh630ZKw)
HashTable是一个线程安全的类，它使用synchronized来锁住整张Hash表来实现线程安全，即每次锁住整张表让线程独占，相当于所有线程进行读写时都去竞争一把锁，导致效率非常低下。ConcurrentHashMap可以做到读取数据不加锁，并且其内部的结构可以让其在进行写操作的时候能够将锁的粒度保持地尽量地小，允许多个修改操作并发进行，其关键在于使用了**锁分段技术**。它使用了多个锁来控制对hash表的不同部分进行的修改。对于JDK1.7版本的实现, ConcurrentHashMap内部使用段(Segment)来表示这些不同的部分，每个段其实就是一个小的Hashtable，它们有自己的锁。只要多个修改操作发生在不同的段上，它们就可以并发进行。JDK1.8的实现降低锁的粒度，**JDK1.7版本锁的粒度是基于Segment的**，包含多个HashEntry，而**JDK1.8锁的粒度就是HashEntry（首节点）**。
JAVA7之前ConcurrentHashMap主要采用锁机制，在对某个Segment进行操作时，将该Segment锁定，不允许对其进行非查询操作，而在**JAVA8之后采用CAS无锁算法**，这种乐观操作在完成前进行判断，如果符合预期结果才给予执行，对并发操作提供良好的优化.

### Put
#### Java 7
首先是通过 key 定位到 Segment，之后在对应的 Segment 中进行具体的 put。
-  虽然 HashEntry 中的 value 是用 volatile 关键词修饰的，但是并不能保证并发的原子性，所以 put 操作时仍然需要加锁处理。
-  首先第一步的时候会尝试获取锁，如果获取失败肯定就有其他线程存在竞争，则利用 scanAndLockForPut() 自旋获取锁:
- 尝试自旋获取锁。 如果重试的次数达到了 MAX_SCAN_RETRIES 则改为阻塞锁获取，保证能获取成功。
- 将当前 Segment 中的 table 通过 key 的 hashcode 定位到 HashEntry。
- 遍历该 HashEntry，如果不为空则判断传入的 key 和当前遍历的 key 是否相等，相等则覆盖旧的 value。
- 为空则需要新建一个 HashEntry 并加入到 Segment 中，同时会先判断是否需要扩容。
- 最后会使用unlock()解除当前 Segment 的锁。

#### Java 8 
-   根据 key 计算出 hashcode 。
-   判断是否需要进行初始化。
-   如果当前 key 定位出的 Node 为空表示当前位置可以写入数据，利用 CAS 尝试写入，失败则自旋保证成功。
-   如果当前位置的 hashcode == MOVED == -1,则需要进行扩容。
-   如果都不满足，则利用 synchronized 锁写入数据。
-   最后，如果数量大于 TREEIFY_THRESHOLD 则要转换为红黑树。


### Get
#### 1.7
-   只需要将 Key 通过 Hash 之后定位到具体的 Segment ，再通过一次 Hash 定位到具体的元素上。
-   由于 HashEntry 中的 value 属性是用 volatile 关键词修饰的，保证了内存可见性，所以每次获取时都是最新值。
-   ConcurrentHashMap 的 get 方法是非常高效的，因为整个过程都不需要加锁。


#### 1.8
-   根据计算出来的 hashcode 寻址，如果就在桶上那么直接返回值。
-   如果是红黑树那就按照树的方式获取值。
-   就不满足那就按照链表的方式遍历获取值。



## Java 中的 map 与 Android 的 SparseArray 的比较

SparseArray 比HashMap更省内存，在某些条件下性能更好，主要是因为它避免了对key的自动装箱（int转为Integer类型），它内部则是通过两个数组来进行数据存储的，一个存储 key，另外一个存储 value，为了优化性能，它内部对数据还采取了压缩的方式来表示稀疏数组的数据，从而节约内存空间，我们从源码中可以看到key和value分别是用数组表示：

private int[] mKeys;
private Object[] mValues;

key 的下标 index 和 values 的下标 index 是一一对应的，在查询的时候，可以根据 key 进行二分查找，查找到 index 就是 values 的下标，效率较高。

### ArrayMap
还有一个类是 ArrayMap，结构也是两个数组，一个存储 hash，一个存储  value；
查询的时候，也是二分查找，查找数组里存储的hash值和 key 的 hash 值一致的 下标；


### 对比

- 0-1,000时，SparseArray 具备更好的性能：

1、如果key的类型已经确定为int类型，那么使用SparseArray，因为它避免了自动装箱的过程，如果key为long类型，它还提供了一个LongSparseArray来确保key为long类型时的使用
2、如果key类型为其它的类型，则使用ArrayMap。


- 1,000 - 10,000 
 hashmap 具备更好的插入优势；SparseArray 具备更好的查询优势；

- 10,000 以上
hashmap 优势降低。









## 设计模式
### 创建型（5）
#### 单例模式
[参考的文章](https://www.cnblogs.com/xz816111/p/8470048.html)

双重检查锁的意义
- 外层的判空，是为了提升性能；
- 内层的加锁是为了原子话；
- 变量 volatile 是为了防止指令重排；

通过静态内部类实现单例模式有哪些优点？
- 不用 synchronized ，节省时间。
- 调用 getInstance() 的时候才会创建对象，不调用不创建，节省空间，这有点像传说中的懒汉式。

builder 模式，工厂模式， 抽象工厂模式


### 结构型（7）
适配器模式、装饰器模式、代理模式、外观模式、桥接模式、组合模式、享元模式


### 行为型（11）
策略模式、模板方法模式、观察者模式、迭代子模式、责任链模式、命令模式、备忘录模式、状态模式、访问者模式、中介者模式、解释器模式。



### 六大原则
> 开闭原则是基础

#### 开放封闭原则（Open Close Principle）
> 对扩展开放，对修改封闭

-   原则思想：尽量通过扩展软件实体来解决需求变化，而不是通过修改已有的代码来完成变化
-   描述：一个软件产品在生命周期内，都会发生变化，既然变化是一个既定的事实，我们就应该在设计的时候尽量适应这些变化，以提高项目的稳定性和灵活性。
-   优点：单一原则告诉我们，每个类都有自己负责的职责，里氏替换原则不能破坏继承关系的体系。

#### 里氏代换原则（Liskov Substitution Principle）
> 高度抽象的继承体系，即使增加子类，不需要修改父类

-   原则思想：使用的基类可以在任何地方使用继承的子类，完美的替换基类。
-   大概意思是：子类可以扩展父类的功能，但不能改变父类原有的功能。子类可以实现父类的抽象方法，但不能覆盖父类的非抽象方法，子类中可以增加自己特有的方法。
-   优点：增加程序的健壮性，即使增加了子类，原有的子类还可以继续运行，互不影响。

#### 依赖倒转原则（Dependence Inversion Principle）
> 面向接口编程，接口需要高度的抽象

-   依赖倒置原则的核心思想是面向接口编程.
-   依赖倒转原则要求我们在程序代码中传递参数时或在关联关系中，尽量引用层次高的抽象层类，
-   这个是开放封闭原则的基础，具体内容是：对接口编程，依赖于抽象而不依赖于具体。

#### 接口隔离原则（Interface Segregation Principle）‘
> 接口的信息要合适，不需要含有复杂冗余的方法

-   这个原则的意思是：使用多个隔离的接口，比使用单个接口要好。还是一个降低类之间的耦合度的意思，从这儿我们看出，其实设计模式就是一个软件的设计思想，从大型软件架构出发，为了升级和维护方便。所以上文中多次出现：降低依赖，降低耦合。
-   例如：支付类的接口和订单类的接口，需要把这俩个类别的接口变成俩个隔离的接口

#### 迪米特法则（最少知道原则）（Demeter Principle）
> 知道的越少越好
-   原则思想：一个对象应当对其他对象有尽可能少地了解，简称类间解耦
-   大概意思就是一个类尽量减少自己对其他对象的依赖，原则是低耦合，高内聚，只有使各个模块之间的耦合尽量的低，才能提高代码的复用率。
-   优点：低耦合，高内聚。

#### 单一职责原则（Principle of single responsibility）
> 聚焦
-   原则思想：一个方法只负责一件事情。
-   描述：单一职责原则很简单，一个方法 一个类只负责一个职责，各个职责的程序改动，不影响其它程序。 这是常识，几乎所有程序员都会遵循这个原则。
-   优点：降低类和类的耦合，提高可读性，增加可维护性和可拓展性，降低可变性的风险。



## 动态代理原理及实现

- 静态代理通常只代理一个类，动态代理是代理一个接口下的多个实现类。
- 静态代理事先知道要代理的是什么，而动态代理不知道要代理什么东西，只有在运行时才知道。

动态代理是实现 JDK 里的 InvocationHandler 接口的 invoke 方法，但注意的是代理的是接口，也就是你的业务类必须要实现接口，通过 Proxy 里的 newProxyInstance 得到代理对象。还有一种动态代理 CGLIB，代理的是类，不需要业务类继承接口，通过派生的子类来实现代理。通过在运行时，动态修改字节码达到修改类的目的。AOP 编程就是基于动态代理实现的，比如著名的 Spring 框架、Hibernate 框架等等都是动态代理的使用例子。


```java
final List<String> list = new ArrayList<String>();
List<String> proxyInstance =
        (List<String>) Proxy.newProxyInstance(list.getClass().getClassLoader(),
                list.getClass().getInterfaces(),
                new InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        return method.invoke(list, args);
                    }
                });
    proxyInstance.add("你好");
    System.out.println(list);
```



- 问题1：为什么 JDK 动态代理要基于接口实现？而不是基于继承来实现？
解答：因为 JDK 动态代理生成的对象默认是继承 Proxy ，Java 不支持多继承，所以 JDK 动态代理要基于接口来实现。

- 问题2：JDK 动态代理中，目标对象调用自己的另一个方法，会经过代理对象么？
解答：内部调用方法使用的对象是目标对象本身，被调用的方法不会经过代理对象。



- 应用
Retrofit 应用：Retrofit 通过动态代理，为我们定义的请求接口都生成一个动态代理对象，实现请求



## 静态嵌套类和内部类的区别
- 静态嵌套类：Static Nested Class是被声明为静态（static）的内部类，它可以不依赖于外部类实例被实例化。
- 内部类：需要在外部类实例化后才能实例化，其语法看起来挺诡异的。


## 反射
反射的核心是 JVM 在运行时才动态加载类或调用方法/访问属性，它不需要事先（写代码的时候或编译期）知道运行对象是谁。

Java 反射主要提供以下功能：
- 在运行时判断任意一个对象所属的类；
- 在运行时构造任意一个类的对象；
- 在运行时判断任意一个类所具有的成员变量和方法（通过反射甚至可以调用private方法）；
- 在运行时调用任意一个对象的方法

重点：是运行时而不是编译时。

要想解剖一个类,必须先要获取到该类的字节码文件对象。而解剖使用的就是Class类中的方法.所以先要获取到每一个字节码文件对应的Class类型的对象.


## 注解
三种类型的注解
- SOURCE：存在于源码级别，编译期可以读取，但是不会保存到 class 文件中；
- CLASS：编译级别，存在于 class 文件中，但是运行时会被丢弃；
- RUNTIME：存储在 class 文件中，并且 vm 运行时也会保留，可以通过反射获取；


```java
[@Retention(RetentionPolicy.RUNTIME)
public @interface Description {
    String value();
}](<@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface CustomAnnotation{
}>)
```


```java
@SupportedAnnotationTypes("com.example.CustomAnnotation")
public class CompileTimeAnnotationProcessor extends AbstractProcessor {

    @Override
    public boolean process(Set<? extends TypeElement> annotations, 
                           RoundEnvironment roundEnv) {
        Set<? extends Element> elements = roundEnv.getElementsAnnotatedWith(CustomAnnotation.class);
        for(Element e : elements){
            if(!e.getClass().equals(ParticularType.class)){
                processingEnv.getMessager().printMessage(Kind.ERROR,
                     "@CustomAnnotation annotated fields must be of type ParticularType");
            }
        }
        return true;
    }

}
```



## 泛型
> 参数化类型，操作的类型以参数的形式传递。可以用于类、接口、方法中。
> 目的是为了安全。


型是Java SE1.5的新特性，**泛型的本质是参数化类型，也就是说所操的数据类型被指定为一个参数**。这种参数类型可以用在类、接口和方法的创建中，分别称为泛型类、泛型接口、泛型方法。 Java语言引入泛型的好处是**安全简单**。

在Java SE 1.5之前，没有泛型的情况的下，通过对类型Object的引用来实现参数的“任意化”，“任意化”带来的缺点是要做显式的强制类型转换，而这种转换是要求开发者实际参数类型可以预知的情况下进行的。对于强制类型换错误的情况，编译器可能不提示错误，在运行的时候出现异常，这是一个**安全隐患**。

泛型的好处是在编译的时候检查类型安全，并且所有的转换都是自动和隐式的，提高代码的重用率。

1、泛型的类型参数只能是类类型（包括自定义类），不是简单类型。

2、同一种泛型可以对应多个版本（因为参数类型是不确的），不同版本的泛型类实例是不兼容的。

3、泛型的类型参数可以有多个。

4、泛型的参数类型可以使用 extends 语句，例如。习惯上称为“有界类型”。

5、泛型的参数类型还可以是通配符类型。例如
```java
Class<?> classType = Class.forName("java.lang.String");
```


### 泛型擦除以及相关的概念
> 泛型的信息只存在编译期间，等到进入 jvm 时候，已经被擦除；
> 擦除的时候，如果有指定上限，则使用上限，没指定，则使用 object；
> 

泛型信息只存在代码编译阶段，在进入JVM之前，与泛型关的信息都会被擦除掉。

在类型擦除的时候，如果泛型类里的类型参数没有指定上限，则会被转成Object类型，如果指定了上限，则会被传转换成对应的类型上限。

Java中的泛型基本上都是在编译器这个层次来实现的。生成的Java字节码中是不包含泛型中的类型信息的。使用泛型的时候加上的类型参数，会在编译器在编译的时候擦除掉。这个过程就称为类型擦除。

类型擦除引起的问题及解决方法：

1、先检查，在编译，以及检查编译的对象和引用传递的题

2、自动类型转换

3、类型擦除与多态的冲突和解决方法

4、泛型类型变量不能是基本数据类型

5、运行时类型查询

6、异常中使用泛型的问题

7、数组（这个不属于类型擦除引起的问题）

9、类型擦除后的冲突

10、泛型在静态方法和静态类中的问题



### 协变和逆变；大名鼎鼎的 PECS （producer extend，consumer super），往两头扩散；

将 collection 视为 product 还是 consumer



## 四种引用
- 强引用：
使用最普遍的引用。只要引用链没有断开，强引用就不会断开。当内存空间不足，抛出OutOfMemoryError终止程序也不会回收具有强引用的对象。通过将对象设置为null来弱化引用,使其被回收

- 软引用
对象处在有用但非必须的状态。只有当内存空间不足时, GC会回收该引用的对象的内存。可以用来实现高速缓存（作用）--比如网页缓存、图片缓存。

- 弱引用
弱引用就是只要JVM垃圾回收器发现了它，就会将之回收。非必须的对象,比软引用更弱一些。GC时会被回被回收的概率也不大,因为GC线程优先级比较低。适用于引用偶尔被使用且不影响垃圾收集的对象。

- 虚引用
不会决定对象的生命周期。任何时候都可能被垃圾收集器回收。跟踪对象被垃圾收集器回收的活动,起哨兵作用。

必须和引用队列ReferenceQueue联合使用。当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会把这个虚引用加入到与之 关联的引用队列中。

程序可以通过判断引用队列中是否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收。如果程序发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取必要的行动。用来跟踪对象的 gc 活动。



### LeakCanary 的应用
可以参考的文章
1. [通过一个 Java 演示原理](https://blog.csdn.net/c10WTiybQ1Ye3/article/details/122954616)；

LeakCanary 的检测原理是：
- 弱引用对象在其包装对象被回收后（弱引用对象创建时，传入了引用队列），该弱引用对象会被加到引用队列中（ReferenceQueue）。
- 通过在 ReferenceQueue 中检测是否有目标对象的弱引用对象存在，即可判断目标对象是否被回收。如果在队列中，则说明是回收了，不存在内存泄漏，反之则存在内存泄漏；


1. 为什么要放入空闲消息里面去执行？
因为gc就是发生在系统空闲的时候的，所以当空闲消息被执行的时候，大概率已经执行过一次gc了。

2. 为什么在空闲消息可以直接检测activity是否被回收？
跟问题1一样，空闲消息被执行的时候，大概率已经发生过gc，所以可以检测下gc后activity是否被回收。

3. 如果没有被回收，应该是已经泄漏了啊，为什么再次执行了一次gc，然后再去检测？
根据问题2，空闲消息被执行的时候，大概率已经发生过gc，但是也可能还没发生gc，那么此时activity没有被回收是正常的，所以我们手动再gc一下，确保发生了gc，再去检测activity是否被回收，从而100%的确定是否发生了内存泄漏。



## ThreadLocal
可以参考的文章
1. [ThreadLocal原理及魔数0x61c88647](https://www.cnblogs.com/hongdada/p/12108611.html)

ThreadLocal是一种线程封闭技术。ThreadLocal提供了get和set等访问接口或方法，这些方法为每个使用该变量的线程都**存有一份独立的副本**，因此get总是返回由**当前执行线程**在调用set时设置的最新值。

从ThreadLocal的set和get方法可以看出，它们所操作的对象都是当前线程的**localValues对象的table数组**，因此在不同线程中访问同一个ThreadLocal的set和get方法，它们对ThreadLocal所做的**读写操作仅限于各自线程的内部**，这就是为什么ThreadLocal可以在多个线程中互不干扰地存储和修改数据。


### example

```java
private static final ThreadLocal<Integer> threadIds = new ThreadLocal<>();
Integer id = threadIds.get();  
if (id == null) {  
    id = nextThreadId.getAndIncrement();  
    threadIds.set(id);  
}
```

一个 threadlocal 只能维护一个变量；等价于每一个线程私有变量都需要  new 一个 threadlocal 实例。


### 内存泄漏问题

threadLocalMap 使用 ThreadLocal 的弱引用作为 key，如果一个 ThreadLocal 不存在外部强引用时，Key(ThreadLocal) 势必会被 GC 回收，这样就会导致 ThreadLocalMap 中 key 为 null， 而 value 还存在着强引用，只有 thead 线程退出以后,value 的强引用链条才会断掉。
但如果当前线程再迟迟不结束的话，这些 key 为 null 的 Entry 的 value 就会一直存在一条强引用链：

`Thread Ref -> Thread -> ThreaLocalMap -> Entry -> value`

永远无法回收，造成内存泄漏。

就比如线程池里面的线程，线程都是复用的，那么之前的线程实例处理完之后，出于复用的目的线程依然存活，所以，ThreadLocal设定的value值被持有，导致内存泄露。

### ThreadLocal 正确的使用方法
- 每次使用完 ThreadLocal 都调用它的 remove() 方法清除数据。
- 将 ThreadLocal 变量定义成 private static，这样就一直存在 ThreadLocal 的强引用，也就能保证任何时候都能通过 ThreadLocal 的弱引用访问到Entry的value值，进而清除掉 。




### 使用场景
- 线程间的数据隔离
- 在Android中，Looper类就是利用了ThreadLocal的特性，保证每个线程只存在一个Looper对象。
- 当某些数据是**以线程为作用域并且不同线程具有不同的数据副本**的时候，就可以考虑采用ThreadLocal。


## 为什么Java里的匿名内部类只能访问final修饰的外部变量？
```java
public class TryUsingAnonymousClass {
    public void useMyInterface() {
        final Integer number = 123;
        System.out.println(number);

        MyInterface myInterface = new MyInterface() {
            @Override
            public void doSomething() {
                System.out.println(number);
            }
        };
        myInterface.doSomething();

        System.out.println(number);
    }
}

// 编译后的结果
class TryUsingAnonymousClass$1 implements MyInterface {
    private final TryUsingAnonymousClass this$0;
    private final Integer paramInteger;

    TryUsingAnonymousClass$1(TryUsingAnonymousClass this$0, Integer paramInteger) {
        this.this$0 = this$0;
        this.paramInteger = paramInteger;
    }

    public void doSomething() {
        System.out.println(this.paramInteger);
    }
}


```


因为匿名内部类最终会编译成一个单独的类，而被该类使用的变量会以构造函数参数的形式传递给该类，例如：Integer paramInteger，这个时候修改传入的参数并不能改动到外部类的变量。这变量相当于脱离了之前类的生命周期。

如果变量不定义成 final 的，paramInteger在匿名内部类被可以被修改，进而造成和外部的paramInteger不一致的问题，为了避免这种不一致的情况，因次Java规定匿名内部类只能访问final修饰的外部变量。



## ConcurrentModificationException

### 出现的原因：
在 ArrayList 中，在迭代遍历 list 的过程中，修改了 list 的大小时，就会抛出这个并发修改的异常。

### 解决方案
- Collections.synchronizedList(new ArrayList<>())，将对 list 的操作修改为线程安全的，带来的损失是性能损耗；
- 改成 CopyOnWriteArrayList （推荐这种）；



### CopyOnWriteArrayList 
```java
/**  
 * Appends the specified element to the end of this list. * * @param e element to be appended to this list  
 * @return {@code true} (as specified by {@link Collection#add})  
 */public boolean add(E e) {  
    synchronized (lock) {  
        Object[] elements = getArray();  
        int len = elements.length;  
        Object[] newElements = Arrays.copyOf(elements, len + 1);  
        newElements[len] = e;  
        setArray(newElements);  
        return true;  
    }  
}
```

针对修改 list 内容的变更，不在原有的数据上修改，而是进行 copy，修改完成之后，调用`setArray(newElements);`   重新指向新数据即可。



## java.util.concurrent 包的学习
1. [参考资料一](https://blog.csdn.net/wbwjx/article/details/57856045)



![脑图](https://img-blog.csdn.net/20170301212847989?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2J3ang=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


-   locks部分：显式锁(互斥锁和速写锁)相关；
-   atomic部分：原子变量类相关，是构建非阻塞算法的基础；
-   executor部分：线程池相关；
-   collections部分：并发容器相关；
-   tools部分：同步工具相关，如信号量、闭锁、栅栏等功能；


### 一些常见的类
#### BlockingQueue
此接口是一个线程安全的 存取实例的队列。
#### ArrayBlockingQueue
ArrayBlockingQueue 是一个有界的阻塞队列。
#### DelayQueue
DelayQueue 对元素进行持有直到一个特定的延迟到期，才能得到运行。注入其中的元素必须实现 java.util.concurrent.Delayed 接口。

#### LinkedBlockingQueue
内部以一个链式结构(链接节点)对其元素进行存储 。

#### PriorityBlockingQueue
一个无界的并发队列，它使用了和类 java.util.PriorityQueue 一样的排序规则。


#### SynchronousQueue
一个特殊的队列，它的内部同时只能够容纳单个元素。
-   如果该队列已有一元素的话，试图向队列中插入一个新元素的线程将会阻塞，直到另一个线程将该元素从队列中抽走。
-   如果该队列为空，试图向队列中抽取一个元素的线程将会阻塞，直到另一个线程向队列中插入了一条新的元素。

#### BlockingDeque
此接口表示一个线程安全放入和提取实例的双端队列。


#### LinkedBlockingDeque
LinkedBlockingDeque 是一个双端队列，可以从任意一端插入或者抽取元素的队列。

#### ConcurrentMap
一个能够对别人的访问(插入和提取)进行并发处理的 java.util.Map接口。 

#### ConcurrentNavigableMap
一个支持并发访问的 java.util.NavigableMap，它还能让它的子 map 具备并发访问的能力。

其中有 headMap tailMap  等方法，可以指定范围的子集。


#### CountDownLatch
CountDownLatch 是一个并发构造，它允许一个或多个线程等待一系列指定操作的完成。
-   CountDownLatch 以一个给定的数量初始化。countDown() 每被调用一次，这一数量就减一。
-   通过调用 await() 方法之一，线程可以阻塞等待这一数量到达零。


#### CyclicBarrier
CyclicBarrier 类是一种同步机制，它能够对处理一些算法的线程实现同步。
循环的障碍物，可以反复使用。

适用于，执行一组固定大小，且偶尔需要彼此相互等待的线程。

等待的线程是需要阻塞的，与 CountDownLatch 的区别是，CountDownLatch 只是进行了计数而已，不会被阻塞，CyclicBarrier 的线程是会被阻塞的。



#### Exchanger
- [参考资料](https://www.jianshu.com/p/990ae2ab1ae0)

Exchanger 是 JDK 1.5 开始提供的一个用于两个工作线程之间交换数据的封装工具类，简单说就是一个线程在完成一定的事务后想与另一个线程交换数据，则第一个先拿出数据的线程会一直等待第二个线程，直到第二个线程拿着数据到来时才能彼此交换对应数据。

#### Semaphore
控制许可集，控制多个共享资源的计数器。

-   保护一个重要(代码)部分防止一次超过 N 个线程进入。
-   在两个线程之间发送信号。


- 用于保护重要的数据
如果你将信号量用于保护一个重要部分，试图进入这一部分的代码通常会首先尝试获得一个许可，然后才能进入重要部分(代码块)，执行完之后，再把许可释放掉。

- 用于通信，在线程之间发送信号

如果你将一个信号量用于在两个线程之间传送信号，通常你应该用一个线程调用 acquire() 方法，而另一个线调用 release() 方法。
如果没有可用的许可，acquire() 调用将会阻塞，直到一个许可被另一个线程释放出来。
如果无法往信号量释放更多许可时，一个 release() 调用也会阻塞。

使用场景
1. 限制同时访问资源的线程总数；
2. 流量控制；


#### ReentrantLock
- [可参考的文章1](https://juejin.cn/post/6844904058302513159#heading-15)
- [可参考的文章2](https://stackoverflow.com/questions/11821801/why-use-a-reentrantlock-if-one-can-use-synchronizedthis)
- [可参考的文章3](https://blog.csdn.net/m0_45406092/article/details/113789547)
- [可参考的文章4](https://winterbe.com/posts/2015/04/30/java8-concurrency-tutorial-synchronized-locks-examples/)


> - ReentrantLock可以尝试获取锁与设置获取锁的时限，需要手动释放锁
> - synchronized不能扩展锁之外的方法或者块边界，尝试获取锁时不能中途取消，获取不到锁会阻塞，synchronized正常或异常退出会自动释放锁。

- 可重入锁，可以对同一把锁，多次上锁；
- 公平锁，FIFO，有队列，先进先出；默认是非公平的；每次 acquire 的时候，都检查等待最长时间的线程；

Synchronized 获取锁的行为是不公平的，并非是按照申请对象锁的先后时间分配锁的，每次对象锁被释放时，每个线程都有机会获得对象锁，这样有利于提高执行性能，但是也会造成线程饥饿现象。


##### ReentrantLock 常用的方法有哪些？
ReentrantLock 常见方法如下：

-   lock()：用于获取锁
-   unlock()：用于释放锁
-   tryLock()：尝试获取锁
-   getHoldCount()：查询当前线程执行 lock() 方法的次数
-   getQueueLength()：返回正在排队等待获取此锁的线程数
-   isFair()：该锁是否为公平锁


##### ReentrantLock 有哪些优势
- 灵活性，自主可控；
- 具备等待超时机制；
- 具备构造公平锁的能力；
- 可以感知等待锁的线程数量；
- 可以被中断；


##### ReentrantLock 有哪些缺点
-   需要使用 import 引入相关的 Class
-   不能忘记在 finally 模块释放锁,这个看起来比 synchronized 丑陋
-   synchronized 可以放在方法的定义里面, 而 reentrantlock 只能放在块里面. 比较起来, synchronized 可以减少嵌套


#### ExecutorService
是一个接口，约定了一些方法，用以管理多个异步任务的执行，可以获取执行任务的 future，用以跟踪任务的执行。

可以执行任务，关闭任务等。

基于此，有一个线程池实现类 - ThreadPoolExecutor。


##### 为什么需要线程池
1. 线程是很重的资源，JVM 的线程和系统的线程是一一对应的，创建和销毁都是系统调用，是很消耗资源的；
2. 线程不少越多越好，线程是 CPU 调度的最小单位,但 CPU 的核心数有限，同时能运行的线程数有限，所以需要根据调度算法切换执行的线程，而线程的切换需要开销，比如替换寄存器的内容、高速缓存的失效等等。如果线程数太多，切换的频率就变高，可能使得多线程带来的好处抵不过线程切换带来的开销，得不偿失。

小结一下：
-   Java中线程与操作系统线程是一比一的关系。
-   线程的创建和销毁是一个“较重”的操作。
-   多线程的主要是为了提高 CPU 的利用率。
-   线程的切换有开销，线程数的多少需要结合 CPU核心数与 I/O 等待占比。IO 占比高的，可以适当提高线程数，以充分利用 CPU 资源；


所以
1. 控制线程的数量；
2. 缓存一批线程，避免重复的创建和销毁；


##### 线程池
- [可以参考的资源1](https://mp.weixin.qq.com/s/NDOx94yY06OnHjrYq2lVYw)

线程池就是事先将多个线程对象放到一个容器中，使用的时候就不用new线程而是直接去池中拿线程即可，节省了开辟子线程的时间，提高了代码执行效率。


熟悉对象池、连接池的朋友肯定对池化技术不陌生，一般池化技术的使用方式是从池子里拿出资源，然后使用，用完了之后归还。但是线程池的实现不太一样，不是说我们从线程池里面拿一个线程来执行任务，等任务执行完了之后再归还线程，你可以想一下这样做是否合理。线程池的常见实现更像是一个黑盒存在，我们设置好线程池的大小之后，直接往线程池里面丢任务，然后就不管了。


线程池其实是一个典型的生产者-消费者模式。


##### 自己设计一个线程池
1. 一个队列，用以存放 task 任务；
2. 一个线程集合，用以执行任务；
3. 定义拒绝策略；

线程池讲白了就是存储线程的一个容器，池内保存之前建立过的线程来重复执行任务，减少创建和销毁线程的开销，提高任务的响应速度，并便于线程的管理。
我个人觉得如果要设计一个线程池的话得考虑池内工作线程的管理、任务编排执行、线程池超负荷处理方案、监控。

初始化线程数、核心线程数、最大线程池都暴露出来可配置，包括超过核心线程数的线程空闲消亡配置。

任务的存储结构可配置，可以是无界队列也可以是有界队列，也可以根据配置分多个队列来分配不同优先级的任务，也可以采用 stealing 的机制来提高线程的利用率。

##### ThreadPoolExecutor
是个线程池实现类。
![线程池示意图](https://mmbiz.qpic.cn/mmbiz_png/eSdk75TK4nFApmmGxuSmkFnGZIczU6HYDguQXRlrglaIKgUCZ5ZCGj64s4S0iaFzIW990yfJiaplaC1z6aYDtoEA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

简单来说线程池把任务的提交和任务的执行剥离开来，当一个任务被提交到线程池之后：
-   如果此时线程数小于核心线程数，那么就会新起一个线程来执行当前的任务。
-   如果此时线程数大于核心线程数，那么就会将任务塞入阻塞队列中，等待被执行。
-   如果阻塞队列满了，并且此时线程数小于最大线程数，那么会创建新线程来执行当前任务。
-   如果阻塞队列满了，并且此时线程数大于最大线程数，那么会采取拒绝策略。


###### 任务是先入队，还是先执行？
看两个条件
1. core 线程数；
2. 任务队列是否已满；

####### 线程数达到核心数的时候，如果任务队列未满
任务是先入队，而不是先创建最大线程数。

####### 线程数达到核心数的时候，如果任务队列满了
如果任务队列也满了，新增最大线程数的线程时，任务是可以直接给予新建的线程执行的，而不是入队。

####### 什么时候创建非核心线程
- 核心线程忙碌状态；
- 等待队列已满；

####### 此时线程数小于核心线程数，并且线程都处于空闲状态，现提交一个任务，是新起一个线程还是给之前创建的线程？
```text
When a new task is submitted in method `[execute(java.lang.Runnable)](https://developer.android.com/reference/java/util/concurrent/ThreadPoolExecutor#execute(java.lang.Runnable))`, if fewer than corePoolSize threads are running, a new thread is created to handle the request, even if other worker threads are idle.
```

[参考链接](https://developer.android.com/reference/java/util/concurrent/ThreadPoolExecutor)

答案 ：新创建一个线程执行任务。



###### 原生线程池的核心线程一定伴随着任务慢慢创建的吗？
并不是，线程池提供了两个方法：
-   prestartCoreThread：启动一个核心线程
-   prestartAllCoreThreads ：启动所有核心线程
不要小看这个预创建方法，预热很重要，不然刚重启的一些服务有时是顶不住瞬时请求的，就立马崩了，所以有预热线程、缓存等等操作。

###### 你是如何理解核心线程的 ?
核心线程指的是线程池承载日常任务的中坚力量，也就是说本质上线程池是需要这么些数量的线程来处理任务的，所以在懒中又急着创建它。
而最大线程数其实是为了应付突发状况。

####### 线程池的核心线程在空闲的时候一定不会被回收吗？
有个 allowCoreThreadTimeOut 方法，把它设置为 true ，则所有线程都会超时，不会有核心数那条线的存在。


###### 你是怎么理解 KeepAliveTime 的？
这就是上面提到的，线程池其实想要的只是核心线程数个线程，但是又预留了一些数量来预防突发状况，当突发状况过去之后，线程池希望只维持核心线程数的线程，所以就弄了个 KeepAliveTime，当线程数大于核心数之后，如果线程空闲了一段时间（KeepAliveTime），就回收线程，直到数量与核心数持平。

线程池希望维持的线程数量就是 core 数量的线程，可能存在突发情况，导致线程的数量达到 max，但是突发事件过后，还是希望恢复到常态 core 的数量。


###### 那 workQueue 有什么用？
缓存任务供线程获取。所以工作队列起到一个缓冲作用，具体队列长度需要结合线程数，任务的执行时长，能承受的等待时间等。


###### 你是如何理解拒绝策略的？

- 线程数达到最大值；
- 队列中的任务数达到最大值；

满足上述两个条件后，
- 默认实现是 AbortPolicy 直接抛出异常。
- 直接丢弃任务；
- 让提交任务的线程自己运行；
- 淘汰老的未执行的任务而空出位置；

具体用哪个策略，根据场景选择。当然也可以自定义拒绝策略，实现 `RejectedExecutionHandler` 这个接口即可。


###### 线程池里的 ctl 的作用？
工作线程数和线程池状态结合在一起维护，低 29 位存放 workerCount，高 3 位存放 runState。


其实并发包中有很多实现都是一个字段存多个值的，比如读写锁的高 16 位存放读锁，低 16 位存放写锁，这种一个字段存放多个值可以更容易的维护多个值之间的一致性，也算是极简主义。


###### 线程池有几种状态吗？
-   RUNNING：能接受新任务，并处理阻塞队列中的任务
-   SHUTDOWN：不接受新任务，但是可以处理阻塞队列中的任务
-   STOP：不接受新任务，并且不处理阻塞队列中的任务，并且还打断正在运行任务的线程，就是直接撂担子不干了！
-   TIDYING：所有任务都终止，并且工作线程也为0，处于关闭之前的状态
-   TERMINATED：已关闭。

![状态变迁图](https://mmbiz.qpic.cn/mmbiz_png/eSdk75TK4nFApmmGxuSmkFnGZIczU6HYs2XzibU6JNtBALibRhZTFWPVicaQ5IYst97Zh1mOIlSdJLbanMEIX8N4w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1****)


###### 为什么要先进队列，而不是直接新建线程执行
其实经过上面的分析可以得知，线程池本意只是让核心数量的线程工作着，不论是 core 的取名，还是 keepalive 的设定，所以你可以直接把 core 的数量设为你想要线程池工作的线程数，而任务队列起到一个缓冲的作用。最大线程数这个参数更像是无奈之举，在最坏的情况下做最后的努力，去新建线程去帮助消化任务。

所以我个人觉得没有为什么，就是这样设计的，并且这样的设定挺合理。

原生版线程池的实现可以认为是偏向 CPU 密集的，也就是当任务过多的时候不是先去创建更多的线程，而是先缓存任务，让核心线程去消化，从上面的分析我们可以知道，当处理 CPU 密集型任务的时，线程太多反而会由于线程频繁切换的开销而得不偿失，所以优先堆积任务而不是创建新的线程。

###### 如何修改原生线程池，使得可以先拉满线程数再入任务队列排队？
如果了解线程池的原理，很轻松的就知道关键点在哪，就是队列的 offer 方法。
execute 方法想必大家都不陌生，就是给线程池提交任务的方法。在这个方法中可以看到只要在 offer 方法内部判断此时线程数还小于最大线程数的时候返回 false，即可走下面 `else if` 中 `addWorker` (新增线程)的逻辑，如果数量已经达到最大线程数，直接入队即可。

自定义队列，让队列在线程数还未到达最大值时，不允许入队，就会进入创建新线程的逻辑。

###### 如果线程池中的线程在执行任务的时候，抛异常了，会怎么样？

把这个线程废了，然后新建一个线程替换之。

所以如果一个任务执行一半就抛出异常，并且你没有自行处理这个异常，那么这个任务就这样戛然而止了，后面也不会有线程继续执行剩下的逻辑，所以要自行捕获和处理业务异常。

- shutdown：一个是安全的关闭线程池，会等待任务都执行完毕；
- shutdownNow：粗暴的直接咔嚓了所有线程，管你在不在运行；
- shutdownNow 了之后还在任务队列中的任务咋办？线程池还算负责，把未执行的任务拖拽到了一个列表中然后返回，至于怎么处理，就交给调用者了！

###### 线程池如何动态修改核心线程数和最大线程数？
-   CPU 密集型的话，核心线程数设置为 CPU核数+1
-   I/O 密集型的话，核心线程数设置为 2*CPU核数
线程数= CPU核数  ✖️ 1 + 线程等待时间 / 线程时间运行时间）
所以说线程数真的很难通过一个公式一劳永逸，线程数的设定是一个迭代的过程，需要压测适时调整，以上的公式做个初始值开始调试是 ok 的。

但是可以根据监控的状态，动态修改线程池的配置。



### 为什么不推荐使用jdk自带的 executors 的方式来创建线程池?
避免资源耗尽的风险；
如果用这种方式设置，则
- FixedThreadPool和SingleThreadPool允许的请求队列长度为Integer.MAX_VALUE，可能会堆积大量的请求，从而导致OOM
- CachedThreadPool和ScheduledThreadPool允许创建的线程数量为Integer.MAX_VALUE，可能会创建大量的线程，从而导致OOM

### executors 的几种线程池
###### newCachedThreadPool
不固定线程数量，且支持最大为Integer.MAX_VALUE的线程数量:
1、线程数无限制。 
2、有空闲线程则复用空闲线程，若无空闲线程则新建线程
3、一定程序减少频繁创建/销毁线程，减少系统开销。

###### newFixedThreadPool
一个固定线程数量的线程池:
1、可控制线程最大并发数（同时执行的线程数）。 
2、超出的线程会在队列中等待。

###### newSingleThreadExecutor
可以理解为线程数量为1的FixedThreadPool:
单线程化的线程池：
1、有且仅有一个工作线程执行任务。
2、所有任务按照指定顺序执行，即遵循队列的入队出队规则。

###### newScheduledThreadPool
支持定时以指定周期循环执行任务: 注意：前三种线程池是ThreadPoolExecutor不同配置的实例，最后一种是ScheduledThreadPoolExecutor的实例。


## AQS
> 提供更多的锁机制
> 实现阻塞锁和与之相关依赖于先进先出队列等待队列的同步器；

[参考的文章1](https://mp.weixin.qq.com/s/iNz6sTen2CSOdLE0j7qu9A)
[参考的文章2](https://mp.weixin.qq.com/s/hB5ncpe7_tVovQj1sNlDRA)
[参考文章3](https://mp.weixin.qq.com/s/trsjgUFRrz40Simq2VKxTA)
[参考的原理讲解](https://www.bilibili.com/video/BV1V84y1H7oF/?spm_id_from=333.788&vd_source=2383886846e4aa5e7cf6fd52f9d0a367)


### 基于许可的多线程控制
为了控制多个线程访问共享资源 ，我们需要为每个访问共享区间的线程派发一个许可。拿到一个许可的线程才能进入共享区间活动。当线程完成工作后，离开共享区间时，必须要归还许可，以确保后续的线程可以正常取得许可。如果许可用完了，那么线程进入共享区间时，就必须等待，这就是控制多线程并行的基本思想。

### 管程
基于许可的控制，有点和管程很像。

### 排他锁和共享锁
第二个重要的概念就是排他锁(exclusive)和共享锁(shared)。顾名思义，在排他模式上，只有一个线程可以访问共享变量，而共享模式则允许多个线程同时访问。简单地说，重入锁是排他的；信号量是共享的。
共享也是一定量的共享。

### LockSupport
-   public static void park() : 如果没有可用许可，则挂起当前线程
-   public static void unpark(Thread thread)：给thread一个可用的许可，让它得以继续执行

```java
LockSupport.unpark(Thread.currentThread());
LockSupport.park();
```

大家可以猜一下，park()之后，当前线程是停止，还是 可以继续执行呢？

答案是：可以继续执行。那是因为在park()之前，先执行了unpark()，进而释放了一个许可，也就是说当前线程有一个可用的许可。而park()在有可用许可的情况下，是不会阻塞线程的。

综上所述，park()和unpark()的执行效果和它调用的先后顺序没有关系。这一点相当重要，因为在一个多线程的环境中，我们往往很难保证函数调用的先后顺序(都在不同的线程中并发执行)，因此，这种基于许可的做法能够最大限度保证程序不出错。

与park()和unpark()相比， 一个典型的反面教材就是Thread.resume()和Thread.suspend()。

### Thread.resume()和Thread.suspend()
```java
Thread.currentThread().resume();
Thread.currentThread().suspend();
```

首先让线程继续执行，接着在挂起线程。这个写法和上面的park()的示例非常接近，但是运行结果却是截然不同的。在这里，当前线程就是卡死。

因此，使用park()和unpark()才是我们的首选。而在AbstractQueuedSynchronizer中，也正是使用了 LockSupport 的park()和unpark()操作来控制线程的运行状态的。

在AbstractQueuedSynchronizer内部，
- 有一个队列，我们把它叫做**同步等待队列**。它的作用是保存等待在这个锁上的线程(由于lock()操作引起的等待）。
- 为了维护等待在条件变量上的等待线程，AbstractQueuedSynchronizer又需要再维护一个**条件变量等待队列**，也就是那些由Condition.await()引起阻塞的线程。
![数据结构](https://mmbiz.qpic.cn/mmbiz_jpg/uChmeeX1Fpx7JSVwOERcCuTUA4ZfuvczicXgb2JJQvlLzklMOhd3NOJ5KVsm3xprYVovO2LvU6fxL0iaUpSk6PicA/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)



##  Java 对象真实占用的内存大小

- [可以参考的资源1](https://www.cnblogs.com/rickiyang/p/14206724.html)

![java对象数据结构](https://img2020.cnblogs.com/blog/1607781/202012/1607781-20201229150956450-1399589204.png)


-   对象头（Header）
	- MarkWord：用于存储对象运行时的数据，好比 HashCode、锁状态标志、GC分代年龄等。这部分在 64 位操作系统下占 8 字节，32 位操作系统下占 4 字节。
	- 指针：对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪一个类的实例。这部分就涉及到指针压缩的概念，在开启指针压缩的状况下占 4 字节，未开启状况下占 8 字节。
	- 数组长度：这部分只有是数组对象才有，若是是非数组对象就没这部分。这部分占 4 字节。
-   实例数据（Instance Data）用于存储对象中的各类类型的字段信息（包括从父类继承来的）
-   对齐填充（Padding），Java 对象的大小默认是按照 8 字节对齐
	- 由于 CPU 进行内存访问时，一次寻址的指针大小是 8 字节，正好也是 L1 缓存行的大小。如果不进行内存对齐，则可能出现跨缓存行的情况，这叫做 缓存行污染。
	- JVM 为对象进行填充，使其大小变为 8 个字节的倍数。使用这些填充后，oops 中的最后三位始终为零。这是因为在二进制中 8 的倍数的数字总是以 000 结尾。由于 JVM 已经知道最后三位始终为零，因此在堆中存储那些零是没有意义的。相反假设它们存在并存储 3 个其他更重要的位，以此来模拟 35 位的内存地址。现在我们有一个带有 3 个右移零的 32 位地址，所以我们将 35 位指针压缩成 32 位指针。这意味着我们可以在不使用 64 位引用的情况下使用最多 32 GB ：  (2(32+3)=235=32 GB) 的堆空间。
	- 当 JVM 需要在内存中找到一个对象时，它将指针向左移动 3 位。另一方面当堆加载指针时，JVM 将指针向右移动 3 位以丢弃先前添加的零。虽然这个操作需要 JVM 执行更多的计算以节省一些空间，不过对于大多数CPU来说，位移是一个非常简单的操作。


### 压缩的原理

CompressedOops 工作原理

32 位内最多可以表示 4GB，64 位地址为 堆的基地址 + 偏移量，当堆内存 < 32GB 时候，在压缩过程中，把 偏移量 / 8 后保存到 32 位地址。在解压再把 32 位地址放大 8 倍，所以启用 CompressedOops 的条件是堆内存要在 4GB * 8=32GB 以内。

JVM 的实现方式是，不再保存所有引用，而是每隔 8 个字节保存一个引用。例如，原来保存每个引用 0、1、2...，现在只保存 0、8、16...。因此，指针压缩后，并不是所有引用都保存在堆中，而是以 8 个字节为间隔保存引用。

在实现上，堆中的引用其实还是按照 0x0、0x1、0x2... 进行存储。只不过当引用被存入 64 位的寄存器时，JVM 将其左移 3 位（相当于末尾添加 3 个0），例如 0x0、0x1、0x2... 分别被转换为 0x0、0x8、0x10。而当从寄存器读出时，JVM 又可以右移 3 位，丢弃末尾的 0。（oop 在堆中是 32 位，在寄存器中是 35 位，2的 35 次方 = 32G。也就是说使用 32 位，来达到 35 位 oop 所能引用的堆内存空间）。

#### 为什么可以压缩？
。。。。。。。 
// todo

### 数据类型占用的空间
#### 基础数据类型占用的内存大小
- 数据类型对应的内存占用，参考：https://www.baeldung.com/java-primitives

#### 引用类型占用的内存大小
引用类型在 32 位系统上每个引用对象占用 4 byte，在 64 位系统上每个引用对象占用 8 byte。


### Java 对象到底占用多大内存
- [可供参考的资料1](https://hg.openjdk.org/jdk8u/jdk8u/hotspot/file/6f33e450999c/src/share/vm/oops/markOop.hpp)
- [JOL](https://github.com/openjdk/jol)


**首先记住公式，对象由 对象头 + 实例数据 + padding 填充字节组成，虚拟机规范要求对象所占内存必须是 8 的倍数，padding 就是干这个的**。

对象头由 Markword + 类指针kclass（该指针指向该类型在方法区的元类型） 组成。


可以借助这个工具进行分析 `org.openjdk.jol:jol-core:0.17`


new String("xxxx"); 这个对象是 24 bytes；

无论是什么对象，一般 header 是 12，剩下的是具体的属性对象的大小，外加对齐的内容。

