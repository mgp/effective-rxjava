### Understand `subscribeOn` and `observeOn`

By default, `Observable` instances specify an execution policy of "immediate." That is, if a thread has an action to perform, then it is executed immediately by that thread. We can observe this in our banal example from the item [Understand `Observable` and observer chains](understand-observable-and-observer-chains.md). To refresh our memory, that example is:

```java
Observable<Integer> o1 = Observable.just(1, 2, 3, 4, 5);
Observable<Integer> o2 = o1.filter(x -> (x % 2) == 1);
Observable<Integer> o3 = o2.map(x -> x * x);
o3.subscribe(integer -> System.out.println("Received value: " + integer));
```

We can observe the effects of the immediate scheduling policy by setting a breakpoint on the line with `subscribe` and then running the program. When this breakpoint is reached the stack trace shows:

![Stack trace for the observed value](images/observer-chain-stacktrace.png)

Note that at the bottom of the stacktrace we find the call to `subscribe` on `o3`. The item [Understand `Observable` and observer chains](understand-observable-and-observer-chains.md) explains in detail why this happens, but to summarize:

* The client explicitly calls `subscribe` on `o3`.
* `o3` then implicitly calls `subscribe` on `o2`.
* `o2` then implicitly calls `subscribe` on `o1`.
* The `subscribe` method of `o1` then performs its work, by emitting each value that it was constructed with.
* The first value of `1` passes the `filter` operator, and then its value is squared. We observe this value at the breakpoint.

This stacktrace gives a complete picture. We can follow the thread of execution from its call to `subscribe` on `o3` to each observed value.

But while this immediate execution policy is easy to reason about, it is not ideal when the work performed by the root `Observable` instance is CPU-intensive or performs I/O. In such cases, the thread of execution can spend significant time computing the result or blocking on a read or write operation. And if the application is an Android application, then the thread of execution is likely the main thread. Consequently the UI will freeze entirely and the user might have no choice but to force-quit your unresponsive application.

To remedy these problems, we can introduce `Scheduler` instances via methods `subscribeOn` or `observeOn`.

#### Choosing a scheduler

