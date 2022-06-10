# Job 和等待子协程

在_结构化并发_一章中，我们提到父子关系会有以下影响：

* 子协程从父协程那里继承上下文
* 父协程会挂起，直到所有子协程都完成
* 当父协程被取消时，所有子协程都将取消
* 当一个子协程被销毁时，它也会销毁父协程

事实上，子协程从父协程继承上下文是协程构建器行为的基本组成部分。

```kotlin
fun main(): Unit = runBlocking(CoroutineName("main")) {
    val name = coroutineContext[CoroutineName]?.name
    println(name) // main
    launch {
        delay(1000)
        val name = coroutineContext[CoroutineName]?.name
        println(name) // main
    }
}
```

而结构化并发的其它三个重要的影响完全依赖 `Job` 上下文。此外，`Job` 还可以用来取消协程、追踪协程的状态等等。它是非常重要且有用的，所以本章和接下来的两章将专门讨论 `Job` 上下文以及与之相关的基本协程机制。

### 什么是 Job?

从概念上讲，一个 `Job` 表示具有生命周期的、可以取消的东西。从形态上将，`Job` 是一个接口，但是它有具体的合约和状态，所以它可以被当做一个抽象类来看待。

A job lifecycle is represented by its state. Here is a graph of states and the transitions between them:

`Job` 的生命周期由状态表示，下面是其状态机图示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/57f0f3f683ac4f2984b79e841d28da4e.png)

在 “Active” 状态下，一个 `Job` 正在运行并执行它的工作。如果 job 是通过协程构建器创建的，这个状态就是协程主体运行时的状态。在这种状态下，我们可以启动子协程。大多数协程会在 “Active” 状态下启动。只有那些延迟启动的才会以 “New” 状态启动。当它完成时候，它的状态变为 “Completing”，等待所有子协程完成。一旦它的所有子协程任务都完成了，其状态就会变为 “Completed”，这是一个最终状态。或者，如果 job 在运行时候（在 “Active” 或者 “Completing” 状态下）取消或失败，其状态将会改变成为 “Cancelling”。在这种状态下，我们会有最后的时机来做一些清理，比如关闭连接或释放资源（我们将在下一章中看到如何做）。完成此操作后， `Job` 将会进入到 “Cancelled” 状态。

状态会在 job 的 `toString` 中展示。在下面例子中，我们看到不同的 job 随着其状态的变化而变化。最后一个是惰性启动的，这意味着它不会自己启动。而其它的一旦被创建将立即变成 “Active” 状态：

下面的代码展示了不同状态的 `Job`。我使用 `join` 来等待协程完成，这将在后面解释：

```kotlin
suspend fun main() = coroutineScope {
    // Job 在创建后就是 Active 状态
    val job = Job()
    println(job) // JobImpl{Active}@ADD

    // 直到我们调用函数让它完成
    job.complete()
    println(job) // JobImpl{Completed}@ADD

    //  launch 初始化后默认是 active 状态
    val activeJob = launch {
        delay(1000)
    }
    println(activeJob) // StandaloneCoroutine{Active}@ADD

    // 我们可以这样子等待 Job 完成
    activeJob.join() // (1 sec)
    println(activeJob) // StandaloneCoroutine{Completed}@ADD
    
    // launch 延迟初始化后状态为 New
    val lazyJob = launch(start = CoroutineStart.LAZY) {
        delay(1000)
    }
    println(lazyJob) // LazyStandaloneCoroutine{New}@ADD
    
    // 我们需要调用 start 方法，激活它
    lazyJob.start()
    println(lazyJob) // LazyStandaloneCoroutine{Active}@ADD

    lazyJob.join() // (1 sec)
    println(lazyJob) //LazyStandaloneCoroutine{Completed}@ADD
}
```

为了在代码中检查 job 的状态，我们可以使用属性 `isActive`、 `isCompleted` 和 `isCancelled`。

| State           | isActive | isCompleted | isCancelled |
| --------------- | -------- | ----------- | ----------- |
| New (可选择初始化的状态) | false    | false       | false       |
| Active(默认初始化状态) | true     | false       | false       |
| Completing(中间态) | true     | false       | false       |
| Cancelling(中间态) | false    | false       | true        |
| Cancelled(最终态)  | false    | true        | true        |
| Completed(最终态)  | false    | true        | false       |

如前面提到的，每个协程都有自己的 job。让我们看看如何访问和使用它。

