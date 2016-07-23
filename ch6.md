# Java 8 并发教程：原子变量和 ConcurrentMap

> 原文：[Java 8 Concurrency Tutorial: Synchronization and Locks](http://winterbe.com/posts/2015/04/30/java8-concurrency-tutorial-synchronized-locks-examples/)

> 译者：[飞龙](https://github.com/wizardforcel)  

> 协议：[CC BY-NC-SA 4.0](http://creativecommons.org/licenses/by-nc-sa/4.0/)

欢迎阅读我的Java8多线程编程系列教程的第三部分。这个教程包含并发API的两个重要部分：原子变量和`ConcurrentMap`。由于最近发布的Java8中的lambda表达式和函数式编程，二者都有了极大的改进。所有这些新特性会以一些简单易懂的代码示例来描述。希望你能喜欢。

+ 第一部分：[线程和执行器](ch4.md)
+ 第二部分：[同步和锁](ch5.md)
+ 第三部分：[原子变量和 ConcurrentMap](ch6.md)

出于简单的因素，这个教程的代码示例使用了定义在[这里](https://github.com/winterbe/java8-tutorial/blob/master/src/com/winterbe/java8/samples/concurrent/ConcurrentUtils.java)的两个辅助函数`sleep(seconds)` 和 `stop(executor)`。

## `AtomicInteger`

`java.concurrent.atomic`包包含了许多实用的类，用于执行原子操作。如果你能够在多线程中同时且安全地执行某个操作，而不需要`synchronized`关键字或[上一章](ch5.md)中的锁，那么这个操作就是原子的。

本质上，原子操作严重依赖于比较与交换（CAS），它是由多数现代CPU直接支持的原子指令。这些指令通常比同步块要快。所以在只需要并发修改单个可变变量的情况下，我建议你优先使用原子类，而不是[上一章](ch5.md)展示的锁。

> 译者注：对于其它语言，一些语言的原子操作用锁实现，而不是原子指令。

现在让我们选取一个原子类，例如`AtomicInteger`：

```java
AtomicInteger atomicInt = new AtomicInteger(0);

ExecutorService executor = Executors.newFixedThreadPool(2);

IntStream.range(0, 1000)
    .forEach(i -> executor.submit(atomicInt::incrementAndGet));

stop(executor);

System.out.println(atomicInt.get());    // => 1000
```

通过使用`AtomicInteger`代替`Integer`，我们就能线程安全地并发增加数值，而不需要同步访问变量。`incrementAndGet()`方法是原子操作，所以我们可以在多个线程中安全调用它。

`AtomicInteger`支持多种原子操作。`updateAndGet()`接受lambda表达式，以便在整数上执行任意操作：

```java
AtomicInteger atomicInt = new AtomicInteger(0);

ExecutorService executor = Executors.newFixedThreadPool(2);

IntStream.range(0, 1000)
    .forEach(i -> {
        Runnable task = () ->
            atomicInt.updateAndGet(n -> n + 2);
        executor.submit(task);
    });

stop(executor);

System.out.println(atomicInt.get());    // => 2000
```

`accumulateAndGet()`方法接受另一种类型`IntBinaryOperator`的lambda表达式。我们在下个例子中，使用这个方法并发计算0~1000所有值的和：

```java
AtomicInteger atomicInt = new AtomicInteger(0);

ExecutorService executor = Executors.newFixedThreadPool(2);

IntStream.range(0, 1000)
    .forEach(i -> {
        Runnable task = () ->
            atomicInt.accumulateAndGet(i, (n, m) -> n + m);
        executor.submit(task);
    });

stop(executor);

System.out.println(atomicInt.get());    // => 499500
```

其它实用的原子类有`AtomicBoolean`、`AtomicLong` 和 `AtomicReference`。

## `LongAdder`

`LongAdder`是`AtomicLong`的替代，用于向某个数值连续添加值。

```java
ExecutorService executor = Executors.newFixedThreadPool(2);

IntStream.range(0, 1000)
    .forEach(i -> executor.submit(adder::increment));

stop(executor);

System.out.println(adder.sumThenReset());   // => 1000
```

`LongAdder`提供了`add()`和`increment()`方法，就像原子数值类一样，同样是线程安全的。但是这个类在内部维护一系列变量来减少线程之间的争用，而不是求和计算单一结果。实际的结果可以通过调用`sum()`或`sumThenReset()`来获取。

当多线程的更新比读取更频繁时，这个类通常比原子数值类性能更好。这种情况在抓取统计数据时经常出现，例如，你希望统计Web服务器上请求的数量。`LongAdder`缺点是较高的内存开销，因为它在内存中储存了一系列变量。

## `LongAccumulator`

`LongAccumulator`是`LongAdder`的更通用的版本。`LongAccumulator`以类型为`LongBinaryOperator`lambda表达式构建，而不是仅仅执行加法操作，像这段代码展示的那样：

```java
LongBinaryOperator op = (x, y) -> 2 * x + y;
LongAccumulator accumulator = new LongAccumulator(op, 1L);

ExecutorService executor = Executors.newFixedThreadPool(2);

IntStream.range(0, 10)
    .forEach(i -> executor.submit(() -> accumulator.accumulate(i)));

stop(executor);

System.out.println(accumulator.getThenReset());     // => 2539
```

我们使用函数`2 * x + y`创建了`LongAccumulator`，初始值为1。每次调用`accumulate(i)`的时候，当前结果和值`i`都会作为参数传入lambda表达式。

`LongAccumulator`就像`LongAdder`那样，在内部维护一系列变量来减少线程之间的争用。

## `ConcurrentMap`

`ConcurrentMap`接口继承自`Map`接口，并定义了最实用的并发集合类型之一。Java8通过将新的方法添加到这个接口，引入了函数式编程。

在下面的代码中，我们使用这个映射示例来展示那些新的方法：

```java
ConcurrentMap<String, String> map = new ConcurrentHashMap<>();
map.put("foo", "bar");
map.put("han", "solo");
map.put("r2", "d2");
map.put("c3", "p0");
```

`forEach()`方法接受类型为`BiConsumer`的lambda表达式，以映射的键和值作为参数传递。它可以作为`for-each`循环的替代，来遍历并发映射中的元素。迭代在当前线程上串行执行。

```java
map.forEach((key, value) -> System.out.printf("%s = %s\n", key, value));
```

新方法`putIfAbsent()`只在提供的键不存在时，将新的值添加到映射中。至少在`ConcurrentHashMap`的实现中，这一方法像`put()`一样是线程安全的，所以你在不同线程中并发访问映射时，不需要任何同步机制。

```java
String value = map.putIfAbsent("c3", "p1");
System.out.println(value);    // p0
```

`getOrDefault()`方法返回指定键的值。在传入的键不存在时，会返回默认值：

```java
String value = map.getOrDefault("hi", "there");
System.out.println(value);    // there
```

`replaceAll()`接受类型为`BiFunction`的lambda表达式。`BiFunction`接受两个参数并返回一个值。函数在这里以每个元素的键和值调用，并返回要映射到当前键的新值。

```java
map.replaceAll((key, value) -> "r2".equals(key) ? "d3" : value);
System.out.println(map.get("r2"));    // d3
```

`compute()`允许我们转换单个元素，而不是替换映射中的所有值。这个方法接受需要处理的键，和用于指定值的转换的`BiFunction`。

```java
map.compute("foo", (key, value) -> value + value);
System.out.println(map.get("foo"));   // barbar
```

除了`compute()`之外还有两个变体：`computeIfAbsent()` 和 `computeIfPresent()`。这些方法的函数式参数只在键不存在或存在时被调用。

最后，`merge()`方法可以用于以映射中的现有值来统一新的值。这个方法接受键、需要并入现有元素的新值，以及指定两个值的合并行为的`BiFunction`。

```java
map.merge("foo", "boo", (oldVal, newVal) -> newVal + " was " + oldVal);
System.out.println(map.get("foo"));   // boo was foo
```

## `ConcurrentHashMap`

所有这些方法都是`ConcurrentMap`接口的一部分，因此可在所有该接口的实现上调用。此外，最重要的实现`ConcurrentHashMap`使用了一些新的方法来改进，便于在映射上执行并行操作。

就像并行流那样，这些方法使用特定的`ForkJoinPool`，由Java8中的`ForkJoinPool.commonPool()`提供。该池使用了取决于可用核心数量的预置并行机制。我的电脑有四个核心可用，这会使并行性的结果为3：

```java
System.out.println(ForkJoinPool.getCommonPoolParallelism());  // 3
```

这个值可以通过设置下列JVM参数来增减：

```
-Djava.util.concurrent.ForkJoinPool.common.parallelism=5
```

我们使用相同的映射示例来展示，但是这次我们使用具体的`ConcurrentHashMap`实现而不是`ConcurrentMap`接口，所以我们可以访问这个类的所有公共方法：

```java
ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();
map.put("foo", "bar");
map.put("han", "solo");
map.put("r2", "d2");
map.put("c3", "p0");
```

Java8引入了三种类型的并行操作：`forEach`、`search` 和 `reduce`。这些操作中每个都以四种形式提供，接受以键、值、元素或键值对为参数的函数。

所有这些方法的第一个参数是通用的`parallelismThreshold`。这一阈值表示操作并行执行时的最小集合大小。例如，如果你传入阈值500，而映射的实际大小是499，那么操作就会在单线程上串行执行。在下一个例子中，我们使用阈值1，始终强制并行执行来展示。

### `forEach`

`forEach()`方法可以并行迭代映射中的键值对。`BiConsumer`以当前迭代元素的键和值调用。为了将并行执行可视化，我们向控制台打印了当前线程的名称。要注意在我这里底层的`ForkJoinPool`最多使用三个线程。

```java
map.forEach(1, (key, value) ->
    System.out.printf("key: %s; value: %s; thread: %s\n",
        key, value, Thread.currentThread().getName()));

// key: r2; value: d2; thread: main
// key: foo; value: bar; thread: ForkJoinPool.commonPool-worker-1
// key: han; value: solo; thread: ForkJoinPool.commonPool-worker-2
// key: c3; value: p0; thread: main
```

### `search`

`search()`方法接受`BiFunction`并为当前的键值对返回一个非空的搜索结果，或者在当前迭代不匹配任何搜索条件时返回`null`。只要返回了非空的结果，就不会往下搜索了。要记住`ConcurrentHashMap`是无序的。搜索函数应该不依赖于映射实际的处理顺序。如果映射的多个元素都满足指定搜索函数，结果是非确定的。

```java
String result = map.search(1, (key, value) -> {
    System.out.println(Thread.currentThread().getName());
    if ("foo".equals(key)) {
        return value;
    }
    return null;
});
System.out.println("Result: " + result);

// ForkJoinPool.commonPool-worker-2
// main
// ForkJoinPool.commonPool-worker-3
// Result: bar
```

下面是另一个例子，仅仅搜索映射中的值：

```java
String result = map.searchValues(1, value -> {
    System.out.println(Thread.currentThread().getName());
    if (value.length() > 3) {
        return value;
    }
    return null;
});

System.out.println("Result: " + result);

// ForkJoinPool.commonPool-worker-2
// main
// main
// ForkJoinPool.commonPool-worker-1
// Result: solo
```

### `reduce`

`reduce()`方法已经在Java 8 的数据流之中用过了，它接受两个`BiFunction`类型的lambda表达式。第一个函数将每个键值对转换为任意类型的单一值。第二个函数将所有这些转换后的值组合为单一结果，并忽略所有可能的`null`值。

```java
String result = map.reduce(1,
    (key, value) -> {
        System.out.println("Transform: " + Thread.currentThread().getName());
        return key + "=" + value;
    },
    (s1, s2) -> {
        System.out.println("Reduce: " + Thread.currentThread().getName());
        return s1 + ", " + s2;
    });

System.out.println("Result: " + result);

// Transform: ForkJoinPool.commonPool-worker-2
// Transform: main
// Transform: ForkJoinPool.commonPool-worker-3
// Reduce: ForkJoinPool.commonPool-worker-3
// Transform: main
// Reduce: main
// Reduce: main
// Result: r2=d2, c3=p0, han=solo, foo=bar
```

我希望你能喜欢我的Java8并发系列教程的第三部分。这个教程的代码示例[托管在Github上](https://github.com/winterbe/java8-tutorial)，还有许多其它的Java8代码片段。欢迎fork我的仓库并自己尝试。

如果你想要支持我的工作，请向你的朋友分享这篇教程。你也可以[在Twiiter上关注我](https://twitter.com/winterbe_)，因为我会不断推送一些Java或编程相关的东西。

+ 第一部分：[线程和执行器](ch4.md)
+ 第二部分：[同步和锁](ch5.md)
+ 第三部分：[原子变量和 ConcurrentMap](ch6.md)
