### Use `compose` for operator sequences

> Note that the definitive article on this subject is [Don't break the chain: use RxJava's `compose()` operator](http://blog.danlew.net/2015/03/02/dont-break-the-chain/) by Android programmer extraordinaire [Dan Lew](http://blog.danlew.net/). You're better of reading that article instead of this one, but I am including this item for completeness.

In the [Khan Academy Android app](https://play.google.com/store/apps/details?id=org.khanacademy.android), we make extensive use of the [`Optional` class](http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/base/Optional.html) from [Google's Guava library](https://github.com/google/guava). Whereas we typically represent "absent values" by using `null`, the `Optional` class encodes the optionality of such values in the type system. This helps ensure that clients deal with such optional values in a direct manner. For example:

```java
final Optional<String> optional = Optional.of("abc");
if (optional.isPresent()) {
  final String s = optional.get();
}
```

The code above first creates an `Optional` instance with the string `"abc"`. Because that instance contains a value, we say that the value is *present*, not *absent*. Its `isPresent` method therefore returns `true`, and the call to `get` assigns the value `"abc"` to `s`.

Our application has many cases where an `Observable` emits `Optional<T>` for some type `T`, and downstream subscribers want to receive only the present values of type `T`.

For example, a `BookmarkEvent` instance represents any update to the user's bookmarks, such as adding a bookmark, removing a bookmark, or updating the download progress associated with a bookmark. Every `BookmarkEvent` instance has an `Optional` value of type `DownloadEvent` that is accessed through its `downloadEventOptional()` method. If the `BookmarkEvent` updates the download progress of a bookmark, then that `Optional<DownloadEvent>` contains a `DownloadEvent` instance with more details, such as how many bytes have been downloaded so far, what the estimated total byte count is, and so on. We can use this `DownloadEvent` to [display a notification](http://developer.android.com/guide/topics/ui/notifiers/notifications.html) with the download progress:

```java
mBookmarkManager.getBookmarkEventObservable()
        .map(bookmarkEvent -> bookmarkEvent.downloadEventOptional())
        .filter(downloadEventOptional -> downloadEventOptional.isPresent())
        .map(downloadEventOptional -> downloadEventOptional.get());
        .subscribe(downloadEvent -> {
            mNotificationManager.displayNotificationForDownloadEvent(downloadEvent);
        });
}
```

The `filter` operator and its following `map` operator consume an `Optional<DownloadEvent>`, and together emit only present `DownloadEvent` instances. Again, this is a common occurrence, but for different types. And so we might define a utility method like so:

```java
/**
 * @return an observable emitting the present values of {@link Optional} instances emitted by
 *         the observable parameter
 */
public static <T> Observable<T> observePresentValues(Observable<Optional<T>> observable) {
    return observable.
            .filter(optional -> optional.isPresent())
            .map(optional -> optional.get());
}
```

And then call it like so:

```java
final Observable<Optional<DownloadEvent>> downloadEventOptionalObservable =
        mBookmarkManager.getBookmarkEventObservable()
            .map(bookmarkEvent -> bookmarkEvent.downloadEventOptional())
ObservableUtils.observePresentValues(downloadEventOptionalObservable)
        .subscribe(downloadEvent -> {
            mNotificationManager.displayNotificationForDownloadEvent(downloadEvent);
        });
```

There are other ways to structure this, but none of them are pleasing. Reading RxJava is arguably most straightforward when there are no temporary variables like `downloadEventOptionalObservable`, and following the left margin from top to bottom yields all operators that constitute the subscriber chain.

To achieve these goals, we can use the `compose` method of `Observable`. This method accepts an instance that implements `Transformer<F, R>`, which transforms an instance of `Observable<F>` to an instance of `Observable<R>`. The above calls to `filter` and then `map` specify a transformation from `Observable<Optional<T>>` to `Observable<T>`. We can instead define a utility method that returns a `Transformer`:


```java
/**
 * @return a transformer for emitting the present values of emitted {@link Optional} instances
 */
public static <T> Transformer<Optional<T>, T> presentValuesTransformer() {
    return observable -> observable
            .filter(optional -> optional.isPresent())
            .map(optional -> optional.get());
}
```

And then pass it to `compose`:

```java
mBookmarkManager.getBookmarkEventObservable()
        .map(bookmarkEvent -> bookmarkEvent.downloadEventOptional())
        .compose(ObservableUtils.presentValuesTransformer())
        .subscribe(downloadEvent -> {
            mNotificationManager.displayNotificationForDownloadEvent(downloadEvent);
        });
}
```

This is much more readable than using the `observePresentValues` utility method.

Note that the `Transformer<Optional<T>, T>` instance returned has no state, and is therefore trivially immutable and safe for sharing. Moreover its sequence of `filter` and `map` operators does not rely on the parameterized type `T`. The same instance, therefore, can transform a `Observable<Optional<T>>` for any type `T`.

To implement this, we create a `Transformer<Optional<Object>, Object>` instance that is both private and static, and then modify `presentValuesTransformer` to return it:

```java
private static Transformer PRESENT_VALUES_TRANSFORMER =
        new Transformer<Optional<Object>, Object>() {
    @Override
    public Observable<Object> call(final Observable<Optional<Object>> optionalObservable) {
        return optionalObservable
                .filter(optional -> optional.isPresent())
                .map(optional -> optional.get());
    }
};

@SuppressWarnings("unchecked")
public static <T> Transformer<Optional<T>, T> presentValuesTransformer() {
    return PRESENT_VALUES_TRANSFORMER;
}
```

The compiler cannot conclude that the cast from `Transformer<Optional<Object>, Object>` to `Transformer<Optional<T>, T>` is safe, and so it generates a compilation warning. The `SuppressWarnings("unchecked")` annotation suppresses that warning.

