---
title: "Java IO: Paths and Files"
date: 2018-11-03T13:18:09
draft: false
tags: [java, java stream api, i/o]
---

The sample code for this repository can be found [on Github](https://github.com/nfisher23/iodemos).

System paths and file manipulation, usually within the java.nio package, in Java allow you to forgo some of the details related to streaming of files--which, while they offer low level details and optimization opportunities, typically take longer to develop and get right.

You can find the absolute path:

```java
@Test
public void visitingDirectories_walkSubDirectories() throws Exception {
    try (Stream&lt;Path&gt; entries  = Files.walk(Paths.get(Utils.pathToResources))) {
        System.out.println(&#34;counting via walk&#34;);
        long count = entries.peek(System.out::println).count();
        assertTrue(count &gt; 0);
    }
}

```

With the absolute path of your current working directory in place, you can then ask for the parent, filename, root, and many other metadata qualities:

```java
@Test
public void someUsefulStuff() {
    Path absolutePath = Paths.get(&#34;&#34;).toAbsolutePath();

    Path parentOfAbsolutePath = absolutePath.getParent();
    System.out.println(parentOfAbsolutePath);
    assertNotNull(parentOfAbsolutePath);

    Path fileNameOfPath = parentOfAbsolutePath.getFileName();
    System.out.println(fileNameOfPath);
    assertNotNull(fileNameOfPath);

    Path root = absolutePath.getRoot();
    System.out.println(root); // different on windows vs unix
    assertNotNull(root);
}

```

You can get a `Scanner` and scan through all the characters:

```java
@Test
public void getScannerFromPath() throws Exception {
    Path pathToExampleFile = Paths.get(Utils.simpleExampleFilePath);

    try (Scanner scanner = new Scanner(pathToExampleFile)) {
        System.out.println(scanner.useDelimiter(&#34;\\Z&#34;).next());
        assertNotNull(scanner);
    }
}

```

Another way to read a file as a String (in this example, a file with UTF-8 character set) is to start by reading all the bytes from that file, and then constructing a String out of that byte array:

```java
@Test
public void files_readAllBytes_thenStrings() throws Exception {
    Path pathToExampleFile = Paths.get(Utils.simpleExampleFilePath);

    byte[] bytes = Files.readAllBytes(pathToExampleFile);
    String content = new String(bytes, StandardCharsets.UTF_8);

    assertEquals(content, &#34;this is some text&#34;);
}

```

You can, similarly, read in all the lines as a `List&lt;String&gt;`:

```java
@Test
public void files_sequenceOfLines() throws Exception {
    Path pathToExampleFile = Paths.get(Utils.simpleExampleFilePath);

    List&lt;String&gt; allLines = Files.readAllLines(pathToExampleFile);

    assertEquals(&#34;this is some text&#34;, allLines.get(0));
}

```

For write operations, you can write a string by converting them to bytes, with the appropriate character set, then using `Files.write(..)`:

```java
@Test
public void files_writeStringToFile() throws Exception {
    String stuffToWrite = &#34;some new text&#34;;

    Path newFilePath = Paths.get(Utils.pathToResources &#43; &#34;new-file-with-text.txt&#34;);

    Files.write(newFilePath, stuffToWrite.getBytes(StandardCharsets.UTF_8));

    assertEquals(stuffToWrite, Utils.readFileAsText(newFilePath));
}

```

The default behavior of `Files.write(..)` is to overwrite a file if it already exists. If you want to append a file, you can do so by passing the `StandardOpenOption` enumerated value into the third argument:

```java
@Test
public void files_appendToFile() throws Exception {
    String beginnings = &#34;some next text\n&#34;;

    Path newFilePath = Paths.get(Utils.pathToResources, &#34;append-string-ex.txt&#34;);
    Files.write(newFilePath, beginnings.getBytes(StandardCharsets.UTF_8));

    // append
    for (int i = 0; i &lt; 5; i&#43;&#43;) {
        Files.write(newFilePath, beginnings.getBytes(StandardCharsets.UTF_8), StandardOpenOption.APPEND);
    }

    String writtenText = Utils.readFileAsText(newFilePath);
    assertTrue(writtenText.startsWith(beginnings &#43; beginnings));
}

```