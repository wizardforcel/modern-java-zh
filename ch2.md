# Java 8 数据流教程

> 原文：[Java 8 Stream Tutorial](http://winterbe.com/posts/2014/07/31/java8-stream-tutorial-examples/) 

> 译者：[飞龙](https://github.com/wizardforcel)  

> 协议：[CC BY-NC-SA 4.0](http://creativecommons.org/licenses/by-nc-sa/4.0/)

这个示例驱动的教程是Java8数据流（Stream）的深入总结。当我第一次看到`Stream`API时，我非常疑惑，因为它听起来和Java IO的`InputStream` 和 `OutputStream`一样。但是Java8的数据流是完全不同的东西。数据流是单体（Monad），并且在Java8函数式编程中起到重要作用。

> 在函数式编程中，单体是一个结构，表示定义为步骤序列的计算。A type with a monad structure defines what it means to chain operations, or nest functions of that type together.

这个教程教给你如何使用Java8数据流，以及如何使用不同种类的可用的数据流操作。你将会学到处理次序以及流操作的次序如何影响运行时效率。这个教程也会详细讲解更加强大的流操作，`reduce`、`collect`和`flatMap`。最后，这个教程会深入探讨并行流。

如果你还不熟悉Java8的lambda表达式，函数式接口和方法引用，你可能需要在开始这一章之前，首先阅读我的[Java8教程](ch1.md)。

更新 - 我现在正在编写用于浏览器的Java8数据流API的JavaScript实现。如果你对此感兴趣，请在Github上访问[Stream.js](https://github.com/winterbe/streamjs)。非常期待你的反馈。

## 数据流如何工作

数据流表示元素的序列，并支持不同种类的操作来执行元素上的计算：

```java
List<String> myList =
    Arrays.asList("a1", "a2", "b1", "c2", "c1");

myList
    .stream()
    .filter(s -> s.startsWith("c"))
    .map(String::toUpperCase)
    .sorted()
    .forEach(System.out::println);

// C1
// C2
```

数据流操作要么是衔接操作，要么是终止操作。衔接操作返回数据流，所以我们可以把多个衔接操作不使用分号来链接到一起。终止操作无返回值，或者返回一个不是流的结果。在上面的例子中，`filter`、`map`和`sorted`都是衔接操作，而`forEach`是终止操作。列表上的所有流式操作请见[数据流的Javadoc](http://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html)。你在上面例子中看到的这种数据流的链式操作也叫作操作流水线。

多数数据流操作都接受一些lambda表达式参数，函数式接口用来指定操作的具体行为。这些操作的大多数必须是无干扰而且是无状态的。它们是什么意思呢？

当一个函数不修改数据流的底层数据源，它就是[无干扰的](http://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html#NonInterference)。例如，在上面的例子中，没有任何lambda表达式通过添加或删除集合元素修改`myList`。

当一个函数的操作的执行是确定性的，它就是[无状态的](http://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html#Statelessness)。例如，在上面的例子中，没有任何lambda表达式依赖于外部作用域中任何在操作过程中可变的变量或状态。

## 数据流的不同类型

数据流可以从多种数据源创建，尤其是集合。`List`和`Set`支持新方法`stream()` 和 `parallelStream()`，来创建串行流或并行流。并行流能够在多个线程上执行操作，它们会在之后的章节中讲到。我们现在来看看串行流：

```java
Arrays.asList("a1", "a2", "a3")
    .stream()
    .findFirst()
    .ifPresent(System.out::println);  // a1
```

在对象列表上调用`stream()`方法会返回一个通常的对象流。但是我们不需要创建一个集合来创建数据流，就像下面那样：

```java
Stream.of("a1", "a2", "a3")
    .findFirst()
    .ifPresent(System.out::println);  // a1
```

只要使用`Stream.of()`，就可以从一系列对象引用中创建数据流。

除了普通的对象数据流，Java8还自带了特殊种类的流，用于处理基本数据类型`int`、`long` 和 `double`。你可能已经猜到了它是`IntStream`、`LongStream` 和 `DoubleStream`。

`IntStream`可以使用`IntStream.range()`替换通常的`for`循环：

```java
IntStream.range(1, 4)
    .forEach(System.out::println);

// 1
// 2
// 3
```

所有这些基本数据流都像通常的对象数据流一样，但有一些不同。基本的数据流使用特殊的lambda表达式，例如，`IntFunction`而不是`Function`，`IntPredicate`而不是`Predicate`。而且基本数据流支持额外的聚合终止操作`sum()`和`average()`：

```java
Arrays.stream(new int[] {1, 2, 3})
    .map(n -> 2 * n + 1)
    .average()
    .ifPresent(System.out::println);  // 5.0
```

有时需要将通常的对象数据流转换为基本数据流，或者相反。处于这种目的，对象数据路支持特殊的映射操作`mapToInt()`、`mapToLong()` 和 `mapToDouble()`：

```java
Stream.of("a1", "a2", "a3")
    .map(s -> s.substring(1))
    .mapToInt(Integer::parseInt)
    .max()
    .ifPresent(System.out::println);  // 3
```

基本数据流可以通过`maoToObj()`转换为对象数据流：

```java
IntStream.range(1, 4)
    .mapToObj(i -> "a" + i)
    .forEach(System.out::println);

// a1
// a2
// a3
```

下面是组合示例：浮点数据流首先映射被整数数据流，之后映射为字符串的对象数据流：

```java
Stream.of(1.0, 2.0, 3.0)
    .mapToInt(Double::intValue)
    .mapToObj(i -> "a" + i)
    .forEach(System.out::println);

// a1
// a2
// a3
```

## 处理顺序

既然我们已经了解了如何创建并使用不同种类的数据流，让我们深入了解数据流操作在背后如何执行吧。

衔接操作的一个重要特性就是延迟性。观察下面没有终止操作的例子：

```java
Stream.of("d2", "a2", "b1", "b3", "c")
    .filter(s -> {
        System.out.println("filter: " + s);
        return true;
    });
```

执行这段代码时，不向控制台打印任何东西。这是因为衔接操作只在终止操作调用时被执行。

让我们通过添加终止操作`forEach`来扩展这个例子：

```java
Stream.of("d2", "a2", "b1", "b3", "c")
    .filter(s -> {
        System.out.println("filter: " + s);
        return true;
    })
    .forEach(s -> System.out.println("forEach: " + s));
```

执行这段代码会得到如下输出：

```
filter:  d2
forEach: d2
filter:  a2
forEach: a2
filter:  b1
forEach: b1
filter:  b3
forEach: b3
filter:  c
forEach: c
```

结果的顺序可能出人意料。原始的方法会在数据流的所有元素上，一个接一个地水平执行所有操作。但是每个元素在调用链上垂直移动。第一个字符串`"d2"`首先经过`filter`然后是`forEach`，执行完后才开始处理第二个字符串`"a2"`。

这种行为可以减少每个元素上所执行的实际操作数量，就像我们在下个例子中看到的那样：

```java
Stream.of("d2", "a2", "b1", "b3", "c")
    .map(s -> {
        System.out.println("map: " + s);
        return s.toUpperCase();
    })
    .anyMatch(s -> {
        System.out.println("anyMatch: " + s);
        return s.startsWith("A");
    });

// map:      d2
// anyMatch: D2
// map:      a2
// anyMatch: A2
```

只要提供的数据元素满足了谓词，`anyMatch`操作就会返回`true`。对于第二个传递`"A2"`的元素，它的结果为真。由于数据流的链式调用时垂直执行的，`map`这里只需要执行两次。所以`map`会执行尽可能少的次数，而不是把所有元素都映射一遍。

### 为什么顺序如此重要

下面的例子由两个衔接操作`map`和`filter`，以及一个终止操作`forEach`组成。让我们再来看看这些操作如何执行：

```java
Stream.of("d2", "a2", "b1", "b3", "c")
    .map(s -> {
        System.out.println("map: " + s);
        return s.toUpperCase();
    })
    .filter(s -> {
        System.out.println("filter: " + s);
        return s.startsWith("A");
    })
    .forEach(s -> System.out.println("forEach: " + s));

// map:     d2
// filter:  D2
// map:     a2
// filter:  A2
// forEach: A2
// map:     b1
// filter:  B1
// map:     b3
// filter:  B3
// map:     c
// filter:  C
```

就像你可能猜到的那样，`map`和`filter`会对底层集合的每个字符串调用五次，而`forEach`只会调用一次。

如果我们调整操作顺序，将`filter`移动到调用链的顶端，就可以极大减少操作的执行次数:

```java
Stream.of("d2", "a2", "b1", "b3", "c")
    .filter(s -> {
        System.out.println("filter: " + s);
        return s.startsWith("a");
    })
    .map(s -> {
        System.out.println("map: " + s);
        return s.toUpperCase();
    })
    .forEach(s -> System.out.println("forEach: " + s));

// filter:  d2
// filter:  a2
// map:     a2
// forEach: A2
// filter:  b1
// filter:  b3
// filter:  c
```

现在，`map`只会调用一次，所以操作流水线对于更多的输入元素会执行更快。在整合复杂的方法链时，要记住这一点。

让我们通过添加额外的方法`sorted`来扩展上面的例子：

```java
Stream.of("d2", "a2", "b1", "b3", "c")
    .sorted((s1, s2) -> {
        System.out.printf("sort: %s; %s\n", s1, s2);
        return s1.compareTo(s2);
    })
    .filter(s -> {
        System.out.println("filter: " + s);
        return s.startsWith("a");
    })
    .map(s -> {
        System.out.println("map: " + s);
        return s.toUpperCase();
    })
    .forEach(s -> System.out.println("forEach: " + s));
```

排序是一类特殊的衔接操作。它是有状态的操作，因为你需要在处理中保存状态来对集合中的元素排序。

执行这个例子会得到如下输入：

```java
sort:    a2; d2
sort:    b1; a2
sort:    b1; d2
sort:    b1; a2
sort:    b3; b1
sort:    b3; d2
sort:    c; b3
sort:    c; d2
filter:  a2
map:     a2
forEach: A2
filter:  b1
filter:  b3
filter:  c
filter:  d2
```

首先，排序操作在整个输入集合上执行。也即是说，`sorted`以水平方式执行。所以这里`sorted`对输入集合中每个元素的多种组合调用了八次。

我们同样可以通过重排调用链来优化性能：

```java
Stream.of("d2", "a2", "b1", "b3", "c")
    .filter(s -> {
        System.out.println("filter: " + s);
        return s.startsWith("a");
    })
    .sorted((s1, s2) -> {
        System.out.printf("sort: %s; %s\n", s1, s2);
        return s1.compareTo(s2);
    })
    .map(s -> {
        System.out.println("map: " + s);
        return s.toUpperCase();
    })
    .forEach(s -> System.out.println("forEach: " + s));

// filter:  d2
// filter:  a2
// filter:  b1
// filter:  b3
// filter:  c
// map:     a2
// forEach: A2
```

这个例子中`sorted`永远不会调用，因为`filter`把输入集合减少至只有一个元素。所以对于更大的输入集合会极大提升性能。

## 复用数据流

Java8的数据流不能被复用。一旦你调用了任何终止操作，数据流就关闭了：

```java
Stream<String> stream =
    Stream.of("d2", "a2", "b1", "b3", "c")
        .filter(s -> s.startsWith("a"));

stream.anyMatch(s -> true);    // ok
stream.noneMatch(s -> true);   // exception
```

在相同数据流上，在`anyMatch`之后调用`noneMatch`会产生下面的异常：

```
java.lang.IllegalStateException: stream has already been operated upon or closed
    at java.util.stream.AbstractPipeline.evaluate(AbstractPipeline.java:229)
    at java.util.stream.ReferencePipeline.noneMatch(ReferencePipeline.java:459)
    at com.winterbe.java8.Streams5.test7(Streams5.java:38)
    at com.winterbe.java8.Streams5.main(Streams5.java:28)
```

要克服这个限制，我们需要为每个我们想要执行的终止操作创建新的数据流调用链。例如，我们创建一个数据流提供者，来构建新的数据流，并且设置好所有衔接操作：

```java
Supplier<Stream<String>> streamSupplier =
    () -> Stream.of("d2", "a2", "b1", "b3", "c")
            .filter(s -> s.startsWith("a"));

streamSupplier.get().anyMatch(s -> true);   // ok
streamSupplier.get().noneMatch(s -> true);  // ok
```

