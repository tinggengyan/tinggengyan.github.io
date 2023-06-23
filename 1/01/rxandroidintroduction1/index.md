# 

title: RxAndroid入门分享(一)
date: 2016-02-08 21:44:26
tags: [RxJava,Android,RxAndroid]
categories: [Mobile,Android]
---
# 概述
本文记录 RxJava 的概念.

<!-- more -->

# RxJava 以及响应式函数思想篇

## Why 技术产生的背景

在编程中，经常需要切换线程，为了能对结果进行相应处理，经常需要进行回调，随着业务需求的增加，嵌套的回调也会随之增加，不仅增加了代码量，也增加了逻辑的复杂性，增加了理解和维护的难度。
所以就需要一种
- 既能方便切换线程
- 又能即是相应变化
- 又可以简化代码的逻辑，方便维护，
- 还不需要回调。

## What ReactiveX是什么

Reactive Extensions,简称 RX，原来只是微软开发的一个 LINQ 的一个扩展。

微软给的定义是，Rx 是一个函数库，让开发者可以利用**可观察序列**和**LINQ风格查询操作符**来编写**异步**和基于**事件**的程序，使用 Rx，开发者可以用Observables 表示异步数据流，用 LINQ 操作符查询异步数据流，用 Schedulers 参数化异步数据流的并发处理，Rx 可以这样定义：**Rx = Observables + LINQ + Schedulers。**

ReactiveX.io 给的定义是，Rx 是一个使用可观察数据流进行异步编程的**编程接口**，ReactiveX 结合了观察者模式、迭代器模式和函数式编程的精华。

看完微软给的定义已经很详细了，开源组织给的更加精简，里面提到了数据流还有事件，我们来自己看看怎么理解。

这里得提到响应式编程的概念，其中有两个关键点，
- 事件，事件可以被观察，等待，过滤，响应，也可以触发其他的事件，事件通过数据流的形式对外呈现。
- 数据流，数据流就像一条河：它可以**被观测，被过滤，被操作，或者与另外一条流合并为一条新的流来给新的消费者消费**。

所以，响应式编程就是一种基于异步**数据流**概念的编程模式。其实 EventBus 还有其他的点击事件一样，本质上就是异步的数据流，我们可以为任何的事件创建数据流。比如我们可以为登录操作创建数据流，然后监听这个数据流，进行登录验证这样的响应操作。

主要特点有：
- 易于并发从而更好的利用服务器的能力。
- 易于有条件的异步执行。
- 一种更好的方式来避免回调地狱。
- 一种响应式方法。

## RxJava与传统的Java的不同

在 Rx 中，开发者用 Observables 模拟可被观察的异步数据流，从纯 Java 的观点看，RxJava 的 Observable 类源自于经典的 Gang Of Four 的观察者模式。


### 与传统观察者的不同

它添加了三个缺少的功能：
- 生产者在没有更多数据可用时能够发出信号通知：onCompleted()事件。
- 生产者在发生错误时能够发出信号通知：onError()事件。
- RxJava Observables 能够组合而不是嵌套，从而避免开发者陷入回调地狱。


### 与传统的Iterable的不同

Observables 和 IterablesAPI 是很相似的：我们在 Iterable 可以执行的许多操作也都同样可以在 Observables 上执行。当然，由于 Observables 流的本质，没有如Iterable.remove() 这样相应的方法,因为数据可能已经发射出去了，remove 也没有任何意义。

使用 Iterable 时，消费者从生产者那里以同步的方式得到值，在这些值得到之前线程处于阻塞状态。相反，使用 Observable 时，生产者以异步的方式把值 push 给观察者，无论何时，这些值都是可用的。这种方法之所以更灵活是因为即便值是同步或异步方式到达，消费者在这两种场景都可以根据自己的需要来处理。


| Pattern        | 一个返回值   |  多个返回值  |
| --------       | -----:  | :----:  |
| Synchronous    | T getData()| Iterable<T>  |
| Asynchronous   | Future<T> getData()  |  Observable<T> getData()   |

Observable 的生命周期包含了三种可能的易于与 Iterable 生命周期事件相比较的事件，下表展示了如何将 Observable async/push 与 Iterable sync/pull 相关联起来。