### 协程构建器基于其父 Job 构建其 Job

来自 Kotlin 协程库的每个协程构建器都会创建其它们自己的 job。大多数协程构建器会返回它们的 job，所以这些 job 可以在其它地方被使用。比如 `launch` 是比较明显的， 它会显式返回 `Job` 类型对象：

```kotlin
fun main(): Unit = runBlocking {
    val job: Job = launch {
        delay(1000)
        println("Test")
    }
}
```

`async` 函数返回的类型是 `Deferred<T>`，而它实现了 `Job` 接口，因此可以用相同的方式使用它：

```kotlin
fun main(): Unit = runBlocking {
    val deferred: Deferred<String> = async {
        delay(1000)
        "Test"
    }
    val job: Job = deferred
}
```

由于 `Job` 是一个协程上下文，我们可以使用 `coroutinContext[Job]` 来访问它。同时还有一个扩展属性 job，它使得我们更容易的访问 job：

```kotlin
// 扩展属性
val CoroutineContext.job: Job
    get() = get(Job) ?: error("Current context doesn't...")

// 使用
fun main(): Unit = runBlocking {
    print(coroutineContext.job.isActive) // true
}
```

有一个非常重要的规则： `Job` 是唯一一个不是子协程直接继承父协程的上下文。每个协程都会创建自己的 `Job`，来自传递参数或者父协程的 job 将会被用作这个子协程所创建 job 的父 job。

```kotlin
fun main(): Unit = runBlocking {
    val name = CoroutineName("Some name")
    val job = Job()

    launch(name + job) {
        val childName = coroutineContext[CoroutineName]
        println(childName == name) // true
        val childJob = coroutineContext[Job]
        println(childJob == job) // false
        println(childJob == job.children.first()) // true
    }
}
```

父上下文可以引用它的所有子上下文，同样子也可以引用父。这种父子关系允许我们在协程范围内实现取消和异常处理。

```kotlin
fun main(): Unit = runBlocking {
    val job: Job = launch {
        delay(1000)
    }

    val parentJob: Job = coroutineContext.job
    // or coroutineContext[Job]!!
    println(job == parentJob) // false
    val parentChildren: Sequence<Job> = parentJob.children
    println(parentChildren.first() == job) // true
}
```

如果新的 `Job` 上下文取代了父 `Job` 的上下文，结构化并发机制将不起作用。为了展示这一点，我们可以使用 `Job()` 工厂函数，它创建了一个 `Job` 上下文（这将在后面解释）。

```kotlin
fun main(): Unit = runBlocking {
    launch(Job()) { // 使用新 job 取代了来自父协程的 job
        delay(1000)
        println("Will not be printed")
    }
}
// (不会打印任何东西，程序会马上结束))
```

在上面的例子中，父协程将不会等待子协程，因为它与子协程没有建立关系。这是因为子协程使用来自参数的 `Job` 作为父 `Job`，因此它与 `runBlocking` 没有关系。

当一个协程有它自己的（独立的） `Job` 时，它几乎与它的父协程没有任何联系。相当于它继承了其它的上下文，所以父子关系将不会适用。这会导致我们失去结构化并发，这是一个应该避免的情况。

### 等待子协程

`Job` 的第一个重要优势是它可以用来等待，直到所有协程完成。为此，我们使用 `join` 方法。 这是一个挂起函数，它挂起直到每个具体的子 `Job` 达到最终状态（Completed 或者 Cancelled）。

```kotlin
fun main(): Unit = runBlocking {
    val job1 = launch {
        delay(1000)
        println("Test1")
    }
    val job2 = launch {
        delay(2000)
        println("Test2")
    }
    job1.join()
    job2.join()
    println("All tests are done")
}
// (1 sec)
// Test1
// (1 sec)
// Test2
// All tests are done
```

`Job` 接口还暴露了一个 `children` 属性，允许我们访问它的所有子 job。我们不妨使用它来等待，直到所有的子 job 都进行最终状态。

```kotlin
fun main(): Unit = runBlocking {
    launch {
        delay(1000)
        println("Test1")
    }
    launch {
        delay(2000)
        println("Test2")
    }
    
    val children = coroutineContext[Job]
        ?.children
    val childrenNum = children?.count()
    println("Number of children: $childrenNum")
    children?.forEach { it.join() }
    println("All tests are done")
}
// Number of children: 2
// (1 sec)
// Test1
// (1 sec)
// Test2
// All tests are done
```

