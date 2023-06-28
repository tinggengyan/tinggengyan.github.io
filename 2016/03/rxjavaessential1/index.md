# RxJava源码解析(一)

# 概述
本文目的是从源码的角度讲解 Rxjava 的重要概念 Observable.

<!-- more -->

# RxJava要点解析
对于 Rxjava 还是有很多不理解的地方，加上又有点好奇心，就看看源码，记录在此，水平有限，肯定存在错误的地方，望路过的同行不吝赐教。

## lift变换操作的原理
看下lift源码,直接拷贝的源码，未做删减。
```java
public final <R> Observable<R> lift(final Operator<? extends R, ? super T> operator) {
        return new Observable<R>(new OnSubscribe<R>() {
            @Override
            public void call(Subscriber<? super R> o) {
                try {
                    Subscriber<? super T> st = hook.onLift(operator).call(o);
                    try {
                        // new Subscriber created and being subscribed with so 'onStart' it
                        st.onStart();
                        onSubscribe.call(st);
                    } catch (Throwable e) {
                        // localized capture of errors rather than it skipping all operators
                        // and ending up in the try/catch of the subscribe method which then
                        // prevents onErrorResumeNext and other similar approaches to error handling
                        Exceptions.throwIfFatal(e);
                        st.onError(e);
                    }
                } catch (Throwable e) {
                    Exceptions.throwIfFatal(e);
                    // if the lift function failed all we can do is pass the error to the final Subscriber
                    // as we don't have the operator available to us
                    o.onError(e);
                }
            }
        });
}
 ```

 为了弄懂，而且变量不多，我们一个变量一个变量的看。

 - lift内部返回的是一个新建的observable，此时产生了一个新的OnSubscribe，此时的OnSubscribe的call方法内传入的Subscriber
   变量o就是我们写在代码中的订阅者。

 - hook.onLift(operator).call(o);新建了一个Subscriber变量st，这个变量用的是我们传入的Subscriber变量o。
   而且还是用的operator创建的，我们先不管如何实现的，待会儿我们看看这个是怎么实现的。

 - onSubscribe.call(st);这个onSubscribe是个final类型，因为目前我们还是处于方法内，所以这个onSubscribe还是源observable
   的onSubscribe对象(比如我们自己写的发射时机等那段代码),这个时候onSubscribe会调用它的call方法，传入的是我们新建的Subscriber变量st。
   接下来，新的Subscriber变量st会接收到源observable发送来的数据。我们可以自然得想到，这个新的st肯定会经过operator对象中的一些定义的方法对数据操作后，又发送到了我们传入的Subscriber变量o，实现整体的连接。

   其实说白了，我们其实是在中间创建了一个代理。hook的意思不就是钩子嘛。

   接下来来看刚刚未能解决的疑问，hook是怎么工作的。

  ```java
  public <T, R> Operator<? extends R, ? super T> onLift(final Operator<? extends R, ? super T> lift) {
         return lift;
     }
  ```

  我们看到，并未做任何变化，直接将operator变换直接返回了。


  接着我们继续看，以filter为例。
  ```java
  public final Observable<T> filter(Func1<? super T, Boolean> predicate) {
          return lift(new OperatorFilter<T>(predicate));
      }
  ```

 传入的是一个 OperatorFilter 对象。

 ```java
 public final class OperatorFilter<T> implements Operator<T, T> {

     private final Func1<? super T, Boolean> predicate;

     public OperatorFilter(Func1<? super T, Boolean> predicate) {
         this.predicate = predicate;
     }

     @Override
     public Subscriber<? super T> call(final Subscriber<? super T> child) {
         return new Subscriber<T>(child) {

             @Override
             public void onCompleted() {
                 child.onCompleted();
             }

             @Override
             public void onError(Throwable e) {
                 child.onError(e);
             }

             @Override
             public void onNext(T t) {
                 try {
                     if (predicate.call(t)) {
                         child.onNext(t);
                     } else {
                         // TODO consider a more complicated version that batches these
                         request(1);
                     }
                 } catch (Throwable e) {
                     Exceptions.throwOrReport(e, child, t);
                 }
             }

         };
     }

 }
 ```

 变量predicate就是我们自己定义的过滤规则，在上面的代码中我们已经看到了，call传入的child就是我们上面分析，我们自己定义的变量o。

 这下基本清晰了，每次源observable发射的数据都被OperatorFilter内新的subscriber给接收了，然后根据传入到OperatorFilter我们自己定义的过滤规则进行判断，通过的，就给child发射过去，这样实现了过滤的作用，实现了新的subscriber将数据传送到了我们自定义的subscriber。


