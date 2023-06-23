# 

title: RxAndroid入门分享(二)
date: 2016-02-26 22:50:26
tags: [RxJava,Android,RxAndroid]
categories: [Mobile,Android]
---
# 概述
本文记录 RxJava 中在 Android 中的应用,介绍 RxAndroid 的使用.

<!-- more -->

# More RxJava 及其在Android上的应用

## 开发环境
- 在 module 的 gradle 中添加 RxAndroid 的相关依赖，如果想体验 rx 在 Android 上的更方便的功能。可以添加 Jake 大神的兼容包 RxBinding。
```groovy
    //RXAndroid
    compile 'io.reactivex:rxandroid:1.1.0'
    //RxBinding
    compile 'com.jakewharton.rxbinding:rxbinding:0.3.0'
    compile 'com.jakewharton.rxbinding:rxbinding-support-v4:0.3.0'
    compile 'com.jakewharton.rxbinding:rxbinding-appcompat-v7:0.3.0'
    compile 'com.jakewharton.rxbinding:rxbinding-design:0.3.0'
    compile 'com.jakewharton.rxbinding:rxbinding-recyclerview-v7:0.3.0'
    //compile 'com.jakewharton.rxbinding:rxbinding-leanback-v17:0.3.0'
```

## 以下开始用一个demo来演示

目的地有三个 tab，每个 tab 内的详情用 RecyclerView 展示，下拉刷新用 SwipeRefreshLayout。
布局界面如下
```xml
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

    <android.support.v4.widget.SwipeRefreshLayout
            android:id="@+id/destinationSwipeRefreshLayout"
            android:layout_width="match_parent"
            android:layout_height="match_parent">

        <android.support.v7.widget.RecyclerView
                android:id="@+id/destinationRecyclerView"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:clipToPadding="false">

        </android.support.v7.widget.RecyclerView>

    </android.support.v4.widget.SwipeRefreshLayout>

</FrameLayout>
```
开始 RxAndroid 的编写，我们开始考虑在目的地页面需要有的步骤
- 网络请求数据(放在 IO 线程)；
- 填充网络请求返回的数据到页面(UI 线程展示)；
RxAndroid 是基于响应式的编程，我们考虑将以上的网络请求产生的结果作为一个事件，他产生的数据就可以定义为数据流了。

### Observable&&create
```java
//网络请求，从头开始，自定义创建一个数据流。自主决定数据流的发射时机。
    private Observable<DestinationDataModel> getDestinationDataObservable(final String url) {
        return Observable.create(new Observable.OnSubscribe<DestinationDataModel>() {
            @Override
            public void call(Subscriber<? super DestinationDataModel> subscriber) {
                Gson gson = new Gson();
                Request request = new Request.Builder().url(url).build();
                Response response = null;
                try {
                    response = client.newCall(request).execute();
                    DestinationDataModel destinationDataModel = gson.fromJson(response.body().string(), DestinationDataModel.class);
                    if (subscriber.isUnsubscribed()){//判断连接是否断开，避免无谓的操作
                        return;
                    }
                    subscriber.onNext(destinationDataModel);
                    if (!subscriber.isUnsubscribed()){
                        subscriber.onCompleted();
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        });
    }
```
这里我们定义了一个方法，这个方法返回一个 Observable，里面封装了数据流,其实不是很计较的情况下，我们先将 Observable 事件等价(自我的理解，未在官方文档找到合适的解释，如有找到望不吝赐教)，事件对外以数据流的形式展现，这个事件就是在发射数据，然后发射结束就通知，失败了也通知。此处我们用到了操作符 create，一种创建操作符，这种操作的扩展性极大，但是我们也需要自己负责发射数据。全程的执行规范都在你的手里。
这里可能会有误区，数据流，不一定是多个，非要像 list，数组似的，那种有多个值，像弹珠似的连续发射。像上面的代码，我们只是发射了一个 destinationDataModel 对象，这个也是数据流，即，你发射出去的就是数据流的子集。


现在，我们有数据流了，就需要在某个适当的地方进行响应。我们思考，需要响应什么？对一个事件可能存在三种情况，事件对应的数据发射失败了( onError )，发射成功了( onCompleted ),还有接收到了发射来的数据( onNext )。
有点需要注意，这里的失败，以及成功，这两个回调，只是一个通知而已。


