# 异常处理

异常处理是协程的一个非常重要的部分。就像程序在没有捕获异常后崩溃一样，协程也会在没有捕获异常的情况下中断。这种情况并不新鲜：线程也会在这种情况下结束。区别在于协程构建器会取消它们的父协程，而每个被取消的父协程也会取消它的所有子协程，让我们看看下面的例子：协程一旦接收到异常，它就会取消自己，并将异常传递给它的父协程（`launch`）。父协程取消自身及所有子协程，然后将异常传递给父协程（`runBlocking`）。 `runBlocking`是一个根协程（它没有父类），所以它做的就是结束程序（`runBlocking` 会重新抛出异常）。

```kotlin
fun main(): Unit = runBlocking {
    launch {
        launch {
            delay(1000)
            throw Error("Some error")
        }
        launch {
            delay(2000)
            println("Will not be printed")
        }
        launch {
            delay(500) // 比异常抛出更快一点
            println("Will be printed")
        }
    }
    launch {
        delay(2000)
        println("Will not be printed")
    }
}
// Will be printed
// Exception in thread "main" java.lang.Error: Some error...
```

额外的启动协程并不会改变任何事情。异常的传播是双向的，当父节点被取消时，它也会取消子节点。因此，如果异常没有被捕获而不停的传递，那么层次结构中的所有协程都将会被取消。&#x20;

