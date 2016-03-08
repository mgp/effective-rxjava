### Test emitted values using `TestObserver`

To test a chain of `Observable` instances within a component, attempt to:

* inject the root `Observable` instance into the component
* observe the emitted values through a public or package-private `Observable` instance that is downstream

You can then inject a `PublishSubject` into the component, thereby allowing you to control the emission of events by the root `Observable`. Subscribing to the public or package-private `Observable` instance using a `TestObserver` lets you assert that it emits the expected values.

If you cannot inject the root `Observable` instance into the component, or if the component is too difficult to get under test, then consider two options:

* Attempt to create a static, package-private method that encapsulates creating the chain of `Observable` instances. Your component can simply delegate to this method to create the chain and subscribe to the returned `Observable`. Similarly, your test can invoke this method to create the chain, and then use a `TestObserver` to assert the behavior of the returned `Observable`.
* Attempt to move the creation of the `Observable` instance and any related functionality into a smaller component that allows injecting the root `Observable` and is easier to get under test. Again, your component can simply delegate to this smaller component, and your test can depend only on this smaller component instead of the larger one.

Below we demonstrate an example of the second strategy from the [Khan Academy Android application](https://play.google.com/store/apps/details?id=org.khanacademy.android).

#### Using `TestObserver`

A user of the Khan Academy Android application can download videos for offline viewing. As a video downloads, an `Observable` emits `DownloadEvent` instances that reflect the progress of the download. We want to observe these events so that we can display and update a notification for the download:

[image]

This notification contains the title of the downloading video. The title is a property of the `ContentItem` that represents the video. However, the `DownloadEvent` that we receive has only a `ContentItemIdentifier` property, which is only a unique identifier for a `ContentItem` instance. We can pass the `ContentItemIdentifier` to the `fetchContentItem` method of a `ContentDatabase` implementation:

```java
interface ContentDatabase {
    Observable<ContentItem> fetchContentItem(ContentItemIdentifier contentItemId);
}
```

Subscribing to the returned `Observable` will fetch the corresponding `ContentItem` from the database and emit it. We now define a `ContentDownloadEvent` class that pairs a `DownloadEvent` with its corresponding `ContentItem`:

```java
@AutoValue
public abstract class ContentDownloadEvent {
    public abstract DownloadEvent downloadEvent();
    public abstract ContentItem contentItem();

    public static ContentDownloadEvent create(final DownloadEvent downloadEvent,
                                              final ContentItem contentItem) {
        return new AutoValue_ContentDownloadEvent(downloadEvent, contentItem);
    }
}
```

And now, by combining these elements, we can create an `Observable` that emits a `ContentDownloadEvent` for each emitted `DownloadEvent`:

```java
public Observable<ContentDownloadEvent> getContentDownloadEventObservable() {
    return mDownloadEventObservable.flatMap(downloadEvent -> {
        final ContentItemIdentifier contentItemId = downloadEvent.contentItemIdentifier();
        return mContentDatabase.fetchContentItem(contentItemId)
                .map(fetchedContentItem -> {
                    return ContentDownloadEvent.create(downloadEvent, fetchedContentItem)
                });
    });
}
```

Above, `mDownloadEventObservable` is the `Observable` that emits `DownloadEvent` instances. And `mContentDatabase` is the `ContentDatabase` implementation.

However, the above method is inefficient: Over the course of downloading a large video, thousands of `DownloadEvent` instances may be emitted with the same `ContentItemIdentifier`. Fetching the same `ContentItem` thousands of times from the `ContentDatabase` will both drain the user's battery and cause poor performance.

To remedy this, we create a new class named `ContentDownloadEventCache` and move method `getContentDownloadEventObservable` into it. It is constructed with the same `Observable<DownloadEvent>` and `ContentDatabase` instances. But internally, it does not necessarily need to fetch from the `ContentDatabase` the `ContentItem` associated with every `DownloadEvent`. Instead, it maintains a `Map` of cached `ContentItem` instances. This map has a maximum size and an LRU eviction policy:

```java
public Observable<ContentDownloadEvent> getContentDownloadEventObservable() {
    return mDownloadEventObservable.flatMap(downloadEvent -> {
        final ContentItemIdentifier contentItemId = downloadEvent.contentItemIdentifier();
        final @Nullable ContentItem cachedContentItem = mCachedContentItemsMap.get(contentItemId));
        if (cachedContentItem != null) {
            return Observable.just(ContentDownloadEvent.create(downloadEvent, cachedContentItem));
        } else {
            return mContentDatabase.fetchContentItem(contentItemId)
                    .observeOn(mScheduler)
                    .doOnNext(fetchedContentItem -> {
                        mCachedContentItemsMap.put(contentItemId, fetchedContentItem);
                    })
                    .map(fetchedContentItem -> {
                        return ContentDownloadEvent.create(downloadEvent, fetchedContentItem)
                    });
        }
    });
}
```

Note that the `Observable` returned by method `fetchContentItem` may perform its work on another `Scheduler` instance like `Schedulers.io()`. To keep this code thread-safe, we must transition back to the main thread before adding the `ContentItem` to the `Map` in the `doOnNext` action. We do this by calling `observeOn(mScheduler)`, where `mScheduler` is a `Scheduler` that is backed by the main thread and passed into the constructor. (In the Android application, we specify it as `AndroidSchedulers.mainThread()`.) See the item [Understand `subscribeOn` and `observeOn`](understand-subscribeon-and-observeon.md) for more details on the use of `observeOn` and its related method `subscribeOn`.

With this in place, only the first `DownloadEvent` for a download will fetch the `ContentItem` from the `ContentDatabase`. All subsequent `DownloadEvent` instances will use the cached `ContentItem` to construct the corresponding `ContentDownloadEvent`. We now want to test this behavior.

##### Test setup

In the setup for our test, we create a `PublishSubject` on which we can manually emit `DownloadEvent` instances. Also, using [Mockito](http://mockito.org/), we create a `ContentDatabase` implementation whose `fetchContentItem` method will return the `ContentItem` for a video, provided its `ContentItemIdentifier` is specified as a parameter:

```java
final ContentItem videoItem = TestUtil.randomVideoItem();
final ContentItemIdentifier videoItemId = videoItem.contentItemIdentifier();

final PublishSubject<DownloadEvent> downloadEventSubject = PublishSubject.create();
final ContentDatabase contentDatabase = mock(ContentDatabase.class);
when(contentDatabase.fetchContentItem(eq(videoItemId))).thenReturn(Observable.just(videoItem));
```

From these values we create the `ContentDownloadEventCache` instance. We also create a `TestObserver` that subscribes to the `Observable` returned by its `getContentDownloadEventObservable` method. This will allow us to assert that we observe the expected values:

```java
final ContentDownloadEventCache contentDownloadEventCache = new ContentDownloadEventCache(
        downloadEventSubject, contentDatabase, Schedulers.immediate()
);

final TestObserver<ContentDownloadEvent> testObserver = new TestObserver<>();
testObserver.subscribe(contentDownloadEventCache.getContentDownloadEventObservable());
```

Now that our setup is complete, we begin making assertions.

##### Test execution

We can emit a `DownloadEvent` instance by passing it to the `onNext` method of the `PublishSubject`. Then, using the `TestObserver` that is subscribed to the `Observable` returned by method `getContentDownloadEventObservable`, we can assert that the expected `ContentDownloadEvent` instances are emitted:

```java
final DownloadEvent addedDownloadEvent = createAddedDownloadEvent(videoItemId);
final DownloadEvent receivedDataDownloadEvent = createReceivedDataDownloadEvent(videoItemId);
downloadEventSubject.onNext(addedDownloadEvent);
downloadEventSubject.onNext(receivedDataDownloadEvent);

final List<ContentDownloadEvent> expectedContentDownloadEvents = ImmutableList.of(
        ContentDownloadEvent.create(addedDownloadEvent, videoItem),
        ContentDownloadEvent.create(receivedDataDownloadEvent, videoItem)
);
testObserver.assertReceivedOnNext(expectedContentDownloadEvents);
```

Above, method `assertReceivedOnNext` of `TestObserver` asserts that it observed the given sequence of `ContentDownloadEvent` values. Because `ContentDownloadEvent` is [defined as a value type using AutoValue](https://github.com/mgp/effective-rxjava/blob/master/items/emit-immutable-values.md#use-autovalue), we can rely on this method to test the equality of the expected and actual sequences.

Moreover, using Mockito we can ensure that the `fetchContentItem` method of `ContentDatabase` was invoked only once. This implies that when `receivedDataDownloadEvent` was emitted, the `ContentItem` was read from the cache instead of fetched from the `ContentDatabase` again:

```java
final InOrder inOrder = inOrder(contentDatabase);
inOrder.verify(contentDatabase).fetchContentItem(videoItemId);
inOrder.verifyNoMoreInteractions();
```

Other tests can verify the eviction of the eldest entries in the `Map` when it reaches capacity, and so on.

