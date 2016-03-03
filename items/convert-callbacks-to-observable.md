### Convert callbacks to `Observable`

Many Java libraries define callback interfaces for operations that are very `Observer`-like:

* If the operation succeeded, an `onSuccess` or similar method provides a value.
* If the operation failed, an `onError` or similar method provides a `Throwable` specifying the error.

By converting such callbacks to `Observable` instances, we can apply operators to the emitted value.

As an example of such an `Observer`-like interface consider the [`ManifestCallback`](http://google.github.io/ExoPlayer/doc/reference/com/google/android/exoplayer/util/ManifestFetcher.ManifestCallback.html) interface of class `ManifestFetcher` in [ExoPlayer](http://google.github.io/ExoPlayer/):

```java
/**
 * @param <T> The type of manifest.
 */
public interface ManifestCallback<T> {
  /**
   * @param manifest the successfully loaded manifest
   */
  void onSingleManifest(T manifest);

  /**
   * Invoked when the load has failed.
   *
   * @param e the cause of the failure for loading the manifest
   */
  void onSingleManifestError(IOException e);
}
```

In the [Khan Academy Android application](https://play.google.com/store/apps/details?id=org.khanacademy.android), we use ExoPlayer to play videos. When the client has an Internet connection, the video player streams videos using [HLS](https://en.wikipedia.org/wiki/HTTP_Live_Streaming). This allows the video player to stream the video at a quality that is commensurate with its available bandwidth. The URLs for the video streams of different qualities are specified by a `HlsPlaylist` instance. The client must therefore first download this `HlsPlaylist` from the server.

Given the `Uri` instance of the playlist, we can retrieve it by constructing and invoking `singleLoad` on a `ManifestFetcher` instance parameterized with type `HlsPlaylist`:

```java
final UriDataSource dataSource = new DefaultUriDataSource(mContext, mUserAgent);
final HlsPlaylistParser parser = new HlsPlaylistParser();
final ManifestFetcher<HlsPlaylist> playlistFetcher = new ManifestFetcher<>(
        uri.toString(), dataSource, parser);

playlistFetcher.singleLoad(mMainHandler.getLooper(), new ManifestCallback<HlsPlaylist>() {
    @Override
    public void onSingleManifest(final HlsPlaylist manifest) {
        // code to configure ExoPlayer with the given playlist goes here ...
    }

    @Override
    public void onSingleManifestError(final IOException e) {
        // code to recover from the error goes here ...
    }
});
```

When `singleLoad` either succeeds loading or fails to load the `HlsPlaylist` from the server, it invokes the corresponding method of its `ManifestCallback` parameter, which is run on a `Looper` instance associated with the main thread.

To convert this to an `Observable`, we use the `create` static factory method:

```java
final Observable<HlsPlaylist> playlistObservable = Observable.create(subscriber -> {
    final UriDataSource dataSource = new DefaultUriDataSource(mContext, mUserAgent);
    final HlsPlaylistParser parser = new HlsPlaylistParser();
    final ManifestFetcher<HlsPlaylist> playlistFetcher = new ManifestFetcher<>(
            uri.toString(), dataSource, parser);

    playlistFetcher.singleLoad(mMainHandler.getLooper(), new ManifestCallback<HlsPlaylist>() {
        @Override
        public void onSingleManifest(final HlsPlaylist manifest) {
            if (!subscriber.isUnsubscribed()) {
                subscriber.onNext(manifest);
                subscriber.onCompleted();
            }
        }

        @Override
        public void onSingleManifestError(final IOException e) {
            if (!subscriber.isUnsubscribed()) {
                subscriber.onError(e);
            }
        }
    });
});
// subscribing to playlistObservable and configuring ExoPlayer follows here ...
```

Whenever a new subscription to the `Observable` is created, the code that was passed to the `create` method is invoked. This is a method that accepts as a parameter a [`Subscriber`](http://reactivex.io/RxJava/javadoc/rx/Subscriber.html) instance belonging to the subscription.

When the method passed to `create` returns, no methods belonging to the `Subscriber` parameter have yet been invoked. The fetching of the `HlsPlaylist` has begun, however, and when the `ManifestFetcher` succeeds or fails to load the `HlsPlaylist`, it again invokes the corresponding method on its handler:

* If the operation succeeded, `onSingleManifest` is invoked with the playlist. This emits the `HlsPlaylist` instance as a single value to the subscriber using `onNext`, and then completes the subscription using `onCompleted`.
* If the operation failed, an `onSingleManifestError` is invoked with the exception describing the failure. This forwards the `IOException` to the subscriber using `onError`, thereby terminating the subscription.

Note that if the client retains its `Subscription` instance, then it may call `unsubscribe` while the `HlsPlaylist` is still being fetched, and consequently before either method `onSingleManifest` or `onSingleManifestError` is invoked. Therefore both methods guard against the client unsubscribing. Note that unsubscribing does not cancel the work of fetching the `HlsPlaylist`, but instead, simply ignores the outcome.

