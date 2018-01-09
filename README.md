Hey, do have a minute for a [quick survey](https://docs.google.com/forms/d/e/1FAIpQLSePA8NqA9Jlbvh28xcFVbmIGUzHW3dnsxuxi23-ZDPPfkWMSQ/viewform) on how we are doing with EventBus? 

EventBus
========
EventBus is a publish/subscribe event bus for Android and Java.<br/>
<img src="EventBus-Publish-Subscribe.png" width="500" height="187"/>

EventBus...

 * simplifies the communication between components
    * decouples event senders and receivers
    * performs well with Activities, Fragments, and background threads
    * avoids complex and error-prone dependencies and life cycle issues
 * makes your code simpler
 * is fast
 * is tiny (~50k jar)
 * is proven in practice by apps with 100,000,000+ installs
 * has advanced features like delivery threads, subscriber priorities, etc.

 [![Build Status](https://travis-ci.org/greenrobot/EventBus.svg?branch=master)](https://travis-ci.org/greenrobot/EventBus)

EventBus in 3 steps
-------------------
1. Define events:

    ```java  
    public static class MessageEvent { /* Additional fields if needed */ }
    ```

2. Prepare subscribers:
    Declare and annotate your subscribing method, optionally specify a [thread mode](http://greenrobot.org/eventbus/documentation/delivery-threads-threadmode/):  

    ```java
    @Subscribe(threadMode = ThreadMode.MAIN)  
    public void onMessageEvent(MessageEvent event) {/* Do something */};
    ```
    Register and unregister your subscriber. For example on Android, activities and fragments should usually register according to their life cycle:

   ```java
    @Override
    public void onStart() {
        super.onStart();
        EventBus.getDefault().register(this);
    }
 
    @Override
    public void onStop() {
        super.onStop();
        EventBus.getDefault().unregister(this);
    }
    ```

3. Post events:

   ```java
    EventBus.getDefault().post(new MessageEvent());
    ```

Read the full [getting started guide](http://greenrobot.org/eventbus/documentation/how-to-get-started/).

Add EventBus to your project
----------------------------
<a href="https://search.maven.org/#search%7Cga%7C1%7Cg%3A%22org.greenrobot%22%20AND%20a%3A%22eventbus%22"><img src="https://img.shields.io/maven-central/v/org.greenrobot/eventbus.svg"></a>

Via Gradle:
```gradle
compile 'org.greenrobot:eventbus:3.1.1'
```

Via Maven:
```xml
<dependency>
    <groupId>org.greenrobot</groupId>
    <artifactId>eventbus</artifactId>
    <version>3.1.1</version>
</dependency>
```

Or download [the latest JAR](https://search.maven.org/remote_content?g=org.greenrobot&a=eventbus&v=LATEST) from Maven Central.

Homepage, Documentation, Links
------------------------------
For more details please check the [EventBus website](http://greenrobot.org/eventbus). Here are some direct links you may find useful:

[Features](http://greenrobot.org/eventbus/features/)

[Documentation](http://greenrobot.org/eventbus/documentation/)

[ProGuard](http://greenrobot.org/eventbus/documentation/proguard)

[Changelog](http://greenrobot.org/eventbus/changelog/)

[FAQ](http://greenrobot.org/eventbus/documentation/faq/)

How does EventBus compare to other solutions, like Otto from Square? Check this [comparison](COMPARISON.md).

License
-------
Copyright (C) 2012-2017 Markus Junginger, greenrobot (http://greenrobot.org)

EventBus binaries and source code can be used according to the [Apache License, Version 2.0](LICENSE).

More Open Source by greenrobot
==============================
[__ObjectBox__](http://objectbox.io/) ([GitHub](https://github.com/objectbox/objectbox-java)) is a new superfast object-oriented database for mobile.

[__Essentials__](http://greenrobot.org/essentials/) ([GitHub](https://github.com/greenrobot/essentials)) is a set of utility classes and hash functions for Android & Java projects.

[__greenDAO__](http://greenrobot.org/greendao/) ([GitHub](https://github.com/greenrobot/greenDAO)) is an ORM optimized for Android: it maps database tables to Java objects and uses code generation for optimal speed.

[Follow us on Google+](https://plus.google.com/b/114381455741141514652/+GreenrobotDe/posts) or check our [homepage](http://greenrobot.org/) to stay up to date.

#EventBus 解析一
----------------
## EventBus的方法解析

### EventBusd的构造方法
      /**
     * Creates a new EventBus instance; each instance is a separate scope in which events are delivered. To use a
     * central bus, consider {@link #getDefault()}.
     */
    public EventBus() {
        this(DEFAULT_BUILDER);
    }

    EventBus(EventBusBuilder builder) {
        subscriptionsByEventType = new HashMap<>();
        typesBySubscriber = new HashMap<>();
        stickyEvents = new ConcurrentHashMap<>();
        mainThreadPoster = new HandlerPoster(this, Looper.getMainLooper(), 10);
        backgroundPoster = new BackgroundPoster(this);
        asyncPoster = new AsyncPoster(this);
        indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;
        subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
                builder.strictMethodVerification, builder.ignoreGeneratedIndex);
        logSubscriberExceptions = builder.logSubscriberExceptions;
        logNoSubscriberMessages = builder.logNoSubscriberMessages;
        sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
        sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
        throwSubscriberException = builder.throwSubscriberException;
        eventInheritance = builder.eventInheritance;
        executorService = builder.executorService;
    }

这个方法有太多的位置变量了，我们先从一个简单的方法入门，之后再对这些方法进行拆解


### EventBus的订阅方法 `EventBus.register(Object subscriber)`
      /**
     * Registers the given subscriber to receive events. Subscribers must call {@link #unregister(Object)} once they
     * are no longer interested in receiving events.
     * <p/>
     * Subscribers have event handling methods that must be annotated by {@link Subscribe}.
     * The {@link Subscribe} annotation also allows configuration like {@link
     * ThreadMode} and priority.
     */
    public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }

首先翻译下这个方法的注释： 注册一个订阅者来接收事件。 一旦一个订阅者不需要接收这个方法，就要使用`unregister(Object)` 方法进行解除订阅。订阅者的事件处理方法必须声明`Subscribe`,同时 方法的声明也支持 `ThreadMode` `Stick` `priority` 等方法声明