```java
      Subscriber<DestinationDataModel> subscriber = new Subscriber<DestinationDataModel>() {
      @Override
      public void onStart() {
        super.onStart();
      }

      @Override
      public void onCompleted() {
        Toast.makeText(getActivity(), "onCompleted", Toast.LENGTH_SHORT).show();
        destinationSwipeRefreshLayout.setRefreshing(false);
      }

      @Override
      public void onError(Throwable e) {
        e.printStackTrace();
        destinationSwipeRefreshLayout.setRefreshing(false);
      }

      @Override
      public void onNext(DestinationDataModel destinationDataModel) {
        destinationRecyclerView.setAdapter(new DestinationRecyclerAdapter(getActivity(), destinationDataModel.getDatas()));
      }
    };
```
我们定义了一个 Subscriber 对象，在上篇中讲到了什么是 Subscriber。这个是对数据流发射的相应，差不多对应了观察者模式中的观察者。当得到失败和成功的通知的时候，我们这里进行 log 的输出并且显示刷新的图标。当接收到数据的时候，我们就创建 recyclerview 的 adapter，进行列表的填充显示。


### subscribe
一旦我们订阅，就会执行数据的发射，默认的情况下，没有订阅操作，数据是不会被发射的。
```java
    getDestinationDataObservable(url).subscribe(subscriber);
```
以上的操作就会完成订阅，正常的数据产生，发射，相应都会发生。但是，真当我们允许的时候，就会报错，原因是我们都知道，对 UI 的操作都必须在 UI 主线程中。

### subscribeOn && observeOn

指定数据产生发射的线程和订阅响应的线程。
 ```java
 getDestinationDataObservable(url).subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread()).subscribe(subscriber);
 ```
 subscribeOn 表示 Observable 执行所在的线程，这里指的是网络请求，请求的数据进行 gson 解析，再将数据发射的线程。
 observeOn 表示的订阅响应的线程，这里指的是填充发射过来的数据到列表中。
 运行就能看到我们想要的效果。到此我们学会了最基本的一些使用，基本上掌握了这几个就可以轻松地展开工作了。当然还有很多其他的技巧。

### From && Just
我们刚刚有说过，数据流不一定是连续的，那么肯定存在连续的，连续不断的弹射，更符合官方文档那种弹珠示意图。from 就是一个这样的操作符。
这个目前未想到在当前这个模块的应用场景。


### repeat
这个是重复，我们让当前的列表中的数据重复发送两次。
```java
getDestinationDataObservableByCreate(url).repeat(2)
            .subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread())
            .subscribe(subscriber);
```
修改下subscriber
```java
@Override
            public void onNext(DestinationDataModel destinationDataModel) {
                if (destinationRecyclerView.getAdapter() == null) {
                    DestinationRecyclerAdapter adapter = new DestinationRecyclerAdapter(getActivity(), destinationDataModel.getDatas());
                    destinationRecyclerView.setAdapter(adapter);
                } else {
                    DestinationRecyclerAdapter adapter = (DestinationRecyclerAdapter) destinationRecyclerView.getAdapter();
                    adapter.datas.get(0).getInfos().addAll(destinationDataModel.getDatas().get(0).getInfos());
                }
            }
```
上面的结果是，请求两次网络。我们会看到数据重复了，同样的数据被发送了两次，并且是从头到尾的重复了两次。

