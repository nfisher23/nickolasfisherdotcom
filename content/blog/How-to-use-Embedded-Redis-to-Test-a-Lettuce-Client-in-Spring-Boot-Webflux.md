---
title: "How to use Embedded Redis to Test a Lettuce Client in Spring Boot Webflux"
date: 2021-03-27T21:22:32
draft: false
tags: [java, distributed systems, spring, maven, reactive, webflux]
---

The source code for this article [can be found on Github](https://github.com/nfisher23/reactive-programming-webflux/tree/master/reactive-redis).

[Lettuce](https://github.com/lettuce-io/lettuce-core) is a redis client with reactive support. There is a super handy [embedded redis for java project](https://github.com/kstyrc/embedded-redis) out there, and this kind of integration testing inside your service is worth its weight in gold, in my humble opinion. This post will detail how to merge both of these worlds together, and set up redis integration tests when you're using a lettuce client.

You will want to start by ensuring that you add lettuce and embedded redis to your **pom.xml**\[the config for gradle is analogous\]:

```xml
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.20</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>io.lettuce</groupId>
            <artifactId>lettuce-core</artifactId>
            <version>6.1.0.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>com.github.kstyrc</groupId>
            <artifactId>embedded-redis</artifactId>
            <version>0.6</version>
            <scope>test</scope>
        </dependency>

```

I also added lombok for convenience, as you can see.

Let's set up a skeleton class with a couple methods which, when implemented, will expose get and set operations for a client:

```java
public class RedisDataService {

    private final RedisStringReactiveCommands<String, String> redisStringReactiveCommands;

    public RedisDataService(RedisStringReactiveCommands<String, String> redisStringReactiveCommands) {
        this.redisStringReactiveCommands = redisStringReactiveCommands;
    }

    public Mono<Void> writeThing(Thing thing) {
        return Mono.empty();
    }

    public Mono<Thing> getThing(Integer id) {
        return Mono.empty();
    }
}

```

Where our **Thing** DTO is \[note the lombok annotations generating code for us\]:

```java
@Builder
@Getter
public class Thing {
    private Integer id;
    private String value;
}

```

Now we can write an automated test to target the behavior that we're about to write:

```java
public class RedisDataServiceTest {
    private static RedisServer redisServer;

    private static int getOpenPort() {
        try {
            int port = -1;
            try (ServerSocket socket = new ServerSocket(0)) {
                port = socket.getLocalPort();
            }
            return port;
        } catch (Exception e) {
            throw new RuntimeException();
        }
    }

    private static int port = getOpenPort();

    private RedisDataService redisDataService;

    @BeforeAll
    public static void setupRedisServer() throws Exception {
        redisServer = new RedisServer(port);
        redisServer.start();
    }

    @BeforeEach
    public void setupRedisClient() {
        RedisClient redisClient = RedisClient.create("redis://localhost:" + port);
        redisDataService = new RedisDataService(redisClient.connect().reactive());
    }

    @Test
    public void canWriteAndReadThing() {
        Mono<Void> writeMono = redisDataService.writeThing(Thing.builder().id(1).value("hello-redis").build());

        StepVerifier.create(writeMono).verifyComplete();

        StepVerifier.create(redisDataService.getThing(1))
                .expectNextMatches(thing ->
                    thing.getId() == 1 &amp;&amp;
                        "hello-redis".equals(thing.getValue())
                )
                .verifyComplete();
    }

    @AfterAll
    public static void teardownRedisServer() {
        redisServer.stop();
    }
}

```

The framework for setting up embedded redis and testing it looks like so:

1. Before the entire test class runs, we grab a random, free open port
2. We tell our embedded redis to start on that port
3. Before each test runs, we create a redis client and tell it to connect to that port
4. We pass in that redis client \[ensuring it's reactive\] to our class under test

As far as the test itself, we are just persisting to and reading from redis and letting the service abstraction handle it for us. Obviously, since we've written no code yet, this will fail. To make it pass we can implement the code like so:

```java
public class RedisDataService {

    private final RedisStringReactiveCommands<String, String> redisStringReactiveCommands;

    public RedisDataService(RedisStringReactiveCommands<String, String> redisStringReactiveCommands) {
        this.redisStringReactiveCommands = redisStringReactiveCommands;
    }

    public Mono<Void> writeThing(Thing thing) {
        return this.redisStringReactiveCommands
                .set(thing.getId().toString(), thing.getValue())
                .then();
    }

    public Mono<Thing> getThing(Integer id) {
        return this.redisStringReactiveCommands.get(id.toString())
                .map(response -> Thing.builder().id(id).value(response).build());
    }
}

```

This code is pretty straightforward: write to redis using a simple key/value **set**, and read from redis using a key/value **get**. Once you get the value out of redis, throw it into a data structure for the client.

Remember to [check out the source code on Github](https://github.com/nfisher23/reactive-programming-webflux/tree/master/reactive-redis).