![](https://img-blog.csdnimg.cn/1a49cf411d5f4520bc81439c92ba5d33.png)

### 被让我的协程中断

在异常破坏协程之前捕获异常是有帮助的。异常信息是通过 `job` 进行的，因此使用 `try-catch` 包装在协程构造器上没有任何帮助。

```kotlin
fun main(): Unit = runBlocking {
    // 不要用 try-catch 封装，它会被忽略
    try {
        launch {
            delay(1000)
            throw Error("Some error")
        }
    } catch (e: Throwable) { // 这样做没有任何帮助
        println("Will not be printed")
    }
    launch {
        delay(2000)
        println("Will not be printed")
    }
}
// Exception in thread "main" java.lang.Error: Some error...
```

### SupervisorJob

阻止协程异常中断的最重要方法，是使用一个 `SupervisorJob`。这是一个特殊的 job，它可以忽略子 job 的所有异常。

![在这里插入图片描述](https://img-blog.csdnimg.cn/26ca200ca1454a57a999479f967daf1b.png)

&#x20;

![](https://img-blog.csdnimg.cn/667dd2fbadc9477faa820d882c67bf44.png)

通常使用 `SuspervisJob` 作为一个作用域的一部分，在这个作用域中，我们可以启动多个协程。

```kotlin
fun main(): Unit = runBlocking {
    val scope = CoroutineScope(SupervisorJob())
    scope.launch {
        delay(1000)
        throw Error("Some error")
    }
    scope.launch {
        delay(2000)
        println("Will be printed")
    }
    delay(3000)
}
// Exception...
// Will be printed
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/7abc6d4509a142f7af9bb24e2eaa71c1.png)

一个常见的错误是使用 `SusperviseJob` 作为父协程的参数，就像下面代码所示。这样做并不能帮助我们处理异常，因为在这种情况下，`SupervisorJob` 只有一个直接的子 job，也就是注释1处启动的子协程。它接收了 `SupervisorJob` 作为参数，因此，在这种情况下，使用 `SupervisorJob` 没有任何优势。

```kotlin
fun main(): Unit = runBlocking {
    // 不要这么做， SupervisorJob 只有一个子 Job
    // 并且没有一个父 Job 会像这里的 Job 一样工作
    launch(SupervisorJob()) { // 1
        launch {
            delay(1000)
            throw Error("Some error")
        }
        launch {
            delay(2000)
            println("Will not be printed")
        }
    }
    delay(3000)
}
// Exception...
```

如果我们使用相同的 job 作为多个协程构建器的上下文，这样会更有意义。因为它们每个都可以取消，但它们不会取消彼此。

```kotlin
fun main(): Unit = runBlocking {
    val job = SupervisorJob()
    launch(job) {
        delay(1000)
        throw Error("Some error")
    }
    launch(job) {
        delay(2000)
        println("Will be printed")
    }
    job.join()
}
// (1 sec)
// Exception...
// (1 sec)
// Will be printed
```

### supervisorScope

另一种停止异常传递的方法是使用 `supervisorScope` 包装协程构建器。这是非常方便的，因为我们仍然保持与父节点的连接，而任何来自协程的异常都将被忽略。

```kotlin
fun main(): Unit = runBlocking {
    supervisorScope {
        launch {
            delay(1000)
            throw Error("Some error")
        }
        launch {
            delay(2000)
            println("Will be printed")
        }
    }
    delay(1000)
    println("Done")
}
// Exception...
// Will be printed
// (1 sec)
// Done
```

`supervisorScope` 只是一个挂起函数，可以被用来包装另一些挂起函数。这些知识将会在下一章中更详细的讲解，它的常用方式是启动多个独立的任务。

```kotlin
suspend fun notifyAnalytics(actions: List<UserAction>) =
    supervisorScope {
        actions.forEach { action ->
            launch {
                notifyAnalytics(action)
            }
    }
}
```

另一种停止异常传播的方法是使用 `coroutineScope`。此函数不会影响父函数，而是抛出一个可以使用 `try-catch` 捕获的异常（与协程构建器相反）。它们都将在下一章中描述。

### Await

因此，我们知道如何停止异常传递，但有时这是不够的。在异常的情况下，`async` 协程构建器会破坏其父协程，将像 `launch` 和其它与父协程有关的协程构建器一样。但是，如果这个过程是静默的（例如使用 `supervisorJob` 或者 `superviseScope`），并调用 `await`，该怎么办？让我们看看下面的例子：

```kotlin
class MyException : Throwable()

suspend fun main() = supervisorScope {
    val str1 = async<String> {
        delay(1000)
        throw MyException()
    }
    val str2 = async {
        delay(2000)
        "Text2"
    }
    try {
        println(str1.await())
    } catch (e: MyException) {
        println(e)
    }
    println(str2.await())
}
// MyException
// Text2
```

我们没有返回值，因为协程是一个异常结束，所以 `MyException` 会由 `await` 抛出。这就是为什么打印了 MyException。另一个 async 则不间断完成，因为我们使用的是 `supervisorScope`。

### CancellationException 不会传递给它的父类

如果异常是 `CancellationException` 的子类，它将不会传递给它的父类。它只会取消当前协程， `CancellationException` 是一个 open 类，因此它可以被我们自己的类或对象来扩展。

```kotlin
object MyNonPropagatingException : CancellationException()

suspend fun main(): Unit = coroutineScope {
    launch { // 1
        launch { // 2
            delay(2000)
            println("Will not be printed")
        }
        throw MyNonPropagatingException // 3
    }
    launch { // 4
        delay(2000)
        println("Will be printed")
    }
}
// (2 sec)
// Will be printed
```

在上面的代码段中，我们使用构建器1和4启动两个协程，在3处，抛出 `MyNonPropagatingException` 异常，它是 `CancellationException` 的子类型，所以这个构造器取消了它自己，然后它也取消了它的子构造器。也就是在2处定义的构造器。而第二次 `lauinch` 将不受影响，2s后会打印 “Will be printed”。

### 协程异常处理器

在处理异常时，有时所有的异常出现时的默认行为是很有帮助的。这就是 `CoroutineExceptionHander` 上下文派上用场的地方。它不会阻止异常传递，但可以使用它定义在发生异常时可以发生的行为（默认情况，它会打印异常堆栈）。

```kotlin
fun main(): Unit = runBlocking {
    val handler =
        CoroutineExceptionHandler { ctx, exception ->
            println("Caught $exception")
        }

    val scope = CoroutineScope(SupervisorJob() + handler)
    scope.launch {
        delay(1000)
        throw Error("Some error")
    }
    scope.launch {
        delay(2000)
        println("Will be printed")
    }
    delay(3000)
}
// Caught java.lang.Error: Some error
// Will be printed
```

这个上下文在许多平台上都有用，可以添加处理异常的默认方式。对于 Android，它通常展示对话框或错误消息来通知用户。

### 总结

异常处理是 kotlinx.coroutines 的重要组成部分。随着时间的推移，我们将不可避免的回到这些话题，现在，我希望你理解在基本构建器中异常是如何从子 job 传到父 job 了，以及该如何阻止它们。现在是时候讨论期待已久的话题了 —— 协程作用域函数了。
