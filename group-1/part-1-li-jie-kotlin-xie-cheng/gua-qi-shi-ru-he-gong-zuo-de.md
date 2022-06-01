# 挂起是如何工作的？

挂起函数是 Kotlin 协程的标志。挂起功能是建立所有其他 Kotlin Coroutines 概念的最基本特性。我们本章的目标就是理解它是如何工作的。

挂起一个协程意味着在中间停止它。这类似与电子游戏中的停止：你在一个检查点保存，关闭游戏后，你和你的电脑可以专注到别的事情上。接着，当你想在一段时间后继续玩游戏时，你可以再次打开游戏，并从保存的检查点恢复，这样你就可以从之前停止的地方开始玩游戏。协程同理，当它们被挂起时，会返回一个 `Continuation`。这就像是游戏中的存档点：我们可以使用它从我们停止的地方继续。

注意，它与线程非常不同，线程不能被保存，只能被阻塞。而协程要强大的多。挂起时，它不会消耗任何资源。协程可以在不同的线程上恢复（resume），并且（至少在理论上）一个协程可以被序列化、反序列化，然后再恢复。

### 恢复

让我们来看看实际情况。为此我们需要一个协程。我们使用协程构建器（如 `runBlocking` 或 `launch`）来启动协程，这会在后面介绍到。也有一种更简单的方法：我们可以挂起一个 `main` 函数。

挂起函数是一个可以挂起协程的函数。这意味着它们必须从协程（或另一个挂起函数）中调用。最后，它们需要一些东西来挂起，下面的 `main` 函数就是一个挂起函数，所以当我们运行它时，Kotlin 将在协程中启动它。

```kotlin
suspend fun main() {
    println("Before")
    println("After")
}
// Before
// After
```

这是一个简单的程序，将会打印“Before”和“After”。如果我们停在这两个打印函数之间，会发生什么呢？为此，我们可以使用 Kotlin 协程库提供的 `suspendCoroutine` 函数。

```kotlin
suspend fun main() {
    println("Before")
    suspendCoroutine<Unit> { }
    println("After")
}
// Before
```

如果你调用上述代码，你将看不到“After”，并且代码不会停止运行（因为我们的 `main` 函数永远不会结束）。协程在“Before”之后挂起，我们的函数停止了，并永远不会恢复。那么怎样才能恢复呢？前面提到的 `Continuation` 在哪里呢？

再看下 `suspendCoroutin` 的调用，注意它以一个 lambda 表达式结束。**作为参数传递的函数将在挂起之前被调用**。该函数会获得一个 `continuation` 作为参数：

```kotlin
suspend fun main() {
    println("Before")
    suspendCoroutine<Unit> { continuation ->
        println("Before too")
    }
    println("After")
}
// Before
// Before too
```

这样的函数就地调用另一个函数并不是什么新鲜事。这类似于 `let`、`apply` 或者 `useLines`。 `suspendCoroutine` 函数以同样的方式设计，这使得在挂起之前使用 `continuation` 成为可能。如果在 `suspendCoroutine` 调用之后运行 lambda 就太晚了，因此，会在函数挂起之前调用作为参数传递给 `suspendCoroutine` 的 lambda 表达式。这个 lambda 的作用是将 `continuation` 存储在某处，或计划是否恢复协程。

我们可以使用它来立即恢复：

```kotlin
suspend fun main() {
    println("Before")
    suspendCoroutine<Unit> { continuation ->
        continuation.resume(Unit)
    }
    println("After")
}
// Before
// After
```

注意，上面例子中的“After”被打印出来了，因为我们在 `suspendCoroutine` 中调用了 `resume` 函数。

从 Kotlin 1.3开始，`Continuation` 的定义发生了变化。代替了传统的 `resume` 和 `resumeWithException`，使用了一个接收 `Result` 的 `resumeWith` 函数。我们正在使用的 `resume` 和 `resumeWithException` ，它们其实是调用了 `resumeWith` 的扩展函数。

```kotlin
inline fun <T> Continuation<T>.resume(value: T): Unit =
    resumeWith(Result.success(value))

inline fun <T> Continuation<T>.resumeWithException(
    exception: Throwable
): Unit = resumeWith(Result.failure(exception))
```

我们也可以启动一个不同的线程，它将睡眠一段时间，并在一定时间后恢复：

```kotlin
suspend fun main() {
    println("Before")
    suspendCoroutine<Unit> { continuation ->
        thread {
            println("Suspended")
            Thread.sleep(1000)
            continuation.resume(Unit)
            println("Resumed")
        }
    }
    println("After")
}
// Before
// Suspended
// (1 second delay)
// After
// Resumed
```