## Scheduler 线程切换的原理
注意：以下的版本是rxjava1.1.1(上下两部分的总结时间不一样，也不去考证是哪个版本了)
上面说到了变换的时候，用到的线程的切换的问题，那到底是怎么切换的线程呢？
说到线程切换，必须说到两个操作。
- subscribeOn：指定observable调用obsubsriber发射数据所在的线程。
- observeOn： 指定订阅者进行订阅处理所在的线程。

### subscribeOn分析

```java
public final Observable<T> subscribeOn(Scheduler scheduler) {
        if (this instanceof ScalarSynchronousObservable) {
            return ((ScalarSynchronousObservable<T>)this).scalarScheduleOn(scheduler);
        }
        return create(new OperatorSubscribeOn<T>(this, scheduler));
}
```
以上是subscribeOn的源码，传入了指定的发送数据所在的线程Scheduler对象。判断当前的observable是否是一个ScalarSynchronousObservable，这个ScalarSynchronousObservable对象就是直接发射传入数据的对象，就是我们平时使用just所产生的对象。

-  如果是，则调用将scheduler传入scalarScheduleOn方法，创建一个新的Observable作为返回值。
```java
public Observable<T> scalarScheduleOn(final Scheduler scheduler) {
        final Func1<Action0, Subscription> onSchedule;
        if (scheduler instanceof EventLoopsScheduler) {
            onSchedule = COMPUTATION_ONSCHEDULE;
        } else {
            onSchedule = new Func1<Action0, Subscription>() {
                @Override
                public Subscription call(final Action0 a) {
                    final Scheduler.Worker w = scheduler.createWorker();
                    w.schedule(new Action0() {
                        @Override
                        public void call() {
                            try {
                                a.call();
                            } finally {
                                w.unsubscribe();
                            }
                        }
                    });
                    return w;
                }
            };
        }
        return create(new ScalarAsyncOnSubscribe<T>(t, onSchedule));
    }
```
以上是scalarScheduleOn的源码，可以看到内部是通过Func1函数进行转换的，通过传入的scheduler创建指定的线程，在指定的线程上调用Func1中传入进来的Action0。
那么Action0代表的又是什么呢？我们看到onSchedule又作为参数去构造ScalarAsyncOnSubscribe了。

```java
/**
     * The OnSubscribe implementation that creates the ScalarAsyncProducer for each
     * incoming subscriber.
     *
     * @param <T> the value type
     */
    static final class ScalarAsyncOnSubscribe<T> implements OnSubscribe<T> {
        final T value;
        final Func1<Action0, Subscription> onSchedule;

        ScalarAsyncOnSubscribe(T value, Func1<Action0, Subscription> onSchedule) {
            this.value = value;
            this.onSchedule = onSchedule;
        }

        @Override
        public void call(Subscriber<? super T> s) {
            s.setProducer(new ScalarAsyncProducer<T>(s, value, onSchedule));
        }
    }
```
根据代码注释，知道是为每个传入进来的subscriber创建一个ScalarAsyncProducer。看到这里的call方法，我们可以知道，这个call方法就是被Observable在create中被调用的call方法。在call方法里调用了Subscriber的setProducer方法，给它设置了一个ScalarAsyncProducer对象，这里的Subscriber对象s就是我们自定义的订阅者对象，接下来就看看ScalarAsyncProducer的源码。

