# 使用 Intellij IDEA 解决 Java 8 的数据流问题

> 原文：[Fixing Java 8 Stream Gotchas with IntelliJ IDEA](http://winterbe.com/posts/2015/03/05/fixing-java-8-stream-gotchas-with-intellij-idea/) 

> 译者：[飞龙](https://github.com/wizardforcel)  

> 协议：[CC BY-NC-SA 4.0](http://creativecommons.org/licenses/by-nc-sa/4.0/)

Java8在2014年三月发布，距离现在（2015年三月五号）快有一年了。我们打算将[Pondus](http://www.pondus.de/)的所有生产服务器升级到这一新版本。从那时起，我们将大部分代码库迁移到[lambda表达式](ch1.md)、[数据流](ch2.md)和新的日期API上。我们也会使用[Nashorn](ch3.md)来把我们的应用中运行时发生改变的部分变成动态脚本。

除了lambda，最实用的特性是新的数据流API。集合操作在任何我见过的代码库中都随处可见。而且对于那些集合操作，数据流是提升代码可读性的好方法。

但是一件关于数据流的事情十分令我困扰：数据流只提供了几个终止操作，例如`reduce`和`findFirst`属于直接操作，其它的只能通过`collect`来访问。工具类`Collctors`提供了一些便利的收集器，例如`toList`、`toSet`、`joining`和`groupingBy`。

例如，下面的代码对一个字符串集合进行过滤，并创建新的列表：

```java
stringCollection
    .stream()
    .filter(e -> e.startsWith("a"))
    .collect(Collectors.toList());
```

在迁移了300k行代码到数据流之后，我可以说，`toList`、`toSet`、和`groupingBy`是你的项目中最常用的终止操作。所以我不能理解为什么不把这些方法直接集成到`Stream`接口上面，这样你就可以直接编写：

```java
stringCollection
    .stream()
    .filter(e -> e.startsWith("a"))
    .toList();
```

这在开始看起来是个小缺陷，但是如果你需要一遍又一遍地编写这些代码，它会非常烦人。

有`toArray()`方法但是没有`toList()`，所以我真心希望一些便利的收集器可以在Java9中这样添加到`Stream`接口中。是吧，[Brian](https://twitter.com/briangoetz)？ಠ_ಠ

> 注：[Stream.js](https://github.com/winterbe/streamjs)是浏览器上的Java 8 数据流API的JavaScript接口，并解决了上述问题。所有重要的终止操作都可以直接在流上访问，十分方便。详情请见[API文档](https://github.com/winterbe/streamjs/blob/master/APIDOC.md#groupingbykeymapper)。

无论如何，[IntelliJ IDEA](https://www.jetbrains.com/idea/)声称它是最智能的Java IDE。所以让我们看看如何使用IDEA来解决这一问题。

## 使用 IntelliJ IDEA 来帮忙

IntelliJ IDEA自带了一个便利的特性，叫做实时模板（Live Template）。如果你还不知道它是什么：实时模板是一些常用代码段的快捷方式。例如，你键入`sout`并按下TAB键，IDEA就会插入代码段`System.out.println()`。更多信息请见[这里](https://www.jetbrains.com/idea/help/live-templates.html)。

如何用实时模板来解决上述问题？实际上我们只需要为所有普遍使用的默认数据流收集器创建我们自己的实时模板。例如，我们可以创建`.toList`缩写的实时模板，来自动插入适当的收集器`.collect(Collectors.toList())`。

下面是它在实际工作中的样子：

![](http://winterbe.com/image/posts/livetemplate-streams1.gif)

## 构建你自己的实时模板

让我们看看如何自己构建它。首先访问设置（Settings）并在左侧的菜单中选择实时模板。你也可以使用对话框左上角的便利的输入过滤。

![](http://winterbe.com/image/posts/livetemplate-settings.png)

下面我们可以通过右侧的`+`图标创建一个新的组，叫做`Stream`。接下来我们向组中添加所有数据流相关的实时模板。我经常使用默认的收集器`toList`、`toSet`、`groupingBy` 和 `join`，所以我为每个这些方法都创建了新的实时模板。

这一步非常重要。在添加新的实时模板之后，你需要在对话框底部指定合适的上下文。你需要选择`Java → Other`，然后定义缩写、描述和实际的模板代码。

```java
// Abbreviation: .toList
.collect(Collectors.toList())

// Abbreviation: .toSet
.collect(Collectors.toSet())

// Abbreviation: .join
.collect(Collectors.joining("$END$"))

// Abbreviation: .groupBy
.collect(Collectors.groupingBy(e -> $END$))
```

特殊的变量`$END$`指定在使用模板之后的光标位置，所以你可以直接在这个位置上打字，例如，定义连接分隔符。

> 提示：你应该开启"Add unambiguous imports on the fly"（自动添加明确的导入）选项，便于让IDEA自动添加`java.util.stream.Collectors`的导入语句。选项在`Editor → General → Auto Import`中。

让我们在实际工作中看看这两个模板：

### 连接

![](http://winterbe.com/image/posts/livetemplate-streams2.gif)

### 分组

![](http://winterbe.com/image/posts/livetemplate-streams3.gif)

Intellij IDEA中的实时模板非常灵活且强大。你可以用它来极大提升代码的生产力。你知道实时模板可以拯救生活的其它例子吗？[请让我知道](http://winterbe.com/contact/)！

仍然不满意吗？在我的[数据流教程](ch1.md)中学习所有你想要学到的东西。

祝编程愉快！