这是一个重要的观察结果，请注意，我们可以创建一个函数，该函数将在一个定义的周期之后恢复 `continuation`。在这种情况下，`continuation` 由 lambda 表达式获得，如下面代码段所示：

```kotlin
fun continueAfterSecond(continuation: Continuation<Unit>) {
    thread {
        Thread.sleep(1000)
        continuation.resume(Unit)
    }
}

suspend fun main() {
    println("Before")
    suspendCoroutine<Unit> { continuation ->
        continueAfterSecond(continuation)
    }
    println("After")
}
// Before
// (1 sec)
// After
//sampleEnd
```

这种机制是可行的，但它没有必要创建线程，只是为了在一秒钟不活动之后结束它们，线程并不廉价，所以为什么要浪费它们呢？一个更好的方法是设置一个“闹钟”。在 JVM 中，我们可以使用 `ScheduledExecutorService`。我们可以设置它在一段指定的时间后调用 `continuation.resume(Unit)`。

```kotlin
private val executor = 
    Executors.newSingleThreadScheduledExecutor {
        Thread(it, "scheduler").apply { isDaemon = true }
}

suspend fun main() {
    println("Before")
    suspendCoroutine<Unit> { continuation ->
        executor.schedule({
            continuation.resume(Unit)
        }, 1000, TimeUnit.MILLISECONDS)
    }
    println("After")
}
// Before
// (1 second delay)
// After
```

挂起一段时间似乎是一个有用的功能。让我们把它提取成一个函数，将它命名为 `delay`：

```kotlin
private val executor = 
    Executors.newSingleThreadScheduledExecutor {
        Thread(it, "scheduler").apply { isDaemon = true }
}

suspend fun delay(timeMillis: Long): Unit =
    suspendCoroutine { cont ->
        executor.schedule({
            cont.resume(Unit)
        }, timeMillis, TimeUnit.MILLISECONDS)
}

suspend fun main() {
    println("Before")
    delay(1000)
    println("After")
}
// Before
// (1 second delay)
// After
```

执行程序仍然使用一个线程，但对于所有使用 `delay` 函数的协程来说，它又是一个线程，这比每次需要等待一段时间阻塞一个线程要好得多。

这正是 Kotlin Coroutines 库过去实现 `delay` 函数的方式。目前的实现比较复杂，主要是为了支持测试，但其基本思想仍然不变。

### 携带一个值恢复

你可能会关心的一件事情是：为什么我们要将 Unit 传递给 `resume` 函数？你可能还想知道为什么使用 `Unit` 作为 `suspendCoroutine` 上的类型参数（泛型）。事实上，这两者相同并不是巧合。`Unit` 也是该函数的返回值，并且是 `Continuation` 的泛型类型：

```kotlin
val ret: Unit =
    suspendCoroutine<Unit> { cont: Continuation<Unit> ->
        cont.resume(Unit)
    }
```

当调用 `suspendCoroutine` 时，可以指定其 `continuation` 返回的类型。调用 `resume` 时需要使用相同的类型。

```kotlin
suspend fun main() {
    val i: Int = suspendCoroutine<Int> { cont ->
        cont.resume(42)
    }
    println(i) // 42

    val str: String = suspendCoroutine<String> { cont ->
        cont.resume("Some text")
    }
    println(str) // Some text
    
    val b: Boolean = suspendCoroutine<Boolean> { cont ->
        cont.resume(true)
    }
    println(b) // true
}
```

这与游戏类比并不相符。我不知道有哪款游戏能够在恢复保存时添加新内容（除非你开挂，或者google了一下）。然而，它对于协程来说是非常有意义的。我们经常挂起是因为我们在等待一些数据，比如来自 API 的网络响应。这是一个常见的场景。线程一直在运行业务逻辑，直到它得到一些数据。如果没有协程，我们的线程需要坐下来慢慢等待这个子线程返回数据。这是一个巨大的浪费，因为线程是昂贵的，特别是如果这是一个重要的线程，例如 Android 的主线程。对于协程，它只是挂起并给我们一个 `Continueation` 的指令：“一旦你得到了数据，就把将其发送至 `resume`”。然后线程就可以做其他事情了，一旦拿到数据，线程将被用来从协程被挂起的地方恢复。

为了查看实际情况，让我们看看如何挂起直到接收到一些数据。在下面的例子中，我们使用了一个外部实现的回调函数 `requestUser`：