```java
/**
     * Represents a producer which schedules the emission of a scalar value on
     * the first positive request via the given scheduler callback.
     *
     * @param <T> the value type
     */
    static final class ScalarAsyncProducer<T> extends AtomicBoolean implements Producer, Action0 {
        /** */
        private static final long serialVersionUID = -2466317989629281651L;
        final Subscriber<? super T> actual;
        final T value;
        final Func1<Action0, Subscription> onSchedule;

        public ScalarAsyncProducer(Subscriber<? super T> actual, T value, Func1<Action0, Subscription> onSchedule) {
            this.actual = actual;
            this.value = value;
            this.onSchedule = onSchedule;
        }

        @Override
        public void request(long n) {
            if (n < 0L) {
                throw new IllegalArgumentException("n >= 0 required but it was " + n);
            }
            if (n != 0 && compareAndSet(false, true)) {
                actual.add(onSchedule.call(this));
            }
        }

        @Override
        public void call() {
            Subscriber<? super T> a = actual;
            if (a.isUnsubscribed()) {
                return;
            }
            T v = value;
            try {
                a.onNext(v);
            } catch (Throwable e) {
                Exceptions.throwOrReport(e, a, v);
                return;
            }
            if (a.isUnsubscribed()) {
                return;
            }
            a.onCompleted();
        }

        @Override
        public String toString() {
            return "ScalarAsyncProducer[" + value + ", " + get() + "]";
        }
    }
```
注释上说，这个类是用来代表一个生产者，这个生产者通过给定的线程回调，在第一个激活的请求上发射数据。在具体看代码之前，我们可以根据注释大致猜到这个是充当了发射数据的生产者，并且要是能实现线程切换，应该是在作为构造参数传递进来的onSchedule上进行调用和回调的。看下里面的源码，在request方法中，这里的actual就是刚刚传入的自定义订阅者对象s,我们看到call方法中就是直接调用了actual的onNext方法，将数据传递到订阅者。
再看重点，request方法，有个参数n代表一次性请求的数据的数量，actual.add(onSchedule.call(this))；这个命令我们看到有调用onSchedule的call方法，那么经过这么长时间下来，这个call方法又是啥玩意？这个onSchedule就是我们在scalarScheduleOn中定义的Func1，这个也就是说在这里调用了Func1的call方法，传递进去的按理说是Action0对象，我们注意到ScalarAsyncProducer已经实现了Action0接口。也就是说这里调用了ScalarAsyncProducer(生产者)的call方法。当前ScalarAsyncProducer的call方法就是直接调用自定义订阅者的onNext发射数据。我们在上面说了Func1的call方法会在一个新建的线程中调用call方法传递进来的action0对象(就是此处实现Action0接口的ScalarAsyncProducer)的call方法。到目前为止，关于ScalarSynchronousObservable对象的线程切换的原理就分析结束了。

-  接下来看下，如果不是ScalarSynchronousObservable对象调用subscribeOn方法又会是什么逻辑呢？

再贴下subscribeOn的源码。
```java
public final Observable<T> subscribeOn(Scheduler scheduler) {
        if (this instanceof ScalarSynchronousObservable) {
            return ((ScalarSynchronousObservable<T>)this).scalarScheduleOn(scheduler);
        }
        return create(new OperatorSubscribeOn<T>(this, scheduler));
    }
```

根据当前的observable对象和传递进来的scheduler创建了一个OperatorSubscribeOn对象，然后用这个OperatorSubscribeOn对象创建了一个新的Observable。

