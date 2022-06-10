# 协程的取消

Kotlin 协程一个非常重要的功能就是取消。一些类和库使用挂起函数主要是为了支持取消，这一点非常重要。其中有一个很好的理由：一个好的取消机制是名副其实的金子。仅仅终止线程是一个糟糕的方案，应该有一个更好的方案来关闭链接和释放资源。强迫开发人员频繁检查某些状态是否处于活跃状态并不便利。取消的问题持续了很长一段时间才有了一个好的解决方案，但是 Kotlin 协程库提供的方案却出奇的简单，它们既方便又安全。这是我职业生涯中见过最好的取消机制。让我们来探讨它。

### 基础取消

`Job` 接口有一个 `cancel` 方法，用于取消它，调用它会触发以下效果：

* 协程会在第一个挂起点结束 job （下面例子中的 `delay`）
* 如果一个 job 有几个子 job，它们也会被取消（但是它的父 job 不受影响）
* 一旦一个 job 被取消，它就不能被用作任何新 job 的父 job。它首先处于 “Cancelling” 状态，然后处于 “Cancelled” 状态

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = launch {
        repeat(1_000) { i ->
            delay(200)
            println("Printing $i")
        }
    }
    delay(1100)
    job.cancel()
    job.join()
    println("Cancelled successfully")
}
// Printing 0
// Printing 1
// Printing 2
// Printing 3
// Printing 4
// Cancelled successfully
```

取消可以使用不同的异常（通过将异常作为参数传递给 `cancel` 函数）来指定原因。这个原因需要是 `CancellationException` 的子类，因为只有这种类型的异常才能用来取消协程。

取消之后，我们通常会调用 `join` 方法，程序必须要等到“取消”执行完才能继续。如果没有这个函数，我们可能就会有一些别的竞争。下面代码展示了一个示例，在没有调用 `join` 的情况下，我们将会看到 “Printng4” 在 “Cancelled successfully” 后面：

```kotlin
suspend fun main() = coroutineScope {
    val job = launch {
        repeat(1_000) { i ->
            delay(100)
            Thread.sleep(100) // 我们模拟一些耗时操作
            println("Printing $i")
        }
    }
    delay(1000)
    job.cancel()
    println("Cancelled successfully")
}
// Printing 0
// Printing 1
// Printing 2
// Printing 3
// Cancelled successfully
// Printing 4
```

加上 `job.join` 将会改变这一点， 因为它会挂起，直到一个协程完成取消。

```kotlin
suspend fun main() = coroutineScope {
    val job = launch {
        repeat(1_000) { i ->
            delay(100)
            Thread.sleep(100) // We simulate long operation
            println("Printing $i")
        }
    }
    delay(1000)
    job.cancel()
    job.join()
    println("Cancelled successfully")
}
// Printing 0
// Printing 1
// Printing 2
// Printing 3
// Printing 4
// Cancelled successfully
```

为了更容易地同时调用 `cancel` 和 `join`， `kotlinx.coroutines` 提供了更方便的扩展函数： `cancelAndJoin`。

```kotlin
// 这可能是我见过的最明显的函数名
public suspend fun Job.cancelAndJoin() {
    cancel()
    return join()
}
```

使用 `Job()` 工厂函数创建的 job 可以以同样的方式被取消。这通常用于一次性取消多个协程。

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = Job()
    launch(job) {
        repeat(1_000) { i ->
            delay(200)
            println("Printing $i")
        }
    }
    delay(1100)
    job.cancelAndJoin()
    println("Cancelled successfully")
}
// Printing 0
// Printing 1
// Printing 2
// Printing 3
// Printing 4
// Cancelled successfully
```

这是一个至关重要的能力。在许多平台上，我们经常需要取消一组并发任务。例如，在 Android 中，当用户离开一个视图时，我们需要取消此视图启动的多个协程。

```kotlin
class ProfileViewModel : ViewModel() {
    private val scope =
        CoroutineScope(Dispatchers.Main + SupervisorJob())

    fun onCreate() {
        scope.launch { loadUserData() }
    }

    override fun onCleared() {
        scope.coroutineContext.cancelChildren()
    }
    // ...
}
```

### 取消如何工作

当一个 job 被取消时，它的状态变成 “Cancelling”，然后，在第一个挂起点，抛出一个 `CancellationException` 异常。可以使用 `try-catch` 来捕获这个异常，但我建议抛出它。

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = Job()
    launch(job) {
        try {
            repeat(1_000) { i ->
                delay(200)
                println("Printing $i")
            }
        } catch (e: CancellationException) {
            println(e)
            throw e
        }
    }
    delay(1100)
    job.cancelAndJoin()
    println("Cancelled successfully")
    delay(1000)
}
// Printing 0
// Printing 1
// Printing 2
// Printing 3
// Printing 4
// JobCancellationException...
// Cancelled successfully
```

请记住，一个被取消的协程不是仅仅的停止：它是使用一个异常在内部取消的。因此，我们可以自由地在 `finlay` 块清理所有的东西。例如，我们可以使用 `finally` 块来关闭文件或数据库连接。由于大多数资源关闭机制都依赖 `finally` 块（例如我们使用 `useLines` 读取文件），所以我们完全不需要担心它们。

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = Job()
    launch(job) {
        try {
            delay(Random.nextLong(2000))
            println("Done")
        } finally {
            print("Will always be printed")
        }
    }
    delay(1000)
    job.cancelAndJoin()
}
// Will always be printed
// (或者)
// Done
// Will always be printed
```