```kotlin
suspend fun main() {
    println("Before")
    val user = suspendCoroutine<User> { cont ->
        requestUser { user ->
            cont.resume(user)    
        }
    }
    println(user)
    println("After")
}
// Before
// (1 second delay)
// User(name=Test)
// After
```

直接调用 `suspendCoroutine` 并不方便，我们更希望有一个挂起函数可以替代，我们可以自行提取：

```kotlin
suspend fun requestUser(): User {
    return suspendCoroutine<User> { cont ->
        requestUser { user ->
            cont.resume(user)
        }
    }
}

suspend fun main() {
    println("Before")
    val user = requestUser()
    println(user)
    println("After")
}
```

目前，许多主流的库已经支持挂起功能，例如 `Retrofit` 和 `Room`，这就是为什么我们很少需要在挂起函数中使用回调函数的原因。然而，如果你有这样的需求，我建议使用 `suspendCancellableCoroutine` （代替 `suspendCoroutine`），这将在_取消_章节中解释。

```kotlin
suspend fun requestUser(): User {
    return suspendCancellableCoroutine<User> { cont ->
        requestUser { user ->
            cont.resume(user)
        } 
    }
}
```

你可能知道，如果 API 给我们的不是数据而是某种异常，会发生什么。如果服务器死机或者响应错误该怎么办？在这种情况下，我们不能返回数据，相反，我们应该从协程挂起的地方抛出一个异常。这就是我们需要在例外情况下恢复的地方。

### 携带一个异常恢复

我们调用每个函数都可能返回某个值或抛出异常。对于 `suspendCoroutine` 亦是如此。当调用 `resume` 时，它返回数据。当调用 `resumeWithException` 时，它返回异常，在挂起点抛出：

```kotlin
class MyException : Throwable("Just an exception")
    suspend fun main() {
        try {
            suspendCoroutine<Unit> { cont ->
                cont.resumeWithException(MyException())
            }
        } catch (e: MyException) {
            println("Caught!")
        }
}
// Caught!
```

这种机制用于解决各种各样的问题，例如，发送网络异常的信号：

```kotlin
suspend fun requestUser(): User {
    return suspendCancellableCoroutine<User> { cont ->
        requestUser { resp ->
            if (resp.isSuccessful) {
                cont.resume(resp.data)
            } else {
                val e = ApiException(
                    resp.code,
                    resp.message
                )
                cont.resumeWithException(e)
            }
        }
    }
}

suspend fun requestNews(): News {
    return suspendCancellableCoroutine<News> { cont ->
        requestNews(
            onSuccess = { news -> cont.resume(news) },
            onError = { e -> cont.resumeWithException(e) }
        )
    }
}
```

### 挂起一个协程，而不是函数

这里需要强调的点是：**我们挂起的是协程，而不是函数**。**挂起函数不是协程，只是可以挂起协程的函数**。假设我们将一个函数存储在某个变量中，并试图在函数调用后恢复它：

```kotlin
// 不要这么做！
var continuation: Continuation<Unit>? = null

suspend fun suspendAndSetContinuation() {
    suspendCoroutine<Unit> { cont ->
        continuation = cont
    }
}

suspend fun main() {
    println("Before")
    suspendAndSetContinuation()
    continuation?.resume(Unit)
    println("After")
}
// Before
```

这没有道理的，这相当于停止了游戏，并计划在游戏的存档点之后的某一点开始。`resume` 将永远不会被调用。你只会看到“Before”，程序永远不会结束，除非我们在另一个线程或另一个协程在恢复它。为了展示这一点，我们可以设置另一个协程在一秒钟后恢复：

```kotlin
// 不要这么做，会有潜在的内存泄漏风险
var continuation: Continuation<Unit>? = null

suspend fun suspendAndSetContinuation() {
    suspendCoroutine<Unit> { cont ->
        continuation = cont
    }
}

suspend fun main() = coroutineScope {
    println("Before")
    launch {
        delay(1000)
        continuation?.resume(Unit)
    }
    suspendAndSetContinuation()
    println("After")
}
// Before
// (1 second delay)
// After
```

### 总结

我希望现在你能从使用者的角度清楚的了解挂起是如何工作的。这很重要，我们将在整本书中看到这一点，它也很实用，因为现在你可以将回调函数转化成挂起函数。如果你像我一样，想要确切的知道事情在底层是如何运行的，我们将在下一章讨论，如果你觉得不需要知道，那就跳过它，它并不实用，只是揭示了 Kotlin 协程的魔力。
