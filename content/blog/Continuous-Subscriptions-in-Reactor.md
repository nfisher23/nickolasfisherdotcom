---
title: "Continuous Subscriptions in Reactor"
date: 2020-09-12T17:14:10
draft: false
tags: [java, reactive]
---

There are use cases for wanting to immediately subscribe to a **Flux** or a **Mono** immediately after the subscription has completed. The most obvious use case is if your application needs to continuously poll for values.

To continuously subscribe to a **Flux**, the easiest way to do so is to use **repeat**:

```java
        Flux.generate(synchronousSink -> synchronousSink.next(new Noop()))
            .repeat()
            .subscribe(noop -> {
                int millis = ZonedDateTime.now().getNano() / 1_000_000;
                if (millis % 500 == 0) {
                    System.out.println("noop");
                }
            });

```

Note that I'm using this simple class to facilitate that example and a few below:

```java

    private static class Noop {
        private int something;

        public void setSomething(int something) {
            this.something = something;
        }

        public int getSomething() {
            return this.something;
        }
    }

```

This simple example will print "noop" to the console every time we hit a half or whole second. It's important to note that if your **Flux** throws an exception at this point, then the subscription will be terminated. If you want to just keep blindly retrying every time you hit an unexpected exception, that's a one liner to fix:

```java
        Flux.generate(synchronousSink -> {
                if (ZonedDateTime.now().getNano() / 1_000_000 % 500 == 0) {
                    synchronousSink.error(new RuntimeException());
                }
                synchronousSink.next(new Noop());
            })
            .repeat()
            .retry()
            .subscribe(noop -> {
                        int millis = ZonedDateTime.now().getNano() / 1_000_000;
                        if (millis % 500 == 0) {
                            System.out.println("noop");
                        }
                    }
            );

```

This example generates an error every half or whole minute. A few "noop"s will actually make it through in some cases just due to timing between the first sink being called and the actual subscription getting executed \[usually nanoseconds later\].

We can also retry a **Mono** with the same syntax:

```java
        Mono.fromFuture(CompletableFuture.supplyAsync(() -> new Noop()))
            .repeat()
            .subscribe(noop -> {
                int millis = ZonedDateTime.now().getNano() / 1_000_000;
                if (millis % 500 == 0) {
                    System.out.println("noop");
                }
            });

```

An important related note: if you keep resubscribing to a mono from a **CompletableFuture** like this, the future will only actually execute once, and the value will just get propogated down multiple times. We can demonstrate this behavior like so:

```java
        AtomicInteger count = new AtomicInteger();
        Mono.fromFuture(CompletableFuture.supplyAsync(() -> new Noop() {{ setSomething(count.incrementAndGet()); }}))
            .repeat()
            .subscribe(noop -> {
                int millis = ZonedDateTime.now().getNano() / 1_000_000;
                if (millis % 500 == 0) {
                    System.out.println("noop " + noop.getSomething());
                }
            });

```

This prints out:

```bash
noop 1
noop 1
noop 1
noop 1
noop 1
noop 1
noop 1
...

```

To get the **CompletableFuture** to execute every time, we need to wrap it in a **Supplier**, which will here be a lambda:

```java
        AtomicInteger count = new AtomicInteger();
        Mono.fromFuture(() -> CompletableFuture.supplyAsync(() -> new Noop() {{ setSomething(count.incrementAndGet()); }}))
            .repeat()
            .subscribe(noop -> {
                int millis = ZonedDateTime.now().getNano() / 1_000_000;
                if (millis % 500 == 0) {
                    System.out.println("noop " + noop.getSomething());
                }
            });

```

With this change, we can see in the console:

```bash
noop 231278
noop 231279
noop 231280
...

```

Note that the same restrictions on an exception being thrown and terminating the continuous subscription apply--if you want to avoid that, you need to add in a **retry** just like we did with the **Flux** above.
