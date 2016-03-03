### Use Retrolambda

[Retrolambda](https://github.com/orfjackal/retrolambda) enables lambda expressions and method references on Android by transforming Java 8 bytecode to bytecode for Java 7 and lower. While this may sound scary, lambda expressions and method references remove the syntactic boilerplate from the anonymous classes you must define when providing `Func` and `Action` implementations, thereby greatly improving your code's readability. Additionally, the [Gradle plugin](https://github.com/evant/gradle-retrolambda) makes integrating Retrolambda with your build seamless.

Without Retrolambda, your RxJava code will look like:

```java
mContentDatabase
    .fetchContentItems(ImmutableSet.of(contentItemId))
    .map(new Func1<Map<ContentItemIdentifier, ContentItem>, ContentItem>() {
        @Override
        public ContentItem call(final Map<ContentItemIdentifier, ContentItem> resultSet) {
            return checkNotNull(resultSet.get(contentItemId));
        }
    })
    .map(new Func1<ContentItem, TopicPath>() {
        @Override
        public TopicPath call(final ContentItem contentItem) {
            return contentItem.topicPath();
        }
    })
    .subscribe(new Action1<TopicPath>() {
        @Override
        public void call(final TopicPath topicPath) {
            openContentViewActivity(contentItemId, topicPath);
        }
    });
```

With Retrolambda, your RxJava code will look like:

```java
mContentDatabase
    .fetchContentItems(ImmutableSet.of(contentItemId))
    .map(resultSet -> checkNotNull(resultSet.get(contentItemId)))
    .map(ContentItem::topicPath)
    .subscribe(topicPath -> {
        openContentViewActivity(contentItemId, topicPath);
    });
```

We at [Khan Academy](https://www.khanacademy.org/) have been using it in our [Android application](https://play.google.com/store/apps/details?id=org.khanacademy.android) with no issues. Many other companies use it with their own applications. Use it with your own application if you are using RxJava.