| Event       |  Iterable(pull)   |   Observable(push)  |
| --------    | -----:  | :----:  |
| 检索数据    |  T next()          |   onNext(T)         |
| 发现错误    |  throws Exception  |   onError(Throwable)|
| 完成        | !hasNext()        |    onCompleted()     |



所以，由以上这些新增的特点，开发者只要简单的去请求，当请求完成的时候，会得到一个通知。开发者需要对可能发生的每个事件提供一个清晰的响应链。

### 举个例子：

用户提交完用户名和密码，我们可以用 observable 模拟这个登录的数据流，而后我们需要对可能发生的情况进行定义；
- 用户名密码正确，登录成功，转到登录成功界面。
- 用户名和密码匹配不成功，登录失败，给用户个提示。

这样，我们不需要等待结果，等到有结果的时候，会有通知，这个过程是异步的。这中间可以做很多其他的事情，保存到缓存，显示进度条等等。


## 概念

### Observable

当我们异步执行一些复杂的事情，Java 提供了传统的类，例如 Thread、Future、FutureTask、CompletableFuture 来处理这些问题。当复杂度提升，这些方案就会变得麻烦和难以维护。最糟糕的是，它们都不支持链式调用。RxJava Observables 可以解决这些问题。它可以作用于单个结果程序上，也可以作用于序列上。无论何时你想发射单个标量值，或者一连串值，甚至是无穷个数值流，你都可以使用 Observable。和传统的观察者模式一样，也有冷热之分。
- 热的 observable，只要创建了 observable，就开始发射数据了，所以，后续订阅他的 observer 可能从中间某个位置开始接收数据。
- 冷的 observable，等到有订阅的时候才开始发射数据。

### Observer

观察者，订阅 observable 发射的数据，对其做出相应，对可能出现的情况的定义就在这里。

三个重要的回调方法 (onNext, onCompleted, onError)
通过Subscribe方法可以将观察者连接到 Observable，观察者需要实现以下方法的一个子集:
- onNext(T item):Observable 调用这个方法发射数据，方法的参数就是 Observable 发射的数据，这个方法可能会被调用多次，取决于你的实现。
- onError(Exception ex):当 Observable 遇到错误或者无法返回期望的数据时会调用这个方法，这个调用会终止 Observable，后续不会再调用 onNext 和 onCompleted，onError 方法的参数是抛出的异常。
- onComplete:正常终止，如果没有遇到错误，Observable 在最后一次调用 onNext 之后调用此方法。

根据 Observable 协议的定义，onNext 可能会被调用**零次或者很多次**，最后会有一次 onCompleted 或 onError 调用（不会同时），传递数据给 onNext 通常被称作发射，onCompleted 和 onError 被称作通知。

### Subscriber

Observers 和 Subscribers 是两个“消费”实体。Subscriber 是一个实现了 Observer 的一个抽象类。相对于基本的 Observer，提供了手动解开订阅的方法 unsubscribe 和在 subscribe 刚开始，而事件还未发送之前被调用的方法 onStart。其他的使用方式是一样的。

### Subjects

Subject = Observable + Observer。
subject 是一个神奇的对象，它可以是一个 Observable 同时也可以是一个 Observer：它作为连接这两个世界的一座桥梁。一个 Subject 可以订阅一个 Observable，就像一个观察者，并且它可以发射新的数据，或者传递它接受到的数据，就像一个 Observable。很明显，作为一个 Observable，观察者们或者其它 Subject 都可以订阅它。
一旦 Subject 订阅了 Observable，它将会触发 Observable 开始发射。如果原始的 Observable 是“冷”的，这将会对订阅一个“热”的 Observable 变量产生影响。

## How怎么使用

接下来讨论他的具体使用方法。首先是需要搭建环境，我们就以 AS 为例。
- 因为就是为了 Android 开发所学的，在 module 的 gradle 中添加 RxAndroid 的仓库地址,RxAndroid 本身是依赖 RxJava 的，所以会自动下载 RxJava 的依赖包。
```groovy
compile 'io.reactivex:rxandroid:1.1.0'
```

### 创建Observable

#### Create之从头创建

这个操作符传递一个**含有观察者作为参数的函数**的对象，编写这个函数让它的行为表现为一个 Observable --恰当的调用观察者的 onNext，onError 和 onCompleted 方法。下面是个非常简单的一个例子，先有个直观的大致的认识。

