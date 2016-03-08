## Effective RxJava

This is a collection of items, each specifying one general rule, to help you write [RxJava](https://github.com/ReactiveX/RxJava) code more effectively. It is modeled after two of my favorite technical books, _Effective C++_ and _Effective Java_. I appreciate not only the soundness of their contents, but the brevity of their writing style. I hope that Effective RxJava has the same value proposition.

For each rule I've attempted to provide relevant example code -- most of which is currently in production in the [Khan Academy Android app](https://play.google.com/store/apps/details?id=org.khanacademy.android).

### Items

* [Use Retrolambda](items/use-retrolambda.md)
* [Understand `Observable` and observer chains](items/understand-observable-and-observer-chains.md)
* [Understand `subscribeOn` and `observeOn`](items/understand-subscribeon-and-observeon.md)
* [Understand `switchMap`](items/understand-switch-map.md)
* [Convert functions to `Observable`](items/convert-functions-to-observable.md)
* [Convert callbacks to `Observable`](items/convert-callbacks-to-observable.md)
* [Optimize your subscribers](items/optimize-your-subscribers.md)
* [Use `compose` for operator sequences](items/use-compose-for-operator-sequences.md)
* [Emit immutable values](items/emit-immutable-values.md)
* [Test emitted values using `TestObserver`](items/test-emitted-values-using-testobserver.md)
* [Test work scheduling using `TestScheduler`](items/test-work-scheduling-using-testscheduler.md)

