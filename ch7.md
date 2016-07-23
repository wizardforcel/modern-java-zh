# Java 8 API 示例：字符串、数值、算术和文件

> 原文：[Java 8 API by Example: Strings, Numbers, Math and Files](http://winterbe.com/posts/2015/03/25/java8-examples-string-number-math-files/) 

> 译者：[飞龙](https://github.com/wizardforcel)  

> 协议：[CC BY-NC-SA 4.0](http://creativecommons.org/licenses/by-nc-sa/4.0/)

大量的教程和文章都涉及到Java8中最重要的改变，例如[lambda表达式](ch1.md)和[函数式数据流](ch2.md)。但是此外许多现存的类在[JDK 8 API](http://winterbe.com/posts/2014/03/29/jdk8-api-explorer/)中也有所改进，带有一些实用的特性和方法。

这篇教程涉及到Java 8 API中的那些小修改 -- 每个都使用简单易懂的代码示例来描述。让我们好好看一看字符串、数值、算术和文件。

## 处理字符串

两个新的方法可在字符串类上使用：`join`和`chars`。第一个方法使用指定的分隔符，将任何数量的字符串连接为一个字符串。

```java
String.join(":", "foobar", "foo", "bar");
// => foobar:foo:bar
```

第二个方法`chars`从字符串所有字符创建数据流，所以你可以在这些字符上使用流式操作。

```java
"foobar:foo:bar"
    .chars()
    .distinct()
    .mapToObj(c -> String.valueOf((char)c))
    .sorted()
    .collect(Collectors.joining());
// => :abfor
```

不仅仅是字符串，正则表达式模式串也能受益于数据流。我们可以分割任何模式串，并创建数据流来处理它们，而不是将字符串分割为单个字符的数据流，像下面这样：

```java
Pattern.compile(":")
    .splitAsStream("foobar:foo:bar")
    .filter(s -> s.contains("bar"))
    .sorted()
    .collect(Collectors.joining(":"));
// => bar:foobar
```

此外，正则模式串可以转换为谓词。这些谓词可以像下面那样用于过滤字符串流：

```java
Pattern pattern = Pattern.compile(".*@gmail\\.com");
Stream.of("bob@gmail.com", "alice@hotmail.com")
    .filter(pattern.asPredicate())
    .count();
// => 1
```

上面的模式串接受任何以`@gmail.com`结尾的字符串，并且之后用作Java8的`Predicate`来过滤电子邮件地址流。

## 处理数值

Java8添加了对无符号数的额外支持。Java中的数值总是有符号的，例如，让我们来观察`Integer`：

`int`可表示最多`2 ** 32`个数。Java中的数值默认为有符号的，所以最后一个二进制数字表示符号（0为正数，1为负数）。所以从十进制的0开始，最大的有符号正整数为`2 ** 31 - 1`。

你可以通过`Integer.MAX_VALUE`来访问它：

```java
System.out.println(Integer.MAX_VALUE);      // 2147483647
System.out.println(Integer.MAX_VALUE + 1);  // -2147483648
```

Java8添加了解析无符号整数的支持，让我们看看它如何工作：

```java
long maxUnsignedInt = (1l << 32) - 1;
String string = String.valueOf(maxUnsignedInt);
int unsignedInt = Integer.parseUnsignedInt(string, 10);
String string2 = Integer.toUnsignedString(unsignedInt, 10);
```

就像你看到的那样，现在可以将最大的无符号数`2 ** 32 - 1`解析为整数。而且你也可以将这个数值转换回无符号数的字符串表示。

这在之前不可能使用`parseInt`完成，就像这个例子展示的那样：

```java
try {
    Integer.parseInt(string, 10);
}
catch (NumberFormatException e) {
    System.err.println("could not parse signed int of " + maxUnsignedInt);
}
```

这个数值不可解析为有符号整数，因为它超出了最大范围`2 ** 31 - 1`。

## 算术运算

`Math`工具类新增了一些方法来处理数值溢出。这是什么意思呢？我们已经看到了所有数值类型都有最大值。所以当算术运算的结果不能被它的大小装下时，会发生什么呢？

```java
System.out.println(Integer.MAX_VALUE);      // 2147483647
System.out.println(Integer.MAX_VALUE + 1);  // -2147483648
```

就像你看到的那样，发生了整数溢出，这通常是我们不愿意看到的。

Java8添加了严格数学运算的支持来解决这个问题。`Math`扩展了一些方法，它们全部以`exact`结尾，例如`addExact`。当运算结果不能被数值类型装下时，这些方法通过抛出`ArithmeticException`异常来合理地处理溢出。

```java
try {
    Math.addExact(Integer.MAX_VALUE, 1);
}
catch (ArithmeticException e) {
    System.err.println(e.getMessage());
    // => integer overflow
}
```

当尝试通过`toIntExact`将长整数转换为整数时，可能会抛出同样的异常：

```java
try {
    Math.toIntExact(Long.MAX_VALUE);
}
catch (ArithmeticException e) {
    System.err.println(e.getMessage());
    // => integer overflow
}
```

## 处理文件

`Files`工具类首次在Java7中引入，作为NIO的一部分。JDK8 API添加了一些额外的方法，它们可以将文件用于函数式数据流。让我们深入探索一些代码示例。

### 列出文件

`Files.list`方法将指定目录的所有路径转换为数据流，便于我们在文件系统的内容上使用类似`filter`和`sorted`的流操作。

```java
try (Stream<Path> stream = Files.list(Paths.get(""))) {
    String joined = stream
        .map(String::valueOf)
        .filter(path -> !path.startsWith("."))
        .sorted()
        .collect(Collectors.joining("; "));
    System.out.println("List: " + joined);
}
```

上面的例子列出了当前工作目录的所有文件，之后将每个路径都映射为它的字符串表示。之后结果被过滤、排序，最后连接为一个字符串。如果你还不熟悉函数式数据流，你应该阅读我的[Java8数据流教程](ch2.md)。

你可能已经注意到，数据流的创建包装在`try-with`语句中。数据流实现了`AutoCloseable`，并且这里我们需要显式关闭数据流，因为它基于IO操作。

> 返回的数据流是`DirectoryStream`的封装。如果需要及时处理文件资源，就应该使用`try-with`结构来确保在流式操作完成后，数据流的`close`方法被调用。

### 查找文件

下面的例子演示了如何查找在目录及其子目录下的文件：

```java
Path start = Paths.get("");
int maxDepth = 5;
try (Stream<Path> stream = Files.find(start, maxDepth, (path, attr) ->
        String.valueOf(path).endsWith(".js"))) {
    String joined = stream
        .sorted()
        .map(String::valueOf)
        .collect(Collectors.joining("; "));
    System.out.println("Found: " + joined);
}
```

`find`方法接受三个参数：目录路径`start`是起始点，`maxDepth`定义了最大搜索深度。第三个参数是一个匹配谓词，定义了搜索的逻辑。上面的例子中，我们搜索了所有JavaScirpt文件（以`.js`结尾的文件名）。

我们可以使用`Files.walk`方法来完成相同的行为。这个方法会遍历每个文件，而不需要传递搜索谓词。

```java
Path start = Paths.get("");
int maxDepth = 5;
try (Stream<Path> stream = Files.walk(start, maxDepth)) {
    String joined = stream
        .map(String::valueOf)
        .filter(path -> path.endsWith(".js"))
        .sorted()
        .collect(Collectors.joining("; "));
    System.out.println("walk(): " + joined);
}
```

这个例子中，我们使用了流式操作`filter`来完成和上个例子相同的行为。

### 读写文件

将文本文件读到内存，以及向文本文件写入字符串在Java 8 中是简单的任务。不需要再去摆弄读写器了。`Files.readAllLines`从指定的文件把所有行读进字符串列表中。你可以简单地修改这个列表，并且将它通过`Files.write`写到另一个文件中：

```java
List<String> lines = Files.readAllLines(Paths.get("res/nashorn1.js"));
lines.add("print('foobar');");
Files.write(Paths.get("res/nashorn1-modified.js"), lines);
```

要注意这些方法对内存并不十分高效，因为整个文件都会读进内存。文件越大，所用的堆区也就越大。

你可以使用`Files.lines`方法来作为内存高效的替代。这个方法读取每一行，并使用函数式数据流来对其流式处理，而不是一次性把所有行都读进内存。

```java
try (Stream<String> stream = Files.lines(Paths.get("res/nashorn1.js"))) {
    stream
        .filter(line -> line.contains("print"))
        .map(String::trim)
        .forEach(System.out::println);
}
```

如果你需要更多的精细控制，你需要构造一个新的`BufferedReader`来代替：

```java
Path path = Paths.get("res/nashorn1.js");
try (BufferedReader reader = Files.newBufferedReader(path)) {
    System.out.println(reader.readLine());
}
```

或者，你需要写入文件时，简单地构造一个`BufferedWriter`来代替：

```java
Path path = Paths.get("res/output.js");
try (BufferedWriter writer = Files.newBufferedWriter(path)) {
    writer.write("print('Hello World');");
}
```

`BufferedReader`也可以访问函数式数据流。`lines`方法在它所有行上面构建数据流：

```java
Path path = Paths.get("res/nashorn1.js");
try (BufferedReader reader = Files.newBufferedReader(path)) {
    long countPrints = reader
        .lines()
        .filter(line -> line.contains("print"))
        .count();
    System.out.println(countPrints);
}
```

目前为止你可以看到Java8提供了三个简单的方法来读取文本文件的每一行，使文件处理更加便捷。

不幸的是你需要显式使用`try-with`语句来关闭文件流，这会使示例代码有些凌乱。我期待函数式数据流可以在调用类似`count`和`collect`时可以自动关闭，因为你不能在相同数据流上调用终止操作两次。

我希望你能喜欢这篇文章。所有示例代码都托管在[Github](https://github.com/winterbe/java8-tutorial)上，还有来源于我博客其它[Java8文章](http://winterbe.com/java/)的大量的代码片段。如果这篇文章对你有所帮助，请[收藏](https://github.com/winterbe/java8-tutorial)我的仓库，并且在Twitter上[关注我](https://twitter.com/winterbe_)。

请坚持编程！