### 再多调用一次协程

由于我们可以捕获 `CancellationException` ，在协程真正结束之前可以执行一些操作，你可能想知道有没有什么限制。只要需要清理所有资源，协程就可以运行。然而，挂起是不允许的。 job 已经处于 “Cancelling” 状态，在这种状态下，挂起或启动另一个协程是不可能的。如果我们启动另一个协程，它将被忽略，如果我们尝试挂起，它将会抛出 `CancellationException`。

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = Job()
    launch(job) {
        try {
            delay(2000)
            println("Job is done")
        } finally {
            println("Finally")
            launch { // 会被忽略
                println("Will not be printed")
            }
            delay(1000) // 会抛出异常
            println("Will not be printed")
        }
    }
    delay(1000)
    job.cancelAndJoin()
    println("Cancel done")
}
// (1 sec)
// Finally
// Cancel done
```

有时，当协程已经取消时，我们确实需要使用挂起函数。例如，我们可能需要回滚数据库中的更改。在这种情况下，首选的方法是使用 `withContext(NonCancellable)` 函数来包装这个调用。我们稍后将详细解释 `withContext` 是如何工作的。现在，我们只需要知道它改变了代码的上下文。在 `withContext` 中，我们使用了 `NonCancelable` 对象，这是一个不能被取消的 job。因此，在 block 代码块中，job 处于活跃状态。我们可以调用任何我们想要的挂起函数。

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = Job()
    launch(job) {
        try {
            delay(200)
            println("Coroutine finished")
        } finally {
            println("Finally")
            withContext(NonCancellable) {
                delay(1000L)
                println("Cleanup done")
            }
        }
    }
    delay(100)
    job.cancelAndJoin()
    println("Done")
}
// Finally
// Cleanup done
// Done
```

### invokeOnCompletion

另一个经常用于释放资源的机制是 `Job` 中的 `invokeOnCompletion` 函数。它用于设置当 job 到达最终状态时（即 “Completed” 或 “Cancelled”）回调的代码。

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = launch {
        delay(1000)
    }
    job.invokeOnCompletion { exception: Throwable? ->
        println("Finished")
    }
    delay(400)
    job.cancelAndJoin()
}
// Finished
```

这个回调函数的参数是一个异常，这个异常是：

* 如果没有异常，则为 null
* 如果协程被取消，则为 `CancellationException`
* 一个协程完成时携带的异常（更多信息在下一章）

如果一个 job 在调用 `invokeOnCompletion` 之前已经完成，那么回调函数将立即被调用。

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = launch {
        delay(Random.nextLong(2400))
        println("Finished")
    }
    delay(800)
    job.invokeOnCompletion { exception: Throwable? ->
        println("Will always be printed")
        println("The exception was: $exception")
    }
    delay(800)
    job.cancelAndJoin()
}
// Will always be printed
// The exception was:
// kotlinx.coroutines.JobCancellationException
// (或者)
// Finished
// Will always be printed
// The exception was null
```

`invokeOnCompletion` 会在取消过程中同步调用，我们不会控制运行它的线程。

### 停下不可暂停的东西

因为取消发生在挂起点上，如果没有挂起点就不会发生。为了模拟这种情况，我们可以使用 `Thread.sleep` 而不是 `delay`。这是一种糟糕的做法，所以请不要在任何现实项目中这么做。我们只是试图模拟一种情况，在这种情况下，我们广泛的使用我们的协程，但没有挂起它们。在实践中，如果我们有一些更复杂的计算，比如神经网络学习（是的，为了简化处理并行化，我们也会使用协程），或者当我们需要做一些阻塞调用（例如，读取文件）时，就会发生这种情况。