```java
/**
 * Subscribes Observers on the specified {@code Scheduler}.
 * <p>
 * <img width="640" src="https://github.com/ReactiveX/RxJava/wiki/images/rx-operators/subscribeOn.png" alt="">
 *
 * @param <T> the value type of the actual source
 */
public final class OperatorSubscribeOn<T> implements OnSubscribe<T> {

    final Scheduler scheduler;
    final Observable<T> source;

    public OperatorSubscribeOn(Observable<T> source, Scheduler scheduler) {
        this.scheduler = scheduler;
        this.source = source;
    }

    @Override
    public void call(final Subscriber<? super T> subscriber) {
        final Worker inner = scheduler.createWorker();
        subscriber.add(inner);

        inner.schedule(new Action0() {
            @Override
            public void call() {
                final Thread t = Thread.currentThread();

                Subscriber<T> s = new Subscriber<T>(subscriber) {
                    @Override
                    public void onNext(T t) {
                        subscriber.onNext(t);
                    }

                    @Override
                    public void onError(Throwable e) {
                        try {
                            subscriber.onError(e);
                        } finally {
                            inner.unsubscribe();
                        }
                    }

                    @Override
                    public void onCompleted() {
                        try {
                            subscriber.onCompleted();
                        } finally {
                            inner.unsubscribe();
                        }
                    }

                    @Override
                    public void setProducer(final Producer p) {
                        subscriber.setProducer(new Producer() {
                            @Override
                            public void request(final long n) {
                                if (t == Thread.currentThread()) {
                                    p.request(n);
                                } else {
                                    inner.schedule(new Action0() {
                                        @Override
                                        public void call() {
                                            p.request(n);
                                        }
                                    });
                                }
                            }
                        });
                    }
                };

                source.unsafeSubscribe(s);
            }
        });
    }
}
```
注释直接表明，这个类是用来在指定的线程上订阅subscriber。最主要的方法是call方法，在这个方法里，创建了一个Worker对象，Worker对象调用schedule方法传入一个Action0,在这里Action0的call方法被调用的时候，首先记录当前调用Action0的call方法的线程。在内部用传入的原始subscriber创建一个新的Subscriber对象s(两个subscriber了),在这个s的内部又进行原始subscriber的onNext方法调用发射数据，又覆写了新的subscriber对象的setProducer方法，在覆写的时候对原始的subscriber对象设置一个新的Producer(两个Producer)，在这个新的生产者的request方法中，会判断当前的线程会否就是scheduler指定的线程(刚刚在Action0的call方法中记录了)，如果是，则立即执行；否则将交给scheduler对象指定的Worker对象重新安排这个任务，直到线程一致为止才执行。最后让执行subscribeOn产生的新的observable去订阅这个新产生的subscriber即可。

####  subscribeOn总结：

线程的切换是由Scheduler对象中的Worker对象负责安排这些任务，不同类型的Scheduler创建出对应的Worker对象，在Worker对象内部会在相应的线程上创建新的Subscriber，通过给新的subscriber设置Producer，通过新的Producer在指定的线程上请求数据，代理了实现发射线程的切换。

### observeOn分析

接下来分析observeOn,了解在异步线程上分发事件就可以知道线程切换的大致原理了。

```java
public final Observable<T> observeOn(Scheduler scheduler) {
        if (this instanceof ScalarSynchronousObservable) {
            return ((ScalarSynchronousObservable<T>)this).scalarScheduleOn(scheduler);
        }
        return lift(new OperatorObserveOn<T>(scheduler, false));
}
```
可以看到假如是 ScalarSynchronousObservable 实例的话，操作和 subscribeOn 是一样的。直接看看第二种不是 ScalarSynchronousObservable 的情况。
我们看到使用的是 lift 操作符，Operator 的构造方式和上面的 filter 有点不一样，传入了 scheduler 和一个 false 。看下这个 OperatorObserveOn 的代码。按照刚刚 lift 操作的原理，操作符中的 call 方法是关键，是会被代理调用的。这边的代码太长，贴关键部分的代码，上面的都是全的。

```java
@Override
    public Subscriber<? super T> call(Subscriber<? super T> child) {
        if (scheduler instanceof ImmediateScheduler) {
            // avoid overhead, execute directly
            return child;
        } else if (scheduler instanceof TrampolineScheduler) {
            // avoid overhead, execute directly
            return child;
        } else {
            ObserveOnSubscriber<T> parent = new ObserveOnSubscriber<T>(scheduler, child, delayError);
            parent.init();
            return parent;
        }
    }
```
根据上面的经验，这里的返回的 Subscriber 的将会在 lift 新创建的 observable 中的 OnSubscribe 对象的 call 方法中被原始自定义的 OnSubscribe 当做参数传递给它自身的 call 方法中，这里有点乱，但是是和上面的 lift 是一样的，可以回到上面看看 lift 的分析。所以数据会先通过这里创建的 ObserveOnSubscriber ，也就是这里返回的 child 或者 parent。child 的应该比较简单，是直接返回自定义的  Subscriber，这样在 lift 中的情况也一样，其实是直接发给了原始的 observable，相当于没做任何的线程变换。
那么重点就是在 ObserveOnSubscriber 上了，这个应该是在做线程的切换了。

