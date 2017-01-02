### Convert functions to `Observable`

From the Javadoc for `Observable`, it is not immediately clear how to create an `Observable` that invokes a given `Func0` upon subscription. We can accomplish this by combining its `defer` and `just` operators:

```java
/**
 * @return an {@link Observable} that emits invokes {@code function} upon subscription and emits
 *         its value
 */
public static <O> Observable<O> makeObservable(final Func0<O> function) {
    checkNotNull(function);

    return Observable.defer(() -> Observable.just(function.call()));
}
```

Sometimes this method is useful by itself:

```java
mVideoDownloadManager.getDownloadedJsonTranscriptUri(videoItemId)
    .flatMap(downloadedJsonTranscriptUriOptional -> {
        if (downloadedJsonTranscriptUriOptional.isPresent()) {
            // Read the transcript from the downloaded file.
            return ObservableUtils.makeObservable(() -> {
                final URI uri = downloadedJsonTranscriptUriOptional.get();
                final BufferedReader reader = FileUtil.newUtf8Reader(new File(uri));
                return VideoSubtitleSequenceJsonDecoder.read(new JsonReader(reader));
            })
                    .subscribeOn(Schedulers.io());
        } else {
            // Return the transcript from the network using Retrofit.
            return mContentApi.downloadVideoSubtitles(youTubeId);
        }
    })
    // more operators follow here ...
```

In many cases, we have found this method useful when defining a class that wraps another class and converts its public API to expose an `Observable` for each `public` method. For example, we have a `ContentDatabase` in the [Khan Academy Android app](https://play.google.com/store/apps/details?id=org.khanacademy.android) through which the client can load content, such as videos and articles and their parent topics (e.g. "math" and "3rd grade math"), from a database on the device. Its methods are synchronous:

```java
public interface ContentDatabase extends Closeable {
    Optional<Topic> fetchTopic(String topicSlug);

    Optional<ContentItem> fetchContentItem(ContentItemIdentifier identifier);

    // more methods follow here ...
}
```

The synchronous API of `ContentDatabase` makes testing an implementation easy:

```java
@Test
public void testFetchMissingContentItem() {
    final ContentItemIdentifier missingContentItemId = TestUtil.randomContentItemId();
    final Optional<ContentItem> contentItemOptional =
        mContentDatabase.fetchContentItem(missingContentItemId);
    assertEquals(Optional.absent(), contentItemOptional);
}
```

But our application should not invoke methods on `ContentDatabase` directly, as the main thread will block upon reading from the database. We therefore introduce an `ObservableContentDatabase` class that converts the return type of each method of `ContentDatabase` to an `Observable`. A `ObservableContentDatabase` instance delegates to its underlying `ContentDatabase` instance, but invokes each method on a given `Scheduler` so that all reads from the database happen off the main thread:

```java
public class ObservableContentDatabase implements Closeable {
    private final ContentDatabase mContentDatabase;
    private final Scheduler mScheduler;

    public ObservableContentDatabase(final ContentDatabase contentDatabase) {
        this(contentDatabase, SchedulerUtils.newSingleThreadIoScheduler());
    }
    public ObservableContentDatabase(final ContentDatabase contentDatabase,
                                     final Scheduler scheduler) {
        mContentDatabase = checkNotNull(contentDatabase);
        mScheduler = checkNotNull(scheduler);
    }

    public Observable<Optional<Topic>> fetchTopic(final String topicSlug) {
        return subscribeOnScheduler(() -> mContentDatabase.fetchTopic(topicSlug));
    }
    public Observable<Optional<ContentItem>> fetchContentItem(
            final ContentItemIdentifier contentItemId) {
        return subscribeOnScheduler(() -> mContentDatabase.fetchContentItem(contentItemId));
    }
    // more delegating methods follow here ...
    
    private <T> Observable<T> subscribeOnScheduler(final Func0<T> function) {
        return ObservableUtils.makeObservable(function)
                .subscribeOn(mScheduler);
    }
}
```

Each of the `public` methods that return an `Observable` uses the `private` method `subscribeOnScheduler` to create that `Observable`. That method accepts a `Func0` parameter specifying the work to execute on the `Scheduler` instance `mScheduler`. It uses our `makeObservable` method to convert that work into an `Observable`, and uses the `subscribeOn` operator to ensure that it is executed on the `Scheduler` instance `mScheduler`.

It is up to a client, upon subscribing to the `Observable` instances from `ObservableContentDatabase`, to invoke `observeOn(AndroidSchedulers.mainThread())` if necessary.