```Java
Observable.OnSubscribe<String> f = new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> o) {
                //完全自己决定发射数据给subscriber和通知subscriber的时机以及方式
                o.onNext("发射的数据");
                o.onCompleted();
            }
        };
        Observable observable = Observable.create(f);
        //创建订阅者
        Subscriber subscriber = new Subscriber() {
            @Override
            public void onCompleted() {
                //正常结束，收到发射的通知
            }

            @Override
            public void onError(Throwable e) {
                //出现了错误的通知
            }

            @Override
            public void onNext(Object o) {
                //收到observable发射的数据
            }
        };
        //订阅，一旦订阅发生，observable将开始发射数据
        observable.subscribe(subscriber);
```

#### From

这个操作符需要传入数组或者列表等可以迭代的类型，将会返回一个 Observable 对象，这个 Observable 会迭代列表里的数据，然后将数据一个一个的发射出去。

#### Just

这个操作符会返回一个 Observable，这个 Observable 将传入的对象直接发射出去。这个操作符对于进行旧版本的改造非常有用，对于暂时不想做过多操作的函数，可以直接传入到 just 操作符中，这样就自动构造出了一个数据流。

#### Repeat

这个操作符需要一个整形数字作为参数，代表了重复发射的次数，比如发射“123”三次，就会变成发射"123123123"。

#### defer

这个操作符可以延迟 Observable 的创建，当有订阅者的时候才开始创建，这个对于一些不是每次都需要创建的数据流而言，很有用。怎么理解呢，我们简单的看个例子。

```Java
public class MainActivity extends AppCompatActivity {

    private Subscriber subscriber;
    private Observable simpleObservable;

    private String doSomeThing() {
        System.out.println("在执行just的时候，这里需要执行的操作已经执行结束了。。。");
        return "SteveYan";
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        simpleObservable = Observable.just(doSomeThing());
        subscriber = new Subscriber() {
            @Override
            public void onCompleted() {
                //正常结束，收到发射的通知
                System.out.println("onCompleted");
            }

            @Override
            public void onError(Throwable e) {
                //出现了错误的通知
            }

            @Override
            public void onNext(Object o) {
                //收到observable发射的数据
                System.out.println("Receive " + o.toString());
            }
        };
        init();
    }

    private void init() {
        findViewById(R.id.button).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //每次点击的时候都进行订阅
                simpleObservable.subscribe(subscriber);
            }
        });
    }

}
```
在上面的例子中，每次点击就进行一次订阅，在 onCreate 方法里，在执行 just 的时候，doSomeThing 已经执行完了，
但是并未发射数据，但是假如使用 defer 操作符的话，doSomeThing 则会等到点击的时候才执行。
修改成的 defer 操作符的
```Java
public class MainActivity extends AppCompatActivity {

    private Subscriber subscriber;
    private Observable simpleObservable;

    private String doSomeThing() {
        System.out.println("Do Some");
        return "SteveYan";
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        simpleObservable = Observable.defer(new Func0<Observable<String>>() {
            @Override
            public Observable<String> call() {
                return Observable.just(doSomeThing());
            }
        });
        subscriber = new Subscriber() {
            @Override
            public void onCompleted() {
                //正常结束，收到发射的通知
                System.out.println("onCompleted");
            }

            @Override
            public void onError(Throwable e) {
                //出现了错误的通知
            }

            @Override
            public void onNext(Object o) {
                //收到observable发射的数据
                System.out.println("Receive " + o.toString());
            }
        };
        init();
    }

    private void init() {
        findViewById(R.id.button).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //每次点击的时候都进行订阅
                simpleObservable.subscribe(subscriber);
            }
        });
    }

}
```

#### range 从一个指定的数字X开始发射N个数字

range() 函数用两个数字作为参数：第一个是起始点，第二个是我们想发射数字的个数。目前未发现在实际项目中的用处。

#### interval  重复轮训操作

interval() 函数的两个参数：一个指定两次发射的时间间隔，另一个是用到的时间单位。需要创建一个轮询程序时非常好用

#### timer 一段时间之后才发射的Observable

接受两个参数，一个是延迟发射的时间，第二个参数是时间的。

### 可观测序列的本质：过滤

过滤：如何从发射的 Observable 中选取我们想要的值，如何获取有限个数的值，如何处理溢出的场景，以及更多的有用的技巧。

#### filter函数，进行内容的过滤

接受一个参数，对数据流中的每个数据进行过滤。

