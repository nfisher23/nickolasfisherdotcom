---
title: "How to Have a Fallback on Errors Calling Downstream Services in Spring Boot Webflux"
date: 2020-09-26T16:04:14
draft: false
tags: [java, spring, reactive, webflux]
---

The source code for this post is [available on Github](https://github.com/nfisher23/reactive-programming-webflux/tree/master/api-calls-and-resilience).

Things break. When you start adding more and more microservices, things will break a lot more. This post is about how to provide a degraded experience to your users when things break.

I&#39;m going to build off of some of the boilerplate code [written in previous blog posts](https://nickolasfisher.com/blog/How-to-Make-Sequential-API-Calls-and-Merge-the-Results-In-Spring-Boot-Webflux). If you&#39;ll recall, we had a **WebClient** configured like so:

```java
@Configuration
public class Config {

    @Bean(&#34;service-a-web-client&#34;)
    public WebClient serviceAWebClient() {
        HttpClient httpClient = HttpClient.create().tcpConfiguration(tcpClient -&gt;
                tcpClient.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 1000)
                        .doOnConnected(connection -&gt; connection.addHandlerLast(new ReadTimeoutHandler(1000, TimeUnit.MILLISECONDS)))
        );

        return WebClient.builder()
                .baseUrl(&#34;http://your-base-url.com&#34;)
                .clientConnector(new ReactorClientHttpConnector(httpClient))
                .build();
    }
}

```

\[You will obviously want to replace **http://your-base-url.com** with your actual url\]

And with that in place, let&#39;s create the boilerplate for a java service that will wrap a call to an external client, one that sometimes behaves badly and fails:

```java
@Service
public class FallbackService {

    private final WebClient serviceAWebClient;

    public FallbackService(@Qualifier(&#34;service-a-web-client&#34;)
                                   WebClient serviceAWebClient) {
        this.serviceAWebClient = serviceAWebClient;
    }

    public Mono&lt;WelcomeMessage&gt; getWelcomeMessageByLocale(String locale) {
        return Mono.empty();
    }
}

```

This method will allow us to get a **WelcomeMessage** that is locale specific, so that somebody who wants to receive a welcome message in french \[because, say, that&#39;s the only language they speak\] can do so. That DTO looks like this:

```java
public class WelcomeMessage {
    private String message;

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }
}

```

Following TDD, we will now create the test class:

```java
@ExtendWith(MockServerExtension.class)
public class FallbackServiceIT {

    private WebClient webClient;
    private ClientAndServer clientAndServer;

    private FallbackService fallbackService;

    public FallbackServiceIT(ClientAndServer clientAndServer) {
        this.clientAndServer = clientAndServer;
        this.webClient = WebClient.builder()
                .baseUrl(&#34;http://localhost:&#34; &#43; clientAndServer.getPort())
                .build();
    }

    @BeforeEach
    public void setup() {
        fallbackService = new FallbackService(webClient);
    }

    @AfterEach
    public void reset() {
        clientAndServer.reset();
    }

    @Test
    public void welcomeMessage_worksWhenNoErrors() {
        this.clientAndServer.when(
                request()
                        .withPath(&#34;/locale/en_US/message&#34;)
                        .withMethod(HttpMethod.GET.name())
        ).respond(
                HttpResponse
                        .response()
                        .withBody(&#34;{\&#34;message\&#34;: \&#34;hello\&#34;}&#34;)
                        .withContentType(MediaType.APPLICATION_JSON)
        );

        StepVerifier.create(fallbackService.getWelcomeMessageByLocale(&#34;en_US&#34;))
                .expectNextMatches(welcomeMessage -&gt; &#34;hello&#34;.equals(welcomeMessage.getMessage()))
                .verifyComplete();
    }
}

```

Similar to previous posts, we&#39;re leveraging **MockServer** here to simulate a response to a predefined endpoint at **&#34;/locale/en\_US/message&#34;**. The response is a json response that matches our DTO. If you run this test, it will predictably fail.

Now let&#39;s change the service code to make it pass:

```java
    public Mono&lt;WelcomeMessage&gt; getWelcomeMessageByLocale(String locale) {
        return this.serviceAWebClient.get()
                .uri(uriBuilder -&gt; uriBuilder.path(&#34;/locale/{locale}/message&#34;).build(locale))
                .retrieve()
                .bodyToMono(WelcomeMessage.class);
    }

```

As advertised, our test now passes.

Okay, so this is fine if the downstream service is behaving normally, but what if the service is misbehaving and barfing up 500s? In that case, our **WebClient** will just propagate up the error to our service. Now, let&#39;s say that 90% of your user base speaks English, it would seem pretty dumb to bring down this entire portion of the app just because you couldn&#39;t get a specific welcome message, even though that welcome message is almost always going to be in English.

To simulate this failure, we can similarly use **MockServer**:

```java
    @Test
    public void welcomeMessage_fallsBackToEnglishWhenError() {
        this.clientAndServer.when(
                request()
                    .withPath(&#34;/locale/fr/message&#34;)
                    .withMethod(HttpMethod.GET.name())
        ).respond(
                HttpResponse.response()
                    .withStatusCode(503)
        );

        StepVerifier.create(fallbackService.getWelcomeMessageByLocale(&#34;fr&#34;))
                .expectNextMatches(welcomeMessage -&gt; &#34;hello fallback!&#34;.equals(welcomeMessage.getMessage()))
                .verifyComplete();
    }

```

With this now in place, we can start examining the different ways we can accomplish our goals here. One option is to just use **onErrorReturn**:

```java
    public Mono&lt;WelcomeMessage&gt; getWelcomeMessageByLocale(String locale) {
        return this.serviceAWebClient.get()
                .uri(uriBuilder -&gt; uriBuilder.path(&#34;/locale/{locale}/message&#34;).build(locale))
                .retrieve()
                .bodyToMono(WelcomeMessage.class)
                .onErrorReturn(new WelcomeMessage(&#34;hello fallback!&#34;));
    }

```

This is obviously very simple, but pretty crude. I generally prefer to use a different overloaded method for that, which is to use a **Predicate** to first check that the error type is one we are okay with falling back on:

```java
    public Mono&lt;WelcomeMessage&gt; getWelcomeMessageByLocale(String locale) {
        return this.serviceAWebClient.get()
                .uri(uriBuilder -&gt; uriBuilder.path(&#34;/locale/{locale}/message&#34;).build(locale))
                .retrieve()
                .bodyToMono(WelcomeMessage.class)
                .onErrorReturn(
                        throwable -&gt; throwable instanceof WebClientResponseException
                            &amp;&amp; ((WebClientResponseException)throwable).getStatusCode().is5xxServerError(),
                        new WelcomeMessage(&#34;hello fallback!&#34;)
                );
    }

```

With either of those changes, our test now passes, and we&#39;ve added a small but meaningful win to our app! Remember to [check out the source code on Github](https://github.com/nfisher23/reactive-programming-webflux/tree/master/api-calls-and-resilience) to see this in action.