### Emit immutable values

The [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa/) project, which is an Functional Reactive Programming implementation for iOS and OS X, touts itself as "streams of values over time." We propose that you go one step further, and to emit "streams of *immutable values* over time" whenever possible. To quote from the [Android Development Best Practices guide](https://github.com/Khan/style-guides/blob/master/style/android-development-best-practices.md) for [Khan Academy](https://www.khanacademy.org/):

> Our principal goal as programmers is to reduce complexity. At any point in a program, we can reason about the state of a constant trivially -- its state is the state upon assignment. By contrast, the state of a variable can change endlessly.
> 
> With judicious use, immutable objects can lead to quicker execution speed and lower memory usage. The hash value of a `String`, the query parameters of an immutable URL, and the distance traced by an immutable sequence of points are all immutable as well. We can cache their values for later use instead of recompute them on every access.
> 
> An immutable value is safe in many ways. We can share an immutable value with clients without worry that the client will silently mutate the value. Similarly, a client can accept an immutable value as a parameter without needing to defensively copy it. An immutable value is also thread-safe, because a client must acquire a lock only to read data that another client can concurrently modify. An immutable value renders that moot.

To summarize: Immutable values help tame complexity. In RxJava, a data source with an `Observable` should emit immutable values that reflect its current state. When the state of the data source is updated, it should simply emit a new immutable value that reflects the updated state.

If you are developing an Android application, and you are using [ProGuard](http://proguard.sourceforge.net/) to eliminate dead code from your application, then consider using AutoValue and Guava from Google to help construct immutable values. (AutoValue includes Guava as a dependency, and Guava is a large library. So dead code elimination via ProGuard is ideal.)

#### Use AutoValue

[AutoValue](https://github.com/google/auto/tree/master/value) creates immutable value types with very little effort required on your part. From its documentation:

> Classes with value semantics are extremely common in Java. These are classes for which object identity is irrelevant, and instances are considered interchangeable based only on the equality of their field values... AutoValue provides an easier way to create immutable value types, with less code and less room for error.

Instances of immutable value types defined by AutoValue are simply data. They have no behavior. This is ideal because such added behavior will seek to mutate the associated value type instance, and we want to avoid this.

To use AutoValue, create an abstract class with an abstract getter for each desired field. For example, our `ArticleViewFragment` for viewing articles defines a static inner class named `ToolbarViewData` that is annotated with `@AutoValue`:

```java
@AutoValue
static abstract class ToolbarViewData {
    abstract Article article();
    abstract ContentItemThumbnailData thumbnailData();
}
```

Upon compiling, the annotation processor for AutoValue creates a concrete subclass of `ToolbarViewData` named `AutoValue_ArticleViewFragment_ToolbarViewData`. This class has a constructor that is initialized with an `Article` and `ContentItemThumbnailData` instance. These values are assigned to private and `final` instance fields, which the implemented `article` and `thumbnailData` getters return:

```java
@Override
Article article() {
  return article;
}

@Override
ContentItemThumbnailData thumbnailData() {
  return thumbnailData;
}
```

The class also implements `equals` and `hashCode`, making such values suitable for use in sets, or as keys in maps:

```java
@Override
public boolean equals(Object o) {
    if (o == this) {
      return true;
    }
    if (o instanceof ArticleViewFragment.ToolbarViewData) {
        ArticleViewFragment.ToolbarViewData that = (ArticleViewFragment.ToolbarViewData) o;
        return (this.article.equals(that.article()))
                && (this.thumbnailData.equals(that.thumbnailData()));
    }
    return false;
}

@Override
public int hashCode() {
    int h = 1;
    h *= 1000003;
    h ^= article.hashCode();
    h *= 1000003;
    h ^= thumbnailData.hashCode();
    return h;
}
```

Moreover, it also implements the `toString` method, which facilitates with logging such values:

```java
@Override
public String toString() {
    return "ToolbarViewData{"
            + "article=" + article + ", "
            + "thumbnailData=" + thumbnailData
            + "}";
}
```

Because `AutoValue_ArticleViewFragment_ToolbarViewData` is not a convenient name, convention is to add a static factory method named `create` to each class annotated with `@AutoValue`. This method instantiates and returns the corresponding AutoValue implementation. For example, `ToolbarViewData` defines:

```java
public static ToolbarViewData create(final Article article,
                                     final ContentItemThumbnailData thumbnailData) {
    return new AutoValue_ArticleViewFragment_ToolbarViewData(article, thumbnailData);
}
```

A client can now simply call static factory method `create` of `ToolbarViewData` to create such a `AutoValue_ArticleViewFragment_ToolbarViewData` instance.

Other conveniences from AutoValue include:

* The generated class ensures that each constructor parameter is not `null`, unless the corresponding getter in the abstract base class has the `@Nullable` annotation.
* A builder is automatically generated through the use of the `@AutoValue.Builder` annotation.
* The generated class is serializable if the abstract base class implements `Serializable`.

See the [AutoValue documentation](https://github.com/google/auto/tree/master/value) for more details.

If you are developing for Android, consider integrating the [`Parcelable` extension for AutoValue](https://github.com/rharter/auto-value-parcel). As its name suggests, it generates value types that implement the `Parcelable` interface. This allows you to easily persist instances in a `Bundle`, which may be useful when specifying `Intent` parameters or saving the state of an activity or fragment.

#### Use Guava

[Guava](https://github.com/google/guava) contains several of Google's core libraries for its Java based projects. Most notably it includes [immutable collection implementations](https://github.com/google/guava/wiki/ImmutableCollectionsExplained), thereby allowing you to construct immutable `List`, `Map`, `Set`, `SortedMap`, and `SortedSet` instances. This is possible because the mutating methods of each collection interface, such as `add`, `set`, or `remove`, are defined as optional operations. The underlying implementation can choose to implement the method as specified by the interface, or throw an `UnsupportedOperationException`. Guava chooses the latter.

If an `Observable` must emit a collection, use Guava to construct and then emit an immutable copy of the backing collection. For example:

```java
private void emitUpdatedFiles() {
    final Set<File> updatedFiles = ImmutableSet.copyOf(mDownloadsByFile.keySet());
    mDownloadsSubject.onNext(updatedDownloads);
}
```

If entries are later added to or removed from `mDownloadsByFile`, those changes are not reflected in the emitted `Set` because an immutable copy was made.

These classes are also useful for ensuring that AutoValue instances are constructed with immutable collections, and thus the AutoValue instance itself is also immutable. For example:

```java
@AutoValue
public abstract class ProcessedConversionsError {
    public abstract List<String> failedConversionNames();
    public abstract List<String> invalidConversionNames();

    public static ProcessedConversionsError create(
            List<String> failedConversionNames,
            List<String> invalidConversionNames) {
        return new AutoValue_ProcessedConversionsError(
                ImmutableList.copyOf(failedConversionNames),
                ImmutableList.copyOf(invalidConversionNames)
        );
    }
}
```

If the caller mutates the `parameters` argument that was passed into the constructor, those changes are not reflected in the `ProcessedConversionsError` instance, again because an immutable copy was made.

Finally, note that if the value passed into a `copyOf` method is itself an instance of the immutable collection, then the `copyOf` method simply returns its parameter. See the [Guava documentation on immutable collections](https://github.com/google/guava/wiki/ImmutableCollectionsExplained) for more details.

#### Other sources of immutability

Well written third party libraries often have immutable abstractions that are ripe for reuse. For example, if you are using [OkHttp](http://square.github.io/okhttp/), then you can use its `HttpUrl` class to represent an immutable URL. Moreover, OkHttp includes [Okio](https://github.com/square/okio) as a dependency, which has a `ByteString` class that you can use to represent an immutable sequence of bytes.

Peruse the API documentation of third party libraries that you use, and identify immutable value types that you can leverage.