#### Take,取序列的前N个元素

take() 函数用整数N来作为一个参数，从原始的序列中发射前 N 个元素

#### takeLast,取序列的最后的N个元素

如果我们想要最后 N 个元素，接给 takeLast
函数传入 N 作为参数。有一点值得注意，为了得到最后的数据，所以 takeLast 方法只能作用于一组有限的序列（发射元素），它只能应用于一个完整的序列。否则他无从知晓最后到哪。

#### Distinct 有且仅有一个

distinct 函数去掉重复的。就像 takeLast 一样，distinct 也必须作用于一个完整的序列，然后得到重复的过滤项，它需要记录每一个发射的值。如果你在处理一大堆序列或者大的数据记得关注内存使用情况。

#### DistinctUntilsChanged 改变的时候就记录

如果我们想在一个可观测序列发射一个不同于之前的一个新值时，让我们得到通知，就可以用这个操作符。

#### First And Last

从 Observable 中只发射第一个元素或者最后一个元素。这两个都可以传 Func1 作为参数，：一个可以确定我们感兴趣的第一个或者最后一个的谓词。
与 first()和 last()相似的变量有：firstOrDefault() 和 lastOrDefault().这两个函数当可观测序列完成时不再发射任何值时用得上。在这种场景下，如果 Observable 不再发射任何值时我们可以指定发射一个默认的值

#### Skip And SkipLast

它们用整数 N 作参数，从本质上来说，它们不让 Observable 发射前 N 个或者后 N 个值。这个和上面的 First 和 Last 正好相反。

#### elementAt 观察指定位置的数据

elementAt() 函数仅从一个序列中发射第 n 个元素然后就完成了。
如果我们想查找第五个元素但是可观测序列只有三个元素可供发射时该怎么办？我们可以使用 elementAtOrDefault()。

#### sample 每隔一段时间取最近的数据

创建一个新的可观测序列，它将在一个指定的时间间隔里由 Observable 发射最近一次的数值。
如果我们想让它定时发射第一个元素而不是最近的一个元素，我们可以使用 throttleFirst()。

#### timeout 超时操作

使用 timeout() 函数来监听源可观测序列,就是在我们设定的时间间隔内如果没有得到一个值则发射一个错误。

#### debounce 除去发射过快的数据

debounce() 函数过滤掉由 Observable 发射的速率过快的数据；如果在一个指定的时间间隔过去了仍旧没有发射一个，那么它将发射最后的那个。


### 转换Observables

#### Map 

转换发射的数据，将发射数据 A 的 Observable 变换成发射数据 B 的 Observable。适用于对数据的再加工场景。

#### FlatMap 铺平序列

这样的 Observable：它发射一个数据序列，这些数据本身也可以发射 Observable。等于是说发射的数据可以再发射数据。flatMap 函数提供一种铺平序列的方式，然后合并这些 Observables 发射的数据，最后将合并后的结果作为最终的 Observable.

当我们在处理可能有大量的 Observables 时，重要是记住任何一个 Observables 发生错误的情况，flatMap 将会触发它自己的 onError 函数并放弃整个链。

重要的一点提示是关于合并部分：它允许交叉。正如上图所示，这意味着 flatMap 不能够保证在最终生成的 Observable 中源 Observables 确切的发射顺序。

#### ConcatMap 保证有序的铺平

和上面的 FlatMap 一样，就是弥补了交叉这个一个特点。

#### FlatMapIterable

它将源数据两两结成对并生成 Iterable，而不是原始数据项和生成的 Observables。

#### SwitchMap 切换数据流(喜新厌旧)

switchMap() 和 flatMap() 很像，除了一点：每当源 Observable 发射一个新的数据项（Observable）时，它将取消订阅并停止监视之前那个数据项产生的 Observable，并开始监视当前发射的这一个。

#### Scan

RxJava 的 scan() 函数可以看做是一个累积函数。scan 函数对原始 Observable 发射的每一项数据都应用一个函数，计算出函数的结果值，并将该值填充回可观测序列，等待和下一次发射的数据一起使用。简单的说是每次可以处理的数据有本次的和上次的数据。

#### groupBy 分组

接受一个方法，在方法里进行分组操作，返回一个自定义的值，系统将根据这个值将进行分组。

#### buffer