### Job 工厂方法

我们可以使用 `Job()` 工厂方法在没有协程的情况下创建一个 `Job`。它创建的 job 和任意协程都不关联，可以用作上下文。这也意味着我们可以使用这样的 job 作为许多协程的父级 job。

一个常见的错误是使用 `Job()` 来创建一个 job，将其用作某些协程的父协程，然后调用 job 的 `join` 函数。这样的程序永远不会结束，因为 job 将一直处于活动状态，即使它的所有子协程都完成了。这是因为这个上下文仍然可能被其它协程使用。

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = Job()
    launch(job) { // 使用新的 job 替换原有的父 job
        delay(1000)
        println("Text 1")
    }
    launch(job) { // 使用新的 job 替换原有的父 job
        delay(2000)
        println("Text 2")
    }
    job.join() // 在这里我们将永远地等待
    println("Will not be printed")
}
// (1 sec)
// Text 1
// (1 sec)
// Text 2
// (runs forever)
```

一个更好的做法是将 job 所有的子协程 `join` 起来（收敛）：

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = Job()
    launch(job) { // 使用新的 job 替换原有的父 job
        delay(1000)
        println("Text 1")
    }
    launch(job) { // 使用新的 job 替换原有的父 job
        delay(2000)
        println("Text 2")
    }
    job.children.forEach { it.join() }
}
// (1 sec)
// Text 1
// (1 sec)
// Text 2
```

`Job()` 是工厂函数的一个很好的例子。起初，你可能认为正在调用 `Job` 的构造函数，但随后可能会意识到， `Job` 是一个接口，接口是不能有构造函数的。实际上它是一个伪构造函数 —— 看起来像构造函数的简单函数。而且，这个函数返回类型不是一个 `Job`，而是它的子接口 `CompletableJob`。

```kotlin
public fun Job(parent: Job? = null): CompletableJob
```

通过提供两个额外的方法， `CompletableJob` 扩展了 Job 的接口功能：

* `complete(): Boolean` —— 用于完成 job，一旦它被调用，所有子协程将继续进行，直到它们全部完成，但新的协程不能在这个 job 中启动。如果该 job 完成，结果会返回true，否则是false（如果它已经是完成的了）。

```kotlin
fun main() = runBlocking {
    val job = Job()
    launch(job) {
        repeat(5) { num ->
            delay(200)
            println("Rep$num")
        }
    }
    launch {
        delay(500)
        job.complete()
    }
    job.join()
    launch(job) {
        println("Will not be printed")
    }
    println("Done")
}
// Rep0
// Rep1
// Rep2
// Rep3
// Rep4
// Done
```

* `completeExceptionally(exception: Throwable): Boolean` —— 使用一个异常来结束一个 job。这意味着所有的子协程将**立即被取消**（`CancellationException` 包装了异常信息）

```kotlin
fun main() = runBlocking {
    val job = Job()
    launch(job) {
        repeat(5) { num ->
            delay(200)
            println("Rep$num")
        }
    }
    launch {
        delay(500)
        job.completeExceptionally(Error("Some error"))
    }
    job.join()
        launch(job) {
        println("Will not be printed")
    }
    println("Done")
}
// Rep0
// Rep1
// Done
```

当我们在一个 job 上启动最后一个协程后，通常会调用 `complete()` 函数，因此，我们可以使用 `join` 函数来等待函数完成。

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = Job()
    launch(job) { // 使用新的 job 替换原有的父 job
        delay(1000)
        println("Text 1")
    }
    launch(job) { // 使用新的 job 替换原有的父 job
       delay(2000)
       println("Text 2")
    }
    job.complete()
    job.join()
}
// (1 sec)
// Text 1
// (1 sec)
// Text 2
```

可以用父 job 作为 `Job()` 的参数传递构建子 job，由于这个原因，这样的 job 可以在父 job 取消时也被取消。

```kotlin
suspend fun main(): Unit = coroutineScope {
    val parentJob = Job()
    val job = Job(parentJob)
    launch(job) {
        delay(1000)
        println("Text 1")
    }
    launch(job) {
        delay(2000)
        println("Text 2")
    }
    delay(1100)
    parentJob.cancel()
    job.children.forEach { it.join() }
}
// Text 1
```

接下来的两章将描述 Kotlin 协程中的取消和异常处理。这两个重要的机制完全依赖于使用 Job 缔造的父子关系。