A `Scheduler` instance defines the thread on which actions should happen. If you are familiar with the concurrency libraries from Java, you can think of a `Scheduler` as resembling an [`Executor`](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Executor.html). (In fact, the [`Schedulers` class from RxJava](http://reactivex.io/RxJava/javadoc/rx/schedulers/Schedulers.html) has a `from` method that adapts an `Executor` instance to the `Scheduler` interface.)

The `Schedulers` class has static factory methods for creating common `Scheduler` implementations:

* The `computation` method returns a `Scheduler` for CPU-intensive work. The work is typically performed by a backing [thread pool](https://en.wikipedia.org/wiki/Thread_pool), where the number of threads equals the number of CPU cores. If new work is scheduled but all thread in the pool are busy, then that work is enqueued for later execution.
* The `io` method returns a `Scheduler` for I/O-bound work. The work is performed by a backing thread pool that grows as needed. This growth strategy is chosen because most threads in this pool will be blocked on a read or on a write operation. These threads do not contend for the CPU.
* The `immediate` method returns a `Scheduler` that executes work immediately on the current thread. It resembles the default execution policy explored at the beginning of this item.

Finally, if your application has a default [run loop](https://en.wikipedia.org/wiki/Event_loop), then consider a `Scheduler` implementation that adapts it. For example, the [RxAndroid](https://github.com/ReactiveX/RxAndroid) project provides an `AndroidSchedulers` class. Its static method `mainThread` returns a `Scheduler` for the main UI thread.

Some methods of `Observable` that perform work take a `Scheduler` parameter specifying the thread on which the work is performed. But for the most part, you specify a `Scheduler` in your chain of `Observable` instances using the `subscribeOn` and `observeOn` methods. Let's consider `subscribeOn` first.

#### Using `subscribeOn`

The `subscribeOn` method applies to upstream `Observable` instances. Its `Scheduler` parameter specifies the thread on which the upstream `subscribe` method is invoked. Consequently, if the root `Observable` instance performs work, a downstream `subscribeOn` specifies the thread on which the work is performed. This allows us to offload I/O or CPU-intensive tasks to another thread.

Let's redefine `o1` so that it reads the integers that it emits from a file. Moreover, we will print the name of the thread of execution:

```java
Observable<Integer> o1 = Observable.create(subscriber -> {
    System.out.println("Reading file on thread " + Thread.currentThread().getName());

    final List<Integer> integers = readIntegersFromFile();
    for (Integer integer : integers) {
        subscriber.onNext(integer);
    }
    subscriber.onCompleted();
});
```

We do not want method `readIntegersFromFile` to execute on the main thread, and so we pass the `Schedulers.io()` instance to the `subscribeOn` method downstream:

```
Observable<Integer> o2 = o1.filter(x -> (x % 2) == 1);
Observable<Integer> o3 = o2.map(x -> x * x);
Observable<Integer> o4 = o3.subscribeOn(Schedulers.io())
o4.subscribe(integer -> System.out.println("Received value: " + integer));
```

This revises our summary of how the observer chain is created:

* The client explicitly calls `subscribe` on `o4`.
* `o4` then enqueues on the `Schedulers.io()` instance the work of calling `subscribe` on `o3`.
* A thread belonging to the thread pool backing `Schedulers.io()` executes this work, thereby calling `subscribe` on `o3`.
* `o3` then implicitly calls `subscribe` on `o2`, which implicitly calls `subscribe` on `o1`, which then performs the work of calling method `readIntegersFromFile` and emitting each returned value.

Consequently, `o1` performs its work on a thread belonging to the thread pool backing the `Schedulers.io()` instance. We validate this by running the program:

```text
Reading file on thread RxCachedThreadScheduler-1
Received value: 1
Received value: 9
Received value: 25
```

`RxCachedThreadScheduler-1` is the name of the thread in the thread pool backing the `Schedulers.io()` instance. As desired, we have offloaded the work of reading the file to another thread.

But there is one consequence that is not obvious: Because a thread belonging to `Schedulers.io()` is executing the work of `o1`, then it is that thread that invokes the downstream operators and calls our observer passed to the `subscribe` method of `o4`. We can verify this by printing the thread name at that point:

```java
o4.subscribe(integer -> {
    System.out.println("Received value " + integer + " on thread " + Thread.currentThread().getName()));
});
```

When run, this prints:

```text
Reading file on thread RxCachedThreadScheduler-1
Received value 1 on thread RxCachedThreadScheduler-1
Received value 9 on thread RxCachedThreadScheduler-1
Received value 25 on thread RxCachedThreadScheduler-1
```

Above the observer passed to the `subscribe` method of `o4` simply prints the emitted values to the screen. But we can easily imagine cases where the provided observer must execute on the main thread. (Such as an Android application where the upstream `Observable` makes a network request, and the downstream observer consumes this response by updating the application state.)

To ensure that the emitted events are observed on the desired thread, we use method `observeOn`.

#### Using `observeOn`

The `observeOn` method applies to downstream `Observable` instances. Its `Scheduler` parameter specifies the thread on which events, such as the next emitted value or the stream terminating normally or with an error, are observed downstream.

As we have seen, if you use `subscribeOn` so that the upstream `Observable` instance performs work on another thread, and if that work emits events that is consumed downstream, then you should typically also use `observeOn`. This ensures that the consumption of the events happen on the proper thread, which is typically the main thread that called `subscribe`.

Assuming we are writing an Android application, we can observe the events on the main thread like so:

```java
Observable<Integer> o4 = o3
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread());
o4.subscribe(integer -> {
    System.out.println("Received value " + integer + " on thread " + Thread.currentThread().getName()));
});
```

When run, this prints:

```text
Reading file on thread RxCachedThreadScheduler-1
Received value 1 on thread main
Received value 9 on thread main
Received value 25 on thread main
```

Note that by placing the `observeOn` method immediately before the call to `subscribe` on `o4`, it is the `RxCachedThreadScheduler-1` thread that executes the logic of the `filter` and `map` operators. This is ideal, as you should not schedule more work than necessary on your main thread.

Finally, as mentioned earlier, near every call to `subscribeOn` should be a call to `observeOn`. As [Dan Lew](http://blog.danlew.net/) suggests in his excellent [Don't break the chain](http://blog.danlew.net/2015/03/02/dont-break-the-chain/) article, consider defining a `Transformer` that calls both of these methods.