每次发射一组值，是一个列表，而不是一个个的发射。接受一个整形参数 N，表示 N 个一组进行发射。也可以接受两个参数，SKIP，表示 SKIP 个值中取 N 个一组进行发射。

#### window

和 buffer 类似，但是他发射的不是一个列表，而是一个 Observable。

#### cast

和 map 类似，不同的是将数据进行转换成一个新的类型。TODO 目前测试未发现怎么使用。


### 组合Observable

以上的内容是对 Observable 的发射数据进行过滤，接下来谈谈怎么组合数据流。

#### merge

将多个 Observable 发射的值进行合并成一个新的 Observable 然后再发射。中途合并的时候出现任何一个错误都会导致链条断裂，如果想延迟这样的错误处理，可以用 mergeDelayError。

#### zip

这个操作符和 merge 一样，也是合并两个 Observable 的数据；不一样的是，他是将两个未进行打包的数据根据传入的谓词规则进行合并成一个新的数据,再进行发射。值得注意的是，两个数据流的长度必须一样，多余的数据将会因为不能打包而得不到发射。

#### join TODO 待详细验证用途

有四个参数，
- 第一个参数为，Observable，表示和源 Observable 结合的数据流。
- Func1参数：在指定的由时间窗口定义时间间隔内，源 Observable 发射的数据和从第二个 Observable 发射的数据相互配合返回的 Observable。
- Func1参数：在指定的由时间窗口定义时间间隔内，第二个 Observable 发射的数据和从源 Observable 发射的数据相互配合返回的 Observable。
- Func2参数：定义已发射的数据如何与新发射的数据项相结合。

#### combineLatest

和 Zip 一样，去组合两个数据流发射的数据，并且进行组合，合并成一个新的数据进行发射，不同的是，Zip 发射的是最近未进行打包的，而 combineLatest 走的是相反的路线，打包最近发射的数据，不管是否已经打包过了。这样的话，就会弥补 Zip 的长度限制，全部得到发射。

#### startWith

Observable 开始发射他们的数据之前， startWith() 通过传递一个参数来先发射一个数据序列。 表示发射之前先将传入的参数法发射出去。

### Schedulers 随时切换运行的线程

这里大致说一下有哪几种情况，因为在 Java 中的情况和 Android 稍有差异，并且必须结合实例才能明白这个好处。

#### RxJava提供的五种调度器

- .io()
这个调度器时用于 I/O 操作。它基于根据需要，增长或缩减来自适应的线程池。由于它专用于 I/O 操作，所以并不是 RxJava 的默认方法；正确的使用它是由开发者决定的。重点需要注意的是线程池是无限制的，大量的 I/O 调度操作将创建许多个线程并占用内存。
- .computation()
这个是计算工作默认的调度器，与I/O 操作无关。也是许多 RxJava 方法的默认调度器：buffer(),debounce(),delay(),interval(),sample(),skip()。所以可以将一些耗时的，但是与 IO 无关的一些操作。
- .immediate()
这个调度器允许你立即在当前线程执行你指定的工作。它是 timeout(),timeInterval(),以及 timestamp() 方法默认的调度器。
- .newThread()
这个调度器正如它所看起来的那样：它为指定任务启动一个新的线程。
- .trampoline()
当我们想在当前线程执行一个任务时，并不是立即，我们可以用
.trampoline 将它入队。这个调度器将会处理它的队列并且按序运行队列中每一个任务。它是 repeat()和 retry() 方法默认的调度器。

#### SubscribeOn and ObserveOn，指定线程，线程切换

subscribeOn() 方法来用于每个 Observable 对象。subscribeOn() 方法用 Scheduler 来作为参数并在这个 Scheduler 上执行 Observable 调用。
observeOn() 方法将会在指定的调度器上返回结果。observeOn() 方法用 Scheduler 来作为参数，在指定的线程上返回结果，观察者在返回结果的线程上消费这个结果。

### Retrofit 

Retrofit 完美的支持 Rx 编程，可以完美的结合。


##感激,非常感激，万分的感激！

感谢以下的文章以及其作者和翻译的开发者们,排名不分先后

* [RxJava Essentials 中文翻译版](http://rxjava.yuxingxin.com/)
* [ReactiveX文档中文翻译](https://mcxiaoke.gitbooks.io/rxdocs/content/)
* [给 Android 开发者的 RxJava 详解](http://gank.io/post/560e15be2dca930e00da1083)