### defer
延迟操作，等到订阅的时候再准备数据流。这里尤其对 just 和 from 操作符的效果最为明显,以下是国外的一个 just 的例子说明，from 的原理一样。
[借用一个国外的例子](http://blog.danlew.net/2015/07/23/deferring-observable-code-until-subscription-in-rxjava/)
[国内的翻译](http://www.jianshu.com/p/c83996149f5b)
```java
public class SomeType {
private String value;

public void setValue(String value) {
  this.value = value;
}

public Observable<String> valueObservable() {
  return Observable.just(value);
}
}
```
对于以下代码的调用会出现怎么样的结果呢？
```java
SomeType instance = new SomeType();  
Observable<String> value = instance.valueObservable();  
instance.setValue("Some Value");  
value.subscribe(System.out::println); //订阅的时候发射数据
```
如果你认为会打印出 “Some Value”，那就错了。而实际打印结果是 “null”。因为在调用 Observable.just() 的时候，value 已经初始化了。
just()，from() 这类能够创建 Observable 的操作符（译者注：创建 Observable 的操作符）在创建之初，就已经存储了对象的值，而不被订阅的时候。订阅的时候只是发射数据。
这种情况，显然不是预期表现，我想要的 valueObservable() 是无论什么时候请求，都能够表现为当前值。所以我们需要延迟数据的创建直到有人订阅。有两个方法，一个是用 create 自主创建，我们可以自己精确的控制发射什么，什么时候发射，还有一个是用的 defer 延迟操作符。defer() 中的代码直到被订阅才会执行。我们只需要在请求数据的时候调用 Observable.just() 就行了，使用 defer() 操作符的唯一缺点就是，每次订阅都会创建一个新的 Observable 对象。create() 操作符则为每一个订阅者都使用同一个函数，所以，后者效率更高。

因为我学习的时候，难以想清楚延迟和 create 操作符中的 call 的时间顺序和区别，我们用另外一个例子解释一下。
```java
private Observable<Integer> getInt() {

        SimpleDateFormat sdf = new SimpleDateFormat("yyyyMMdd_HHmmss");
        String currentDateandTime = sdf.format(new Date());
        Log.e("GetInt", currentDateandTime);

        return Observable.create(new Observable.OnSubscribe<Integer>() {
            @Override
            public void call(Subscriber<? super Integer> subscriber) {
                if (subscriber.isUnsubscribed()) {
                    return;
                }
                subscriber.onNext(42);
                subscriber.onCompleted();
            }
        });
    }

    //simple defer
    private void simpleDefer() {
        //defer中的getInt操作等到有人订阅deferObservable的时候才会被执行
        //假如这里不用defer，直接用getInt返回，那么调用simpleDefer的时候就会打印时间
        deferObservable = Observable.defer(new Func0<Observable<Integer>>() {
            @Override
            public Observable<Integer> call() {
                return getInt();
            }
        });
//        deferObservable.subscribe(new Action1<Integer>() {
//            @Override
//            public void call(Integer integer) {
//                System.out.println("subscribe:" + integer);
//            }
//        });
    }
```
在不用延迟的情况下，我们调用 simpleDefer 返回一个数据流的时候就会打印时间，反之我们不用延迟的话，则会在调用 simpleDefer 的时候就已经打印了当前的时间。
所以，这里被延迟的是我们 getInt 被调用的时机。注意：create 中的发射 42 和延迟 *无关*，这个 call 函数就是在 *发射* 数据，*订阅的时候才会发射数据* ，一旦订阅发生的额时候，就会发射42。

总之记住，defer 延迟的是参数 function 中的操作。只要将需要延迟创建的操作放到 function 函数中即可。这个对于数据的新鲜度有要求的操作很有用。

### filter
我们接受到的数据常常用些是不满足我们的需求的，这时候就可以用 filter 操作符。
```java
getDestinationDataObservableByCreate(url).filter(new Func1<DestinationDataModel, Boolean>() {
           @Override
           public Boolean call(DestinationDataModel destinationDataModel) {
               return destinationDataModel != null;
           }
       })
           .subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread())
           .subscribe(subscriber);
```
比如我们可以对发射的数据中 null 数据进行过滤，虽然我们也可以在 onNext 或者在 subscriber 中进行过滤，那样就会破坏代码的业务逻辑，这样，每个函数只要注重自身的业务逻辑即可。

### Map
我们有时候随着需求的变更，版本的迭代，可能用同一套数据可能会做不用的用处，亦或者同一个功能的同一个数据源，但是上层的应用对数据结构的需求发生了变化。这时候，如果我们去变更数据提供层，或者让上层去适配，都会破坏代码逻辑。

```java
getDestinationDataObservableByCreate(url).map(new Func1<DestinationDataModel, String>() {
    @Override
    public String call(DestinationDataModel destinationDataModel) {
        return destinationDataModel.getVersion();
    }
}).subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Action1<String>() {
        @Override
        public void call(String s) {
            System.out.println(s);
        }
    });
```
看上面，我们并没有修改 getDestinationDataObservableByCreate 的业务逻辑，这样就不会影响其他的代码逻辑，也不会去贸然修改底层数据提供，用 map 操作符，我们就将 getDestinationDataObservableByCreate 发射的 DestinationDataModel 类型的数据，一个个变换成了 String 类型。
map适用于这些数据结构的变化的操作。
### FlatMap && ConcatMap
对于一些 Observable 本身也会返回 Observable，我们可以将这些子 Observable 发射的数据进行统一，合并这些 Observables 发射的数据，最后将合并后的结果作为最终的 Observable。
这就是为什么叫做铺平的原因。
提示：合并部分是允许交叉的。意味着 flatMap() 不能够保证在最终生成的 Observable 中源 Observables 确切的发射顺序。ConcatMap 可以保证顺序，用法和 flatMap 一样。
```java
private Observable<Observable<DestinationDataModel>> getDestinationDataObservableByFlatMap(final String url) {
        return Observable.create(new Observable.OnSubscribe<Observable<DestinationDataModel>>() {
            @Override
            public void call(Subscriber<? super Observable<DestinationDataModel>> subscriber) {
                Gson gson = new Gson();
                Request request = new Request.Builder().url(url).build();
                Response response = null;
                try {
                    response = client.newCall(request).execute();
                    DestinationDataModel destinationDataModel = gson.fromJson(response.body().string(), DestinationDataModel.class);
                    if (subscriber.isUnsubscribed()) {//判断连接是否断开，避免无谓的操作
                        return;
                    }
                    subscriber.onNext(Observable.just(destinationDataModel));
                    subscriber.onNext(Observable.just(destinationDataModel));
                    subscriber.onCompleted();
                    if (!subscriber.isUnsubscribed()) {
                        subscriber.onCompleted();
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        });
    }
```

```java
getDestinationDataObservableByFlatMap(url).flatMap(new Func1<Observable<DestinationDataModel>, Observable<DestinationDataModel>>() {
            @Override
            public Observable<DestinationDataModel> call(Observable<DestinationDataModel> destinationDataModelObservable) {
                return destinationDataModelObservable;
            }
        })
        .subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread())subscribe(subscriber);
```

上面我们用 subscriber.onNext(Observable.just(destinationDataModel));模拟发送两个 Observable,然后通过 flatMap 进行扁平化处理。我们看到效果是列表中有两组相同的数据。
flatMap() 和 map() 有一个相同点：它也是把传入的参数转化之后返回另一个对象。 但需要注意，和 map() 不同的是， flatMap() 中返回的是个 Observable 对象，并且这个 Observable 对象并不是被直接发送到了 Subscriber 的回调方法中。
 flatMap() 的原理是这样的：
 1. 使用传入的事件对象创建一个 Observable 对象；
 2. 并不发送这个 Observable, 而是将它激活，于是它开始发送事件；
 3. 每一个创建出来的 Observable 发送的事件，都被汇入同一个 Observable ，而这个 Observable 负责将这些事件统一交给 Subscriber 的回调方法。
这三个步骤，把事件拆成了两级，通过一组新创建的 Observable 将初始的对象『铺平』之后通过统一路径分发了下去。 而这个『铺平』就是 flatMap() 所谓的 flat。

### SwitchMap
和上面的操作符类似 ，都是 Observable 发射 Observable,不同的是，这个操作不会合并数据项中的所有数据，而是当遇到后一个 Observable 发射数据的时候，就停止对前一个 Observable 的接收。
```java
getDestinationDataObservableByFlatMap(url).switchMap(new Func1<Observable<DestinationDataModel>, Observable<DestinationDataModel>>() {
    @Override
    public Observable<DestinationDataModel> call(Observable<DestinationDataModel> destinationDataModelObservable) {
        return destinationDataModelObservable;
    }
}).subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread())
    .subscribe(subscriber);
```
只要换个操作符即可 switchMap，这里因为发射的数据只有一个，效果不明显，如果是列表，交叉发射的话，会很明显，效果回事丢失一部分数据。

### GroupBy

我们对数据按照某个依据进行分组。
```java
Observable<GroupedObservable<String, DestinationDataModel>> groupedObservableObservable = getDestinationDataObservableByCreate(url).groupBy(new Func1<DestinationDataModel, String>() {
            @Override
            public String call(DestinationDataModel destinationDataModel) {
                return destinationDataModel.getVersion();
            }
        });
        Observable.concat(groupedObservableObservable).subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread()).subscribe(subscriber);
```
以上依旧版本号对数据进行分组，一组的将在一起当做一个 Observable 发射。

### merge
对数据进行整合一起发射。
```java
Observable<DestinationDataModel> merge1 = getDestinationDataObservableByCreate(url);
Observable<DestinationDataModel> merge2 = getDestinationDataObservableByCreate(url);
Observable<DestinationDataModel> merge = Observable.merge(merge1, merge2);
merge.subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread()).subscribe(subscriber);
```
merge 作为一个可观测序列，发射源 merge1，merge2 中的所有数据。注意发射的数据被交叉合并到一个 Observable 里面。如果同步的合并 Observable，它们将连接在一起并且不会交叉。

### zip $$ join && combineLatest
上面的数据，是原样的放在一个可观测序列中进行发射的，然而如果我们想对源数据中两两的进行操作后再放到一个数据列中呢？
```java
Observable<DestinationDataModel> zip1 = getDestinationDataObservableByCreate(url);
       Observable<DestinationDataModel> zip2 = getDestinationDataObservableByCreate(url);
       Observable<DestinationDataModel> zip = Observable.zip(zip1, zip2, new Func2<DestinationDataModel, DestinationDataModel, DestinationDataModel>() {
           @Override
           public DestinationDataModel call(DestinationDataModel destinationDataModel, DestinationDataModel destinationDataModel2) {
               //进行数据的合并操作
               destinationDataModel.setCode(destinationDataModel.getCode() + destinationDataModel2.getCode());
               return destinationDataModel;
           }
       });
       zip.subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread()).subscribe(subscriber);
```

上面在 Func2 中对两个数据源中的数据进行两两操作，我们只是简单的将 model 的 code 值相加来模拟合并操作，作为新的数据返回，假如两数据源的长度相同，则只会返回一个数据源长度的数据，假如长度不一样，以短的数据长度为准。

zip 作用于最近未打包的两个 Observables,还有一个需求就是我们不一定要非都是未打包的，并不一定要两个数据源要一定的长度相等，这个时候可以用 combineLatest，相反，combineLatest() 作用于最近发射的数据项：如果 Observable1 发射了 A 并且 Observable2 发射了 B 和 C，combineLatest() 将会分组处理 AB 和 AC。

join 操作符把类似于 combineLatest 操作符，也是两个 Observable 产生的结果进行合并，合并的结果组成一个新的 Observable，但是 join 操作符可以控制每个 Observable 产生结果的生命周期，在每个结果的生命周期内，可以与另一个 Observable 产生的结果按照一定的规则进行合并。


## RxAndroid rxbinding
Jake 为 Android 控件写的包，这个的使用就太多了。这个我们以一个登陆界面为例。
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/linearLayoutRoot"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <TextView
        android:id="@+id/textViewRxAndroid"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Result" />

    <EditText
        android:id="@+id/editTextRXAndroid"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="输入自己的内容，上面的textview会在400毫秒之后显示出来"
        android:textColor="@color/blue_light" />

    <EditText
        android:id="@+id/email"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="Email"
        android:textColor="@color/blue_light" />

    <EditText
        android:id="@+id/phone"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="Phone"
        android:textColor="@color/blue_light" />

    <EditText
        android:id="@+id/username"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="UserName"
        android:textColor="@color/blue_light" />

    <Button
        android:id="@+id/LoginButton"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:enabled="false"
        android:text="注册" />

</LinearLayout>
```
我们用第一个 TextView 来演示怎么样定时操作 TextView 的响应。场景常见于搜索，智能提示，我们不能用户输入什么就不停地网络请求。比如用户停止输入的内容有变，500秒之后再进行。

```java
//两个事件源的时间间隔小于规定的时间单位的，都会被忽略。
private void simpleDebounce() {
        RxTextView.textChangeEvents(editTextRXAndroid)
            .debounce(400, TimeUnit.MILLISECONDS)
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(new Observer<TextViewTextChangeEvent>() {
                @Override
                public void onCompleted() {
                }

                @Override
                public void onError(Throwable e) {
                }

                @Override
                public void onNext(TextViewTextChangeEvent textViewTextChangeEvent) {
                  //模拟响应
                    textViewRxAndroid.setText(textViewTextChangeEvent.text());
                }
            });
}
```
以上的代码中，在400毫秒的时间窗口内，如若临近的事件发生间隔在400毫秒以内的，都将被忽略，当相差的时间间隔达到400毫秒到达的时候，发射最近的一次事件。


在登录的时候，必须验证三要素都齐全才能进行注册的网络请求
```java
//使用combineLatest合并最近N个结点
private void simpleCombineLatest() {
        Observable<CharSequence> emailChangeObservable = RxTextView.textChanges(email);
        Observable<CharSequence> phoneChangeObservable = RxTextView.textChanges(phone);
        Observable<CharSequence> usernameChangeObservable = RxTextView.textChanges(username);
        Observable.combineLatest(emailChangeObservable, phoneChangeObservable, usernameChangeObservable, new Func3<CharSequence, CharSequence, CharSequence, Boolean>() {
            @Override
            public Boolean call(CharSequence charSequence, CharSequence charSequence2, CharSequence charSequence3) {
                if (!TextUtils.isEmpty(charSequence) && !TextUtils.isEmpty(charSequence2) && !TextUtils.isEmpty(charSequence3)) {
                    return true;
                } else {
                    return false;
                }
            }
        }).subscribe(new Action1<Boolean>() {
            @Override
            public void call(Boolean aBoolean) {
                if (aBoolean) {
                    LoginButton.setEnabled(true);
                } else {
                    LoginButton.setEnabled(false);
                }
            }
        });

}
```
我们使用 combineLatest 操作符，合并三个 TextView 的事件源，判断三个事件源都满足条件的时候，我们就返回 true，接着进行响应。


有时候，我们获取数据的渠道有很多，最后我们还是需要放到一起进行显示。
```java
  //使用merge合并两个数据源。
    private void simpleMerge() {
        Observable.merge(getDataFromFile(), getDataFromNet())
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(new Action1<String>() {
                @Override
                public void call(String s) {
                    textViewRxAndroid.setText(s + textViewRxAndroid.getText());
                }
            });
    }
```
以上我们对来自于本地的和网络请求的数据进行 merge 操作，产生新的数据流。


对于数据获取，我们有时候也有优先级，先去判断内存中是否有数据，有我们取内存中的，依次接下里是本地缓存，再次我们再去请求网络获取数据。
```java
//依次检查memory、disk、network
        Observable.concat(memory, disk, network)
            .first()
            .subscribeOn(Schedulers.newThread())
            .subscribe(new Action1<String>() {
                @Override
                public void call(String s) {
                    System.out.println("选择了：" + s);
                }
            });
```


使用 timer 做定时操作。当有“x秒后执行y操作”类似的需求的时候，想到使用 timer
```java
private void simpleTimer() {
        Observable.timer(2, TimeUnit.SECONDS)
            .subscribe(new Observer<Long>() {
                @Override
                public void onCompleted() {
                    System.out.println("现在是2秒之后");
                }

                @Override
                public void onError(Throwable e) {

                }

                @Override
                public void onNext(Long aLong) {

                }
            });

}
```


使用 interval 做周期性操作。当有“每隔xx秒后执行yy操作”类似的需求的时候，想到使用 interval
```java
private void simpleInterval() {
        Observable.interval(2, TimeUnit.SECONDS)
            .subscribe(new Observer<Long>() {
                @Override
                public void onCompleted() {
                }

                @Override
                public void onError(Throwable e) {
                }

                @Override
                public void onNext(Long number) {
                }
            });
    }
```
以两秒为周期，不断的发送数据。

使用 throttleFirst 防止按钮重复点击,ps：debounce 也能达到同样的效果
```java
    private void simpleThrottleFirst() {
        RxView.clicks(LoginButton)
            .throttleFirst(1, TimeUnit.SECONDS)
            .subscribe(new Subscriber<Void>() {
                @Override
                public void onCompleted() {

                }

                @Override
                public void onError(Throwable e) {

                }

                @Override
                public void onNext(Void aVoid) {

                }
            });
    }
```
以上的含义是在一秒以内，只接收第一个数据，其他的将会被忽略。



