---
title: "How to Make Sequential API Calls and Merge the Results In Spring Boot Webflux"
date: 2020-09-19T16:01:14
draft: false
tags: [java, spring, reactive, webflux]
---

The source code for this article [can be found on Github](https://github.com/nfisher23/reactive-programming-webflux/tree/master/api-calls-and-resilience).

In reactive programming, it's a game of callbacks. In the vast majority of cases, you will want to defer all of your I/O operations to the library you are using \[typically, netty, under the hood\], and stay focused on setting up the flow so that the right functions are invoked in the right order. Sometimes you will want to make calls in parallel, sometimes you need data from a previous call or operation available in order to invoke that right function.

The key to coordinating operations like this is to defer to the [various operations on Mono](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html), or in the streaming case, [those on flux](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html). This post is focused on a very specific need:

1. You make a call to get data from Service A
2. You need the result of the call from Service A in order to make another call
3. You want to do something with the results of 1 and 2

In this case, we need **zipWhen**.

## Setup the App

You can spin up an application in the sprint boot initializr or whatever is most comfortable to you \[ensure you're selecting the **Spring Reactive Web** option\]. Since we're making a network call, we will want to start by setting up our WebClient:

```java
public class Config {

    @Bean("service-a-web-client")
    public WebClient serviceAWebClient() {
        HttpClient httpClient = HttpClient.create().tcpConfiguration(tcpClient ->
                tcpClient.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 1000)
                        .doOnConnected(connection -> connection.addHandlerLast(new ReadTimeoutHandler(1000, TimeUnit.MILLISECONDS)))
        );

        return WebClient.builder()
                .baseUrl("http://your-base-url.com")
                .clientConnector(new ReactorClientHttpConnector(httpClient))
                .build();
    }
}

```

Here I've included a connection and read timeout of 1 second just because the default is like 30 seconds which nobody in their right mind would ever want to use.

Now let's set up the situation, first we will make two DTOs, expected to be received on two different calls: **FirstCallDTO** and **SecondCallDTO**:

```java
package com.nickolasfisher.webflux.model;

public class FirstCallDTO {
    private Integer fieldFromFirstCall;

    public Integer getFieldFromFirstCall() {
        return fieldFromFirstCall;
    }

    public void setFieldFromFirstCall(Integer fieldFromFirstCall) {
        this.fieldFromFirstCall = fieldFromFirstCall;
    }
}

...in another file...

package com.nickolasfisher.webflux.model;

public class SecondCallDTO {
    private String fieldFromSecondCall;

    public String getFieldFromSecondCall() {
        return fieldFromSecondCall;
    }

    public void setFieldFromSecondCall(String fieldFromSecondCall) {
        this.fieldFromSecondCall = fieldFromSecondCall;
    }
}

```

I'm also going to add mockserver as a test dependency and write the test first \[TDD\]. So you can modify your **pom.xml** to include:

```java
        <dependency>
            <groupId>org.mock-server</groupId>
            <artifactId>mockserver-junit-jupiter</artifactId>
            <version>5.11.1</version>
        </dependency>
        <dependency>
            <groupId>org.mock-server</groupId>
            <artifactId>mockserver-netty</artifactId>
            <version>5.11.0</version>
            <scope>test</scope>
        </dependency>

```

We will create a bare bones service called **CombiningCallsService** to work with, starting with it empty as is the TDD way:

```java
@Service
public class CombiningCallsService {

    private final WebClient serviceAWebClient;

    public CombiningCallsService(@Qualifier("service-a-web-client") WebClient serviceAWebClient) {
        this.serviceAWebClient = serviceAWebClient;
    }

    public Mono<SecondCallDTO> sequentialCalls(Integer key) {
        return null;
    }
}

```

Then we can write the test, leveraging mockserver's direct integration with junit by just using an annotation, and setting up data:

```java
@ExtendWith(MockServerExtension.class)
public class CombiningCallsServiceIT {
    private CombiningCallsService combiningCallsService;

    private WebClient webClient;
    private ClientAndServer clientAndServer;

    public CombiningCallsServiceIT(ClientAndServer clientAndServer) {
        this.clientAndServer = clientAndServer;
        this.webClient = WebClient.builder()
                .baseUrl("http://localhost:" + clientAndServer.getPort())
                .build();
    }

    @BeforeEach
    public void setup() {
        combiningCallsService = new CombiningCallsService(webClient);
    }

    @AfterEach
    public void reset() {
        clientAndServer.reset();
    }

    @Test
    public void callsFirstAndUsesCallToGetSecond() {
        HttpRequest expectedFirstRequest = HttpRequest.request()
                .withMethod(HttpMethod.GET.name())
                .withPath("/first/endpoint/10");

        this.clientAndServer.when(
                expectedFirstRequest
        ).respond(
                HttpResponse.response()
                        .withBody("{\"fieldFromFirstCall\": 100}")
                        .withContentType(MediaType.APPLICATION_JSON)
        );

        HttpRequest expectedSecondRequest = HttpRequest.request()
                .withMethod(HttpMethod.GET.name())
                .withPath("/second/endpoint/100");

        this.clientAndServer.when(
                expectedSecondRequest
        ).respond(
                HttpResponse.response()
                        .withBody("{\"fieldFromSecondCall\": \"hello\"}")
                        .withContentType(MediaType.APPLICATION_JSON)
        );

        StepVerifier.create(this.combiningCallsService.sequentialCalls(10))
                .expectNextMatches(secondCallDTO -> "hello".equals(secondCallDTO.getFieldFromSecondCall()))
                .verifyComplete();

        this.clientAndServer.verify(expectedFirstRequest, VerificationTimes.once());
        this.clientAndServer.verify(expectedSecondRequest, VerificationTimes.once());
    }
}

```

The one test in this class with:

1. Sets up two expectations on mock server: if you make a GET request to **/first/endpoint/10** then we will respond with a json body. If you make a GET request to **/second/endpoint/100** we respond with a different json body.
2. Leverages **StepVerifier** to subscribe to the **Mono** returned from our custom service, and asserts that we will see a single result that matches the lambda passed in \["hello" must equal the field in the result for this test to pass\]
3. Verifies that both requests were actually made.


If you run this test, it predictably fails. We can get it to pass by leveraging zipWhen, which will be a callback that is only invoked after the previous Mono completes, then passes in the result of that mono into the next one:

```java
@Service
public class CombiningCallsService {

    private final WebClient serviceAWebClient;

    public CombiningCallsService(@Qualifier("service-a-web-client") WebClient serviceAWebClient) {
        this.serviceAWebClient = serviceAWebClient;
    }

    public Mono<SecondCallDTO> sequentialCalls(Integer key) {
        return this.serviceAWebClient.get()
                .uri(uriBuilder -> uriBuilder.path("/first/endpoint/{param}").build(key))
                .retrieve()
                .bodyToMono(FirstCallDTO.class)
                .zipWhen(firstCallDTO ->
                    serviceAWebClient.get().uri(
                            uriBuilder ->
                                    uriBuilder.path("/second/endpoint/{param}")
                                            .build(firstCallDTO.getFieldFromFirstCall()))
                            .retrieve()
                            .bodyToMono(SecondCallDTO.class),
                    (firstCallDTO, secondCallDTO) -> secondCallDTO
                );
    }
}

```

That final lambda that just returns **secondCallDTO** is a resolver function. If necessary, you can do more with that by injecting your own custom code there, changing the return type, whatever.

And with that you should be good to go. Reminder: [this code is on github](https://github.com/nfisher23/reactive-programming-webflux/tree/master/api-calls-and-resilience).
