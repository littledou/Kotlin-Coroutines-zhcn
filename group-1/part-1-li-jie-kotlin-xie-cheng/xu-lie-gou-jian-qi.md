# 序列构建器

在其它的一些语言中，如 Python 或 JavaScript，你可以找到一些有限形式的协程结构：

* `async` 函数（也称为 `async/await`）
* 生成器函数（用于产生后续值的函数）

我们已经看到了如何在 Kotlin 中使用异步操作，但这将会在之后的协程构建器中详细讲解。在 Kotlin 中没有使用生成器函数，而是提供了一个序列构建器 —— 用于创建序列的函数。

Kotlin 序列（Sequence）是一个类似于集合（如 `List` 或 `Set`）的概念，但它是惰性计算的，这意味着只有在需要时，才会计算下一个元素。它使序列拥有以下特性：

* 用最少的操作完成任务
* 可以是无限的
* 内存效率更高

由于这些特性，为序列定义一个构建器非常有意义，在这个构建器中，后续的元素都将按需计算，按需生成。我们用 `sequence` 函数来定义它，在它的 lambda 表达式中，我们可以调用 `yield` 函数来生成序列的下一个元素。

```kotlin
val seq = sequence {
    yield(1)
    yield(2)
    yield(3)
}

fun main() {
    for (num in seq) {
        print(num)
} // 123
}
```

这里的 `sequence` 函数是一个小型 DSL。它的参数是一个带有接收者的 lambda 表达式（`suspend SequenceScope<T>.() -> Unit`）。在 lambda 里面， 接收者 `this` 指向的是一个 `SequenceScope<T>` 类型的对象，它有 `yield` 这样的函数。当我们调用 `yeild(1)`，它相当于调用了 `this.yield(1)`，而 `this` 可以隐式调用。如果这是你第一次接触带有接收者的 lambda 表达式，我建议你从了解它们和 DSL 的创建开始，因为它们在 Kotlin 协程库中被大量使用。

最重要的是，每个数字都是根据需要生成的，而不是预先生成的。如果我们在构建器和处理序列的地方都打印一些日志，就能清楚的看到这个过程：

```kotlin
val seq = sequence {
    println("Generating first")
    yield(1)
    println("Generating second")
    yield(2)
    println("Generating third")
    yield(3)
    println("Done")
}

fun main() {
    for (num in seq) {
        println("The next number is $num")
    }
}

// Generating first
// The next number is 1
// Generating second
// The next number is 2
// Generating third
// The next number is 3
// Done
```

让我们来分析一下它是如何工作的。一开始我们请求第一个数字，因此我们进入构建器，打印了"Generating first"，得到的结果是1。然后我们返回处理值的 for 循环去，所以"Next Number is 1"被打印了出来。然后，关键的事情发生了：程序接下来跳到了我们之前停下来寻找另一个数字的地方。如果没有挂起机制，这将是不可能做到的。因为不可能在函数中间一个点停止，并在未来的同一点中恢复它。多亏了挂起，我们可以在 `main` 函数和序列构建器之间来回跳转。

![在这里插入图片描述](https://img-blog.csdnimg.cn/5bd5b63bdbed4c73bb3f8dc59a398639.png)

当我们请求序列的下一个值时，代码将直接在构建器的前一个 `yield` 之后恢复。

为了看的更加清晰，让我们手动从序列中请求一些值：

```kotlin
val seq = sequence {
    println("Generating first")
    yield(1)
    println("Generating second")
    yield(2)
    println("Generating third")
    yield(3)
    println("Done")
}

fun main() {
    val iterator = seq.iterator()
    println("Starting")
    val first = iterator.next()
    println("First: $first")
    val second = iterator.next()
    println("Second: $second")
    // ...
}

// 打印结果:
// Starting
// Generating first
// First: 1
// Generating second
// Second: 2
```

这段代码中，我们使用迭代器来获取下一个值。在任何时候，我们都可以再次调用它，以跳转到构建器函数的中间并生成下一个值。如果没有协程，这可能做到吗？如果我们要专门为它设计一个线程，或许可以。不过这样的线程需要维护，将会产生巨大的成本。使用协程，它是快速和简单的，此外，这个迭代器的开销几乎为零。我们可以想保留多久就保留多久。很快我们就知道这个机制是如何在底层工作的（在下一章节_挂起函数的底层运作_中讲解）。

### 实际使用

存在一些序列构建器的实际用例，最典型的是用来生成算数的序列，例如斐波那契数列。

```kotlin
val fibonacci: Sequence<BigInteger> = sequence {
    var first = 0.toBigInteger()
    var second = 1.toBigInteger()
    while (true) {
        yield(first)
        val temp = first
        first += second
        second = temp
    }
}

fun main() {
    print(fibonacci.take(10).toList())
}
// [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

此构建器还可用于生成随机数或文本：

```kotlin
fun randomNumbers(
    seed: Long = System.currentTimeMillis()
): Sequence<Int> = sequence {
    val random = Random(seed)
    while (true) {
        yield(random.nextInt())
    }
}

fun randomUniqueStrings(
    length: Int,
    seed: Long = System.currentTimeMillis()
): Sequence<String> = sequence {
    val random = Random(seed)
    val charPool = ('a'..'z') + ('A'..'Z') + ('0'..'9')
    while (true) {
        val randomString = (1..length)
        .map { i -> random.nextInt(charPool.size) }
        .map(charPool::get)
        .joinToString("");
        yield(randomString)
    }
}.distinct()
```

序列构建器除了生成操作以外，不应该挂起任何操作。假如，如果你需要拉取数据，最好使用 `Flow`，这将会在本书的后面解释。其构建器的工作方式与序列构建器类似，但 `Flow` 支持其他协程特性。

```kotlin
fun allUsersFlow(
    api: UserApi
): Flow<User> = flow {
    var page = 0
    do {
        val users = api.takePage(page++) // suspending
        emitAll(users)
    } while (!users.isNullOrEmpty())
}
```

我们已经了解了序列构建器以及为什么它需要挂起才能正确工作。现在我们已经看到了挂起的作用，是时候让我们更加深入了解挂起是如何在底层工作的了。