下面的例子展示了一种情况，协程不能取消，因为它里面没有挂起点（我们使用 `Thread.sleep` 而不是 `delay`）。即便它应该在1秒后取消，但实际上执行超过了3分钟。

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = Job()
    launch(job) {
        repeat(1_000) { i ->
            Thread.sleep(200) // 这里我们可能有一些
            // 复杂的操作例如读取文件
            println("Printing $i")
        }
    }
    delay(1000)
    job.cancelAndJoin()
    println("Cancelled successfully")
    delay(1000)
}
// Printing 0
// Printing 1
// Printing 2
// ... (up to 1000)
```

另一种选择是跟踪 `job` 的状态。在协程构建器中，`this`（接收者）引用这个构建器的作用域。 `coroutineContext` 属性引用的是 `CoroutineScope` 的上下文。因此，我们可以访问协程 job（`coroutineContext[job]` 或 `coroutineContext.job`）并检查它的当前状态。由于 job 通常用于检查协程是否处于活跃状态，所以 Kotlin 协程库提供了一个函数来简化。

```kotlin
public val CoroutineScope.isActive: Boolean
    get() = coroutineContext[Job]?.isActive ?: true
```

我们可以使用 `isActive` 属性来检查 job 是否仍然处于活跃状态，并在 job 处于非活跃状态时停止计算。

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = Job()
    launch(job) {
        do {
            Thread.sleep(200)
            println("Printing")
        } while (isActive)
    }
    delay(1100)
    job.cancelAndJoin()
    println("Cancelled successfully")
}
// Printing
// Printing
// Printing
// Printing
// Printing
// Printing
// Cancelled successfully
```

或者，我们也可以使用 `ensureActive()` 函数，它会在 `Job` 不活跃时候抛出 `CancelllationException`。

```kotlin
suspend fun main(): Unit = coroutineScope {
    val job = Job()
    launch(job) {
         repeat(1000) { num ->
            Thread.sleep(200)
            ensureActive()
            println("Printing $num")
         }
    }
    delay(1100)
    job.cancelAndJoin()
    println("Cancelled successfully")
}
// Printing 0
// Printing 1
// Printing 2
// Printing 3
// Printing 4
// Cancelled successfully
```

`ensureActive()` 和 `yield()` 的结果看起来十分相似，但它们有很大的不同。函数 `ensureActive()` 需要在 `CoroutinScope`（或 `CoroutineContext` 、`Job`）上调用。它所做的事情只是在 job 不再活跃时抛出异常。它更轻量，所以通常它应该是首选。函数 `yield` 是一个常规的顶层挂起函数。它不需要任何作用域，因此可以在任意常规挂起函数中使用。由于它执行挂起和恢复操作，因此可能会产生其它影响，例如，如果我们使用带有线程池的分发器，则会导致线程更改（更多信息请参阅_Dispatchers章节_）。 `yield` 通常只用于挂起 CPU 密集型或阻塞线程的函数。

### suspendCancellableCoroutine

在这里，你可能会想起在_挂起如何在工作_一章中的 `suspendCancellableCoroutine`。它的行为类似于 `suspendCoroutine`，但是它的 continuation 被包装到了提供了额外方法的 `CancellableContinuation<T>` 中。最重要的一个方法是 `invokeOnCancellation`，我们使用它来定义取消协程时应该发生什么。我们通常使用它来取消库中的进程或者释放一些资源。

```kotlin
suspend fun someTask() = suspendCancellableCoroutine { cont ->
    cont.invokeOnCancellation {
        // do cleanup
    }
    // rest of the implementation
}
```

下面是一个完整的示例，其中我们使用挂起函数包装了一个 `Retrofit Call`。

```kotlin
suspend fun getOrganizationRepos(
    organization: String
): List<Repo> = suspendCancellableCoroutine { continuation ->
    val orgReposCall = apiService
        .getOrganizationRepos(organization)
    
    orgReposCall.enqueue(object : Callback<List<Repo>> {
        override fun onResponse(
            call: Call<List<Repo>>,
            response: Response<List<Repo>>
            ) {
                if (response.isSuccessful) {
                    val body = response.body()
                    if (body != null) {
                        continuation.resume(body)
                    } else {
                        continuation.resumeWithException(
                            ResponseWithEmptyBody
                        )
                    }
                } else {
                    continuation.resumeWithException(
                        ApiException(
                            response.code(),
                            response.message()
                        )
                    )
                }
            }
        
        override fun onFailure(
            call: Call<List<Repo>>,
            t: Throwable
        ) {
            continuation.resumeWithException(t)
        }
    })
    continuation.invokeOnCancellation {
        orgReposCall.cancel()
    }
}
```

很好，现在我们的 Retrofit 支持了挂起函数：

```kotlin
class GithubApi {
    @GET("orgs/{organization}/repos?per_page=100")
    suspend fun getOrganizationRepos(
        @Path("organization") organization: String
    ): List<Repo>
}
```

`CancellableContinuation<T>` 也允许我们检查 job 的状态（通过使用 `isActive`，`isCompleted`、还有 `isCancelled` 属性），并使用可选的取消原因（异常）取消这个 continuation。

### 总结

取消是一个强大的功能。它通常很容易使用，但有时会很棘手。所以，了解它的工作原理很重要。

正确使用取消操作意味着更少的资源浪费和更少的内存泄漏。这对我们的应用程序的性能很重要，我希望你从现在开始使用这些优点。
