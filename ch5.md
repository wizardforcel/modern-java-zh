# Java 8 并发教程：同步和锁

> 原文：[Java 8 Concurrency Tutorial: Synchronization and Locks](http://winterbe.com/posts/2015/04/30/java8-concurrency-tutorial-synchronized-locks-examples/)

> 译者：[飞龙](https://github.com/wizardforcel)  

> 协议：[CC BY-NC-SA 4.0](http://creativecommons.org/licenses/by-nc-sa/4.0/)

欢迎阅读我的Java8并发教程的第二部分。这份指南将会以简单易懂的代码示例来教给你如何在Java8中进行并发编程。这是一系列教程中的第二部分。在接下来的15分钟，你将会学会如何通过同步关键字，锁和信号量来同步访问共享可变变量。

+ 第一部分：[线程和执行器](ch4.md)
+ 第二部分：[同步和锁](ch5.md)
+ 第三部分：[原子变量和 ConcurrentMap](ch6.md)

这篇文章中展示的中心概念也适用于Java的旧版本，然而代码示例适用于Java 8，并严重依赖于lambda表达式和新的并发特性。如果你还不熟悉lambda，我推荐你先阅读我的[Java 8 教程](ch1.md)。

出于简单的因素，这个教程的代码示例使用了定义在[这里](https://github.com/winterbe/java8-tutorial/blob/master/src/com/winterbe/java8/samples/concurrent/ConcurrentUtils.java)的两个辅助函数`sleep(seconds)` 和 `stop(executor)`。

## 同步

在[上一章](ch4.md)中，我们学到了如何通过执行器服务同时执行代码。当我们编写这种多线程代码时，我们需要特别注意共享可变变量的并发访问。假设我们打算增加某个可被多个线程同时访问的整数。

我们定义了`count`字段，带有`increment()`方法来使`count`加一：

```java
int count = 0;

void increment() {
    count = count + 1;
}
```

当多个线程并发调用这个方法时，我们就会遇到大麻烦：

```java
ExecutorService executor = Executors.newFixedThreadPool(2);

IntStream.range(0, 10000)
    .forEach(i -> executor.submit(this::increment));

stop(executor);

System.out.println(count);  // 9965
```

我们没有看到`count`为10000的结果，上面代码的实际结果在每次执行时都不同。原因是我们在不同的线程上共享可变变量，并且变量访问没有同步机制，这会产生[竞争条件](http://en.wikipedia.org/wiki/Race_condition)。

增加一个数值需要三个步骤：（1）读取当前值，（2）使这个值加一，（3）将新的值写到变量。如果两个线程同时执行，就有可能出现两个线程同时执行步骤1，于是会读到相同的当前值。这会导致无效的写入，所以实际的结果会偏小。上面的例子中，对`count`的非同步并发访问丢失了35次增加操作，但是你在自己执行代码时会看到不同的结果。

幸运的是，Java自从很久之前就通过`synchronized`关键字支持线程同步。我们可以使用`synchronized`来修复上面在增加`count`时的竞争条件。

```java
synchronized void incrementSync() {
    count = count + 1;
}
```

在我们并发调用`incrementSync()`时，我们得到了`count`为10000的预期结果。没有再出现任何竞争条件，并且结果在每次代码执行中都很稳定：

```java
ExecutorService executor = Executors.newFixedThreadPool(2);

IntStream.range(0, 10000)
    .forEach(i -> executor.submit(this::incrementSync));

stop(executor);

System.out.println(count);  // 10000
```

`synchronized`关键字也可用于语句块：

```java
void incrementSync() {
    synchronized (this) {
        count = count + 1;
    }
}
```

Java在内部使用所谓的“监视器”（monitor），也称为监视器锁（monitor lock）或内在锁（ intrinsic lock）来管理同步。监视器绑定在对象上，例如，当使用同步方法时，每个方法都共享相应对象的相同监视器。

所有隐式的监视器都实现了重入（reentrant）特性。重入的意思是锁绑定在当前线程上。线程可以安全地多次获取相同的锁，而不会产生死锁（例如，同步方法调用相同对象的另一个同步方法）。

## 锁

并发API支持多种显式的锁，它们由`Lock`接口规定，用于代替`synchronized`的隐式锁。锁对细粒度的控制支持多种方法，因此它们比隐式的监视器具有更大的开销。

锁的多个实现在标准JDK中提供，它们会在下面的章节中展示。

### `ReentrantLock`

`ReentrantLock`类是互斥锁，与通过`synchronized`访问的隐式监视器具有相同行为，但是具有扩展功能。就像它的名称一样，这个锁实现了重入特性，就像隐式监视器一样。

让我们看看使用`ReentrantLock`之后的上面的例子。

```java
ReentrantLock lock = new ReentrantLock();
int count = 0;

void increment() {
    lock.lock();
    try {
        count++;
    } finally {
        lock.unlock();
    }
}
```

锁可以通过`lock()`来获取，通过`unlock()`来释放。把你的代码包装在`try-finally`代码块中来确保异常情况下的解锁非常重要。这个方法是线程安全的，就像同步副本那样。如果另一个线程已经拿到锁了，再次调用`lock()`会阻塞当前线程，直到锁被释放。在任意给定的时间内，只有一个线程可以拿到锁。

锁对细粒度的控制支持多种方法，就像下面的例子那样：

```java
executor.submit(() -> {
    lock.lock();
    try {
        sleep(1);
    } finally {
        lock.unlock();
    }
});

executor.submit(() -> {
    System.out.println("Locked: " + lock.isLocked());
    System.out.println("Held by me: " + lock.isHeldByCurrentThread());
    boolean locked = lock.tryLock();
    System.out.println("Lock acquired: " + locked);
});

stop(executor);
```

在第一个任务拿到锁的一秒之后，第二个任务获得了锁的当前状态的不同信息。

```
Locked: true
Held by me: false
Lock acquired: false
```

`tryLock()`方法是`lock()`方法的替代，它尝试拿锁而不阻塞当前线程。在访问任何共享可变变量之前，必须使用布尔值结果来检查锁是否已经被获取。

### `ReadWriteLock`

`ReadWriteLock`接口规定了锁的另一种类型，包含用于读写访问的一对锁。读写锁的理念是，只要没有任何线程写入变量，并发读取可变变量通常是安全的。所以读锁可以同时被多个线程持有，只要没有线程持有写锁。这样可以提升性能和吞吐量，因为读取比写入更加频繁。

```java
ExecutorService executor = Executors.newFixedThreadPool(2);
Map<String, String> map = new HashMap<>();
ReadWriteLock lock = new ReentrantReadWriteLock();

executor.submit(() -> {
    lock.writeLock().lock();
    try {
        sleep(1);
        map.put("foo", "bar");
    } finally {
        lock.writeLock().unlock();
    }
});
```

上面的例子在暂停一秒之后，首先获取写锁来向映射添加新的值。在这个任务完成之前，两个其它的任务被启动，尝试读取映射中的元素，并暂停一秒：

```java
Runnable readTask = () -> {
    lock.readLock().lock();
    try {
        System.out.println(map.get("foo"));
        sleep(1);
    } finally {
        lock.readLock().unlock();
    }
};

executor.submit(readTask);
executor.submit(readTask);

stop(executor);
```

当你执行这一代码示例时，你会注意到两个读任务需要等待写任务完成。在释放了写锁之后，两个读任务会同时执行，并同时打印结果。它们不需要相互等待完成，因为读锁可以安全同步获取，只要没有其它线程获取了写锁。

### `StampedLock`

Java 8 自带了一种新的锁，叫做`StampedLock`，它同样支持读写锁，就像上面的例子那样。与`ReadWriteLock`不同的是，`StampedLock`的锁方法会返回表示为`long`的标记。你可以使用这些标记来释放锁，或者检查锁是否有效。此外，`StampedLock`支持另一种叫做乐观锁（optimistic locking）的模式。

让我们使用`StampedLock`代替`ReadWriteLock`重写上面的例子：

```java
ExecutorService executor = Executors.newFixedThreadPool(2);
Map<String, String> map = new HashMap<>();
StampedLock lock = new StampedLock();

executor.submit(() -> {
    long stamp = lock.writeLock();
    try {
        sleep(1);
        map.put("foo", "bar");
    } finally {
        lock.unlockWrite(stamp);
    }
});

Runnable readTask = () -> {
    long stamp = lock.readLock();
    try {
        System.out.println(map.get("foo"));
        sleep(1);
    } finally {
        lock.unlockRead(stamp);
    }
};

executor.submit(readTask);
executor.submit(readTask);

stop(executor);
```

通过`readLock()` 或 `writeLock()`来获取读锁或写锁会返回一个标记，它可以在稍后用于在`finally`块中解锁。要记住`StampedLock`并没有实现重入特性。每次调用加锁都会返回一个新的标记，并且在没有可用的锁时阻塞，即使相同线程已经拿锁了。所以你需要额外注意不要出现死锁。

就像前面的`ReadWriteLock`例子那样，两个读任务都需要等待写锁释放。之后两个读任务同时向控制台打印信息，因为多个读操作不会相互阻塞，只要没有线程拿到写锁。

下面的例子展示了乐观锁：

```java
ExecutorService executor = Executors.newFixedThreadPool(2);
StampedLock lock = new StampedLock();

executor.submit(() -> {
    long stamp = lock.tryOptimisticRead();
    try {
        System.out.println("Optimistic Lock Valid: " + lock.validate(stamp));
        sleep(1);
        System.out.println("Optimistic Lock Valid: " + lock.validate(stamp));
        sleep(2);
        System.out.println("Optimistic Lock Valid: " + lock.validate(stamp));
    } finally {
        lock.unlock(stamp);
    }
});

executor.submit(() -> {
    long stamp = lock.writeLock();
    try {
        System.out.println("Write Lock acquired");
        sleep(2);
    } finally {
        lock.unlock(stamp);
        System.out.println("Write done");
    }
});

stop(executor);
```

乐观的读锁通过调用`tryOptimisticRead()`获取，它总是返回一个标记而不阻塞当前线程，无论锁是否真正可用。如果已经有写锁被拿到，返回的标记等于0。你需要总是通过`lock.validate(stamp)`检查标记是否有效。

执行上面的代码会产生以下输出：

```
Optimistic Lock Valid: true
Write Lock acquired
Optimistic Lock Valid: false
Write done
Optimistic Lock Valid: false
```

乐观锁在刚刚拿到锁之后是有效的。和普通的读锁不同的是，乐观锁不阻止其他线程同时获取写锁。在第一个线程暂停一秒之后，第二个线程拿到写锁而无需等待乐观的读锁被释放。此时，乐观的读锁就不再有效了。甚至当写锁释放时，乐观的读锁还处于无效状态。

所以在使用乐观锁时，你需要每次在访问任何共享可变变量之后都要检查锁，来确保读锁仍然有效。

有时，将读锁转换为写锁而不用再次解锁和加锁十分实用。`StampedLock`为这种目的提供了`tryConvertToWriteLock()`方法，就像下面那样：

```java
ExecutorService executor = Executors.newFixedThreadPool(2);
StampedLock lock = new StampedLock();

executor.submit(() -> {
    long stamp = lock.readLock();
    try {
        if (count == 0) {
            stamp = lock.tryConvertToWriteLock(stamp);
            if (stamp == 0L) {
                System.out.println("Could not convert to write lock");
                stamp = lock.writeLock();
            }
            count = 23;
        }
        System.out.println(count);
    } finally {
        lock.unlock(stamp);
    }
});

stop(executor);
```

第一个任务获取读锁，并向控制台打印`count`字段的当前值。但是如果当前值是零，我们希望将其赋值为`23`。我们首先需要将读锁转换为写锁，来避免打破其它线程潜在的并发访问。`tryConvertToWriteLock()`的调用不会阻塞，但是可能会返回为零的标记，表示当前没有可用的写锁。这种情况下，我们调用`writeLock()`来阻塞当前线程，直到有可用的写锁。

## 信号量

除了锁之外，并发API也支持计数的信号量。不过锁通常用于变量或资源的互斥访问，信号量可以维护整体的准入许可。这在一些不同场景下，例如你需要限制你程序某个部分的并发访问总数时非常实用。

下面是一个例子，演示了如何限制对通过`sleep(5)`模拟的长时间运行任务的访问：

```java
ExecutorService executor = Executors.newFixedThreadPool(10);

Semaphore semaphore = new Semaphore(5);

Runnable longRunningTask = () -> {
    boolean permit = false;
    try {
        permit = semaphore.tryAcquire(1, TimeUnit.SECONDS);
        if (permit) {
            System.out.println("Semaphore acquired");
            sleep(5);
        } else {
            System.out.println("Could not acquire semaphore");
        }
    } catch (InterruptedException e) {
        throw new IllegalStateException(e);
    } finally {
        if (permit) {
            semaphore.release();
        }
    }
}

IntStream.range(0, 10)
    .forEach(i -> executor.submit(longRunningTask));

stop(executor);
```

执行器可能同时运行10个任务，但是我们使用了大小为5的信号量，所以将并发访问限制为5。使用`try-finally`代码块在异常情况中合理释放信号量十分重要。

执行上述代码产生如下结果：

```
Semaphore acquired
Semaphore acquired
Semaphore acquired
Semaphore acquired
Semaphore acquired
Could not acquire semaphore
Could not acquire semaphore
Could not acquire semaphore
Could not acquire semaphore
Could not acquire semaphore
```

信号量限制对通过`sleep(5)`模拟的长时间运行任务的访问，最大5个线程。每个随后的`tryAcquire()`调用在经过最大为一秒的等待超时之后，会向控制台打印不能获取信号量的结果。

这就是我的系列并发教程的第二部分。以后会放出更多的部分，所以敬请等待吧。像以前一样，你可以在[Github](https://github.com/winterbe/java8-tutorial)上找到这篇文档的所有示例代码，所以请随意fork这个仓库，并自己尝试它。

我希望你能喜欢这篇文章。如果你还有任何问题，在下面的评论中向我反馈。你也可以[在Twitter上关注我](https://twitter.com/winterbe_)来获取更多开发相关的信息。

+ 第一部分：[线程和执行器](ch4.md)
+ 第二部分：[同步和锁](ch5.md)
+ 第三部分：[原子变量和 ConcurrentMap](ch6.md)
