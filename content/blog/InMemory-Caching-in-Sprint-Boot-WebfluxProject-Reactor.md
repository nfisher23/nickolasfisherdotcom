---
title: "In-Memory Caching in Sprint Boot Webflux/Project Reactor"
date: 2020-10-03T22:41:59
draft: false
tags: [java, spring, reactive, webflux]
---

Sample code for this article [can be found on Github](https://github.com/nfisher23/reactive-programming-webflux/tree/master/api-calls-and-resilience).

In memory caching can significantly improve performance in a microservices environment, usually because of the tail latency involved in calling downstream services. Caching can also _help_ with resilience, though the extent to which that matters will depend on how you're actually leveraging that caching. There are two flavors of caching that you're like to want to use, the first is using the Mono as a hot source \[which is demonstrated here\], and the second would be when you want to [selectively cache individual key/value pairs](https://nickolasfisher.com/blog/how-to-use-caffeine-caches-effectively-in-spring-boot-webflux).

Caching in reactor when using a Mono as a hot source is pretty straightforward, but there is one gotcha. I will demonstrate how to cache, and also how to write automated tests for that caching behavior, in this post.

## The App

We're going to leverage some work done in a previous post here. Let's recall that we had a service that would retry on any timeout downstream:

```java
@Service
public class RetryService {
    private final WebClient serviceAWebClient;

    public RetryService(@Qualifier("service-a-web-client") WebClient serviceAWebClient) {
        this.serviceAWebClient = serviceAWebClient;
    }

    public Mono<WelcomeMessage> getWelcomeMessageAndHandleTimeout(String locale) {
        return this.serviceAWebClient.get()
                .uri(uriBuilder -> uriBuilder.path("/locale/{locale}/message").build(locale))
                .retrieve()
                .bodyToMono(WelcomeMessage.class)
                .retryWhen(
                    Retry.backoff(2, Duration.ofMillis(25))
                            .filter(throwable -> throwable instanceof TimeoutException)
                );
    }
}

```

Let's say for the sake of this article that the business requirements allow us to cache a welcome message that is in English for about five minutes. There is no hard requirement on welcome messages being updated immediately. Now that we already have resilience around timeouts, let's also add a cache on a successful response by creating a decorator. Because this is a tutorial, we're going to call that decorator service **CachingService**:

```java
@Service
public class CachingService {

    private final RetryService retryService;

    public CachingService(RetryService retryService) {
        this.retryService = retryService;
    }

    public Mono<WelcomeMessage> getEnglishLocaleWelcomeMessage() {
        return this.retryService.getWelcomeMessageAndHandleTimeout("en_US");
    }
}

```

This service currently just proxies directly to the **RetryService** and doesn't do anything remarkable. We're now going to add a test that verifies that the cache is working:

```java
public class CachingServiceIT {

    @Test
    public void englishLocaleWelcomMessage_caches() {
        RetryService mockRetryService = Mockito.mock(RetryService.class);

        AtomicInteger counter = new AtomicInteger();
        Mockito.when(mockRetryService.getWelcomeMessageAndHandleTimeout("en_US"))
                .thenReturn(Mono.defer(() ->
                            Mono.just(new WelcomeMessage("count " + counter.incrementAndGet()))
                        )
                );

        CachingService cachingService = new CachingService(mockRetryService);

        StepVerifier.create(cachingService.getEnglishLocaleWelcomeMessage())
                .expectNextMatches(welcomeMessage -> "count 1".equals(welcomeMessage.getMessage()))
                .verifyComplete();

        StepVerifier.create(cachingService.getEnglishLocaleWelcomeMessage())
                .expectNextMatches(welcomeMessage -> "count 1".equals(welcomeMessage.getMessage()))
                .verifyComplete();
    }
}

```

This test defers to a **Supplier** to return a **Mono** on demand. Every time this mono is subscribed to, it will call our **Supplier**. Under the covers, our **Supplier** will return basically just a count of the number of times it has been invoked.

One way to get this test to pass is to just call one of the **cache** methods on **Mono**. This is one of the easiest:

```java
@Service
public class CachingService {

    private final RetryService retryService;
    private final Mono<WelcomeMessage> cachedEnglishWelcomeMono;

    public CachingService(RetryService retryService) {
        this.retryService = retryService;
        this.cachedEnglishWelcomeMono = this.fallbackService.getWelcomeMessageAndHandleTimeout("en_US")
                .cache(Duration.ofMinutes(5));
    }

    public Mono<WelcomeMessage> getEnglishLocaleWelcomeMessage() {
        return cachedEnglishWelcomeMono;
    }
}

```

Now the test passes, but there is a fatal flaw with this approach: **it also caches errors**. So, if the **RetryService** ends up timing out too many times, or fails for some other reason, then we will have our **CachingService** constantly emitting that error for five minutes. **This is almost always bad** and I wish this wasn't the interface, but it is.

To demonstrate this in action, I'll write another test to target that:

```java
    @Test
    public void cachesSuccessOnly() {
        RetryService mockRetryService = Mockito.mock(RetryService.class);

        AtomicInteger counter = new AtomicInteger();
        Mockito.when(mockRetryService.getWelcomeMessageAndHandleTimeout("en_US"))
                .thenReturn(Mono.defer(() -> {
                            if (counter.incrementAndGet() > 1) {
                                return Mono.just(new WelcomeMessage("count " + counter.get()));
                            } else {
                                return Mono.error(new RuntimeException());
                            }
                        })
                );

        CachingService cachingService = new CachingService(mockRetryService);

        StepVerifier.create(cachingService.getEnglishLocaleWelcomeMessage())
                .expectError()
                .verify();

        StepVerifier.create(cachingService.getEnglishLocaleWelcomeMessage())
                .expectNextMatches(welcomeMessage -> "count 2".equals(welcomeMessage.getMessage()))
                .verifyComplete();

        // previous result should be cached
        StepVerifier.create(cachingService.getEnglishLocaleWelcomeMessage())
                .expectNextMatches(welcomeMessage -> "count 2".equals(welcomeMessage.getMessage()))
                .verifyComplete();
    }

```

You will see this fail, because it will cache the error and keep emitting it. To fix that, there is a golden **cache** override on our Mono that allows us to specify the duration of which each type of response is cached:

```java
@Service
public class CachingService {

    private final RetryService retryService;
    private final Mono<WelcomeMessage> cachedEnglishWelcomeMono;

    public CachingService(RetryService retryService) {
        this.retryService = retryService;
        this.cachedEnglishWelcomeMono = this.retryService.getWelcomeMessageAndHandleTimeout("en_US")
                .cache(welcomeMessage -> Duration.ofMinutes(5),
                        throwable -> Duration.ZERO,
                        () -> Duration.ZERO
                );
    }

    public Mono<WelcomeMessage> getEnglishLocaleWelcomeMessage() {
        return cachedEnglishWelcomeMono;
    }
}

```

Now if we run our tests again, both of them pass. By passing in **Duration.ZERO**, we don't cache errors or empty Monos at all, only the successful one, for five minutes.

Remember to checkout the [source code on Github](https://github.com/nfisher23/reactive-programming-webflux/tree/master/api-calls-and-resilience).