```java
        void init() {
            // don't want this code in the constructor because `this` can escape through the 
            // setProducer call
            Subscriber<? super T> localChild = child;
            
            localChild.setProducer(new Producer() {

                @Override
                public void request(long n) {
                    if (n > 0L) {
                        BackpressureUtils.getAndAddRequest(requested, n);
                        schedule();
                    }
                }

            });
            localChild.add(recursiveScheduler);
            localChild.add(this);
        }

        @Override
        public void onNext(final T t) {
            if (isUnsubscribed() || finished) {
                return;
            }
            if (!queue.offer(on.next(t))) {
                onError(new MissingBackpressureException());
                return;
            }
            schedule();
        }

```

上面看到，在创建 parent的时候，会调用 init 方法，在 init 方法中为 child (自定义的 subscriber )设置了 Producer，在 request 里调用了 schedule 方法，
schedule 方法就是执行在指定的异步线程上的。我们看到在 onNext 方法中，又调用了 schedule 方法，这样就实现了循环分发的效果，直到把所有的数据分发完。

```java
protected void schedule() {
            if (counter.getAndIncrement() == 0) {
                recursiveScheduler.schedule(this);
            }
        }
```

schedule方法里是将当前的对象加入到任务安排中，我们知道schedule方法的参数是一个Action0对象，所以需要看下当前对象实现Action0接口中的call方法。

```java
@Override
        public void call() {
            long emitted = 0L;

            long missed = 1L;

            // these are accessed in a tight loop around atomics so
            // loading them into local variables avoids the mandatory re-reading
            // of the constant fields
            final Queue<Object> q = this.queue;
            final Subscriber<? super T> localChild = this.child;
            final NotificationLite<T> localOn = this.on;
            
            // requested and counter are not included to avoid JIT issues with register spilling
            // and their access is is amortized because they are part of the outer loop which runs
            // less frequently (usually after each RxRingBuffer.SIZE elements)
            
            for (;;) {
                if (checkTerminated(finished, q.isEmpty(), localChild, q)) {
                    return;
                }

                long requestAmount = requested.get();
                boolean unbounded = requestAmount == Long.MAX_VALUE;
                long currentEmission = 0L;
                
                while (requestAmount != 0L) {
                    boolean done = finished;
                    Object v = q.poll();
                    boolean empty = v == null;
                    
                    if (checkTerminated(done, empty, localChild, q)) {
                        return;
                    }
                    
                    if (empty) {
                        break;
                    }
                    
                    localChild.onNext(localOn.getValue(v));
                    
                    requestAmount--;
                    currentEmission--;
                    emitted++;
                }
                
                if (currentEmission != 0L && !unbounded) {
                    requested.addAndGet(currentEmission);
                }
                
                missed = counter.addAndGet(-missed);
                if (missed == 0L) {
                    break;
                }
            }
            
            if (emitted != 0L) {
                request(emitted);
            }
        }
```

可以看到上面其实和 Android 本身的 looper 一样，也是一个循环发射的一个过程，checkTerminated 根据当前发射的完成状况和 subscriber 的订阅状态判断是否需要停止。不去过多的理解这个，看重点for循环。
在 for 循环里还有一个 while 循环，在 while 中给 Subscriber 对象发射数据。这样就是在 recursiveScheduler 中新的线程中发射的数据。这样就实现了 observeOn 中的线程切换。


#### observeOn总结
observeOn 的线程切换也是通过给自定义的 Subscriber 设置新的 Producer，在新的 Producer中 指定分发(subscriber.onNext())调用的线程,这样就实现了 observeOn 线程的切换。


### subscribe分析

再看看订阅的执行流程。

