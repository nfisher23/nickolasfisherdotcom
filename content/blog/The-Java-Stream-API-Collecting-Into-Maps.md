---
title: "The Java Stream API: Collecting Into Maps"
date: 2018-10-21T16:07:06
draft: false
tags: [java, java stream api]
---

You can view the sample code associated with this post [on GitHub](https://github.com/nfisher23/java_stream_api_samples).

Taking a stream of POJOs and transforming it into a map is one of the more common uses of Streams.
There are a few built in ways to get what you want. The simplest way to map one field to another in a POJO is by
using the `.toMap(..)` method.

Starting with a SimplePair object:

```java
public class SimplePair {

    private String name;
    private int id;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    @Override
    public String toString() {
        return &#34;name-&#34; &#43; name &#43; &#34;,id-&#34; &#43; id;
    }
}
```

If we use the following method to set up five SimplePair objects in a collection:

```java
public static List&lt;SimplePair&gt; generateSimplePairs(int numToGenerate) {
    List&lt;SimplePair&gt; pairs = new ArrayList&lt;&gt;();
    for (int i = 1; i &lt;= numToGenerate; i&#43;&#43;) {
        SimplePair pair = new SimplePair();

        pair.setId(i);
        pair.setName(&#34;pair&#34; &#43; i);

        pairs.add(pair);
    }
    return pairs;
}
```

Here, our collection would look like `[(id: 1, name: pair1), (id: 2, name: pair2), (id: 3, name: pair3) ... ]`.

We can thus use Streams to map the ids to the names:

```java
@Test
public void collect_mapIdToName() {
    Map&lt;Integer, String&gt; mapIdToName = simplePairs.stream().collect(Collectors.toMap(SimplePair::getId, SimplePair::getName));

    assertEquals(mapIdToName.get(3), &#34;pair3&#34;);
    assertEquals(mapIdToName.get(5), &#34;pair5&#34;);
}

```

In the more common case where you want to map the id&#39;s to the object itself, use `Function.identity()`:

```java
@Test
public void collect_mapIdToPair() {
    Map&lt;Integer, SimplePair&gt; mapIdToObject = simplePairs.stream().collect(Collectors.toMap(SimplePair::getId, Function.identity()));

    SimplePair pair1 = mapIdToObject.get(1);
    SimplePair pair4 = mapIdToObject.get(4);

    assertEquals(pair1.toString(), &#34;name-pair1,id-1&#34;);
    assertEquals(pair4.toString(), &#34;name-pair4,id-4&#34;);
}

```

Unfortunately, there are caveats to this approach. If you insist on using the `toMap(..)` method, you will have to resolve duplicate values, if there are any,
by adding a lambda that resolves the disparity. If we have more pairs with the same ids, by adding two with the id of three:

```java
private void addDuplicatePairs() {
    simplePairs.add(new SimplePair() {{
        setId(3);
        setName(&#34;another-pair3&#34;);
    }});

    simplePairs.add(new SimplePair() {{
        setId(3);
        setName(&#34;yet-another-pair3&#34;);
    }});
}

```

We can resolve to the existing value or the new value, or by any other determination, like so:

```java
@Test
public void collect_resolveConflictsOnMap() {
    addDuplicatePairs();

    Map&lt;Integer, SimplePair&gt; mapWithExistingValue = simplePairs.stream().collect(Collectors.toMap(
            SimplePair::getId,
            Function.identity(),
            (existingValue, newValue) -&gt; existingValue // established is always better
    ));

    assertEquals(&#34;pair3&#34;, mapWithExistingValue.get(3).getName());

    Map&lt;Integer, SimplePair&gt; mapWithNewestValue = simplePairs.stream().collect(Collectors.toMap(
            SimplePair::getId,
            Function.identity(),
            (existingValue, newValue) -&gt; newValue // newer is always better
    ));

    assertEquals(&#34;yet-another-pair3&#34;, mapWithNewestValue.get(3).getName());
}

```

To collect it into a list gets more complicated if you stay with the `.toMap(..)` method, so we&#39;ll bail on that for simplicity right now. When we want a Map&lt;&gt; from a collection, the most common reason would be to collect each one into a bucket based on some criteria, usually by a single property field.

The easiest and most straightforward way to do that is to use the built in `groupingBy(..)` collector:

```java
@Test
public void groupingBy_sortToMap() {
    addDuplicatePairs();

    Map&lt;Integer, List&lt;SimplePair&gt;&gt; mapIdsToPairs =
            simplePairs.stream().collect(Collectors.groupingBy(SimplePair::getId));

    assertEquals(3, mapIdsToPairs.get(3).size());
    assertEquals(3, mapIdsToPairs.get(3).get(0).getId());
    assertEquals(3, mapIdsToPairs.get(3).get(1).getId());
    assertEquals(&#34;another-pair3&#34;, mapIdsToPairs.get(3).get(1).getName());
}

```

Finally, `partitioningBy(..)` allows you to map boolean values to a collection, where the &#34;true&#34; key maps to all the objects for which the lambda expression evaluates to &#34;true&#34;.
Here, we add our duplicate pairs again, then place all the pairs that have an id of 3 into the true bucket, and all the others into the false bucket:

```java
@Test
public void parititioningBy_sortByTrueAndFalse() {
    addDuplicatePairs();

    Map&lt;Boolean, List&lt;SimplePair&gt;&gt; partitionedByIdOf3 =
            simplePairs.stream().collect(
                    Collectors.partitioningBy(pair -&gt; pair.getId() == 3)
            );

    assertEquals(3, partitionedByIdOf3.get(true).size());
    assertEquals(4, partitionedByIdOf3.get(false).size());
}

```