```java
public final Subscription subscribe(Subscriber<? super T> subscriber) {
        return Observable.subscribe(subscriber, this);
}
private static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
     // validate and proceed
        if (subscriber == null) {
            throw new IllegalArgumentException("observer can not be null");
        }
        if (observable.onSubscribe == null) {
            throw new IllegalStateException("onSubscribe function can not be null.");
        }
        // new Subscriber so onStart it
        subscriber.onStart();
        // if not already wrapped
        if (!(subscriber instanceof SafeSubscriber)) {
            // assign to `observer` so we return the protected version
            subscriber = new SafeSubscriber<T>(subscriber);
        }
        // The code below is exactly the same an unsafeSubscribe but not used because it would 
        // add a significant depth to already huge call stacks.
        try {
            // allow the hook to intercept and/or decorate
            hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
            return hook.onSubscribeReturn(subscriber);
        } catch (Throwable e) {
            // special handling for certain Throwable/Error/Exception types
            Exceptions.throwIfFatal(e);
            // if an unhandled error occurs executing the onSubscribe we will propagate it
            try {
                subscriber.onError(hook.onSubscribeError(e));
            } catch (Throwable e2) {
                Exceptions.throwIfFatal(e2);
                // if this happens it means the onError itself failed (perhaps an invalid function implementation)
                // so we are unable to propagate the error correctly and will just throw
                RuntimeException r = new RuntimeException("Error occurred attempting to subscribe [" + e.getMessage() + "] and then again while trying to pass to onError.", e2);
                // TODO could the hook be the cause of the error in the on error handling.
                hook.onSubscribeError(r);
                // TODO why aren't we throwing the hook's return value.
                throw r;
            }
            return Subscriptions.unsubscribed();
        }
    }
```
我们看到hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);这就是所谓的只要在订阅的时候才会发射数据的原因。 subscriber 作为参数传递到调用 subscribe 方法的 observable 中 onSubscribe 的 call 方法了。也就是说最终的subscriber是被传递给了最后一个调用它的 observable 了，因为我们知道在整个操作链中，每个操作符都会返回一个新的 observable ，并且内部都是创建了一个新的 subscriber ，利用代理的方式调用我们自定义的 subscriber。


### 实例讲解流程
```java
private void simpleFilter() {
    Observable.create(new Observable.OnSubscribe<Integer>() {
      @Override
      public void call(Subscriber<? super Integer> subscriber) {
        System.out.println("Observable:" + Thread.currentThread());
        for (int i = 0; i < 10; i++) {
          subscriber.onNext(i);
        }
        subscriber.onCompleted();
      }
    })
        //  决定了最终在哪个线程调用OnSubscribe的call方法，这会让源observable订阅subscribeOn内新创建的subscriber,
        //  内部新的subscriber会在指定的IO线程上执行。
        .subscribeOn(Schedulers.io())
            // filter会返回一个 observable，这个observable会订阅后面的subscriber，接收到之后交给Operator，Operator调用Func1操作完之后再交给调用这个filter
            // 的observable中的OnSubscribe调用，运行在调用这个filter的observable的call方法运行的线程上。
        .filter(new Func1<Integer, Boolean>() {
          @Override
          public Boolean call(Integer integer) {
            System.out.println("filter:" + Thread.currentThread());
            if (integer > 2) {
              return true;
            } else {
              return false;
            }
          }
        })

        // 指定subscriber中的call方法运行的线程，内部也是通过lift操作实现的，也新建了一个subscriber，
            // 这个新的subscriber为后面订阅的subscriber设置了新producer，
            // 新的producer指定了后面订阅的subscriber的分发数据的线程，也就是订阅的subscriber调用onNext的线程。
        .observeOn(AndroidSchedulers.mainThread())
            //订阅，直接运行，分发数据
        .subscribe(new Action1<Integer>() {
          @Override
          public void call(Integer integer) {
            System.out.println("Action1:" + Thread.currentThread());
            textViewMain.setText(integer.toString());
          }
        });

  }
```
具体的解释就在上面的注释部分了，暂时rxjava部分的解析先暂停，以上的内容，还有许多待斟酌的，遇到错误，看到的希望指教。

