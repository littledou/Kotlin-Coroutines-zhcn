# Dispatchers 协程调度器

Kotlin 协程库提供的一个重要的功能是让我们能决定协程应该运行在哪个线程（或线程池）上，这是通过使用调度器来完成的。

在英语词典中，dispatcher 的意思是“负责将人员或者车辆运送到指需要地方的人，尤其是那些紧急车辆”。在 Kotlin 协程中， `CoroutineContext` 可以决定某个协程在哪个线程上运行。

Dispatchers in Kotlin Coroutines are a similar concept to RxJava Schedulers.

> Kotlin 协程中的 Dispatchers 与 RxJava 中的 Schedulers 是一个类似的概念。

### 默认调度器

如果不设置任何调度器，则默认选择 `Dispatchers.Default`，它被设计用于运行 CPU 密集型操作。它有一个线程池，其大小等于运行代码的机器上的 CPU 核数量（但不少于2个）。至少从理论上来讲，这是线程池的最佳数量，前提是你能有效的使用这些线程，也就是说，要执行 cpu 密集型计算，而不阻塞它们。

```kotlin
suspend fun main() = coroutineScope {
    repeat(1000) {
        launch { // 或者 launch(Dispatchers.Default) {
            // 不断计算
            List(1000) { Random.nextLong() }.maxOrNull()
            val threadName = Thread.currentThread().name
            println("Running on thread: $threadName")
        }
    }
}
```

在我的机器上的示例结果如下（我的电脑有12个内核，所以池中有12个线程）：

```kotlin
Running on thread: DefaultDispatcher-worker-1
Running on thread: DefaultDispatcher-worker-5
Running on thread: DefaultDispatcher-worker-7
Running on thread: DefaultDispatcher-worker-6
Running on thread: DefaultDispatcher-worker-11
Running on thread: DefaultDispatcher-worker-2
Running on thread: DefaultDispatcher-worker-10
Running on thread: DefaultDispatcher-worker-4
...
```

### 限制默认调度器

假设你有一个开销很大的任务，你怀疑它可能使用了 `Dispatchers.Default` 上所有的线程，从而让其它使用相同调度器的协程饥饿。这种情况下，我们可以在 `Dispatchers.Default` 上使用 `limitedParallelism` 来创建一个调度器，该调度将运行在相同的线程上，但它会限制在使用时同时不超过一定数量的线程：

```kotlin
private val dispatcher = Dispatchers.Default
    .limitedParallelism(5)
```

这种机制其实并不是被用来限制 `Dispatcher.Default` 的，但值得记住它。因为很快我们将介绍 `Dispatcher.IO` 上的 `limitedParallelism`，它更加重要，也常见的多。

> `limitedParallelism` 是从 kotlinx.coroutines 1.6 版本引入的

### 主调度器

Android 和许多其它应用程序框架都有主线程或UI线程的概念，它们通常是最重要的线程。在 Android 中，它是唯一可以用来和 UI 交互的线程。因此，它需要经常使用，但也要非常小心。当主线程被阻塞时，整个应用会卡住。如果我们使用 kotlinx-coroutines-android 构件，且在主线程上运行协程，我们可以使用 `Dispatchers.Main` 来调度。同样，如果我们使用 `kotlinx-coroutines-javaFx` ，它也可以在 JavaFx 上使用。如果我们使用 kotlinx-coroutines-swing，它可以在 Swing 上使用。如果没有指定调度器的依赖项，则无法使用这些东西。注意，单元测试中通常不使用前端库，所以这里没有定义 `Dispatchers.Main`，为了能够使用它，你需要使用 kotlinx-coroutines-test 中的 `Dispathers.setMain(dispatcher)` 来设置一个调度器。

```kotlin
class SomeTest {
    private val dispatcher = Executors
        .newSingleThreadExecutor()
        .asCoroutineDispatcher()
   
    @Before
    fun setup() {
        Dispatchers.setMain(dispatcher)
    }
    
    @After
    fun tearDown() {
        // 重置主调度器
        Dispatchers.resetMain()
        dispatcher.close()
    }

    @Test
    fun testSomeUI() = runBlocking {
        launch(Dispatchers.Main) {
            // ...
        }
    }
}
```

在 Andorid 上，我们通常使用主调度器来作为默认调度器。如果你使用挂起而非阻塞的库，并且你不做任何复杂的计算，那么在实践中通常只使用 `Dispatcher.Main`。如果要执行一些 cpu 密集型操作，则应该在 `Dispatcher.Default` 上运行它们，这两个对于是许多场景来说已经足够用了，但如果你需要阻塞线程呢？例如，如果你需要执行长时间的 I/O 操作（例如，读取大文件）或如果你需要使用带有阻塞函数的库。你不能阻塞主线程，因为你的应用程序将会卡住。如果阻塞了默认调度器，则有可能阻塞线程池中的所有线程，在这种情况下，你将无法进行任何计算。这就是为什么在这种情况下，我们需要一个另外一个调度器，也就是 `Dispatchers.IO`。

### IO调度器

`Dispatchers.IO` 被设计当我们执行阻塞线程的 I/O 操作时使用，例如当我们读写文件、使用 Android 的 shared preferences（首选项），或者调用阻塞函数。下面代码运行大约需要一秒钟，因为 `Dispatcher.IO` 可以允许同时有 50 多个活跃线程：

```kotlin
suspend fun main() {
    val time = measureTimeMillis {
        coroutineScope {
            repeat(50) {
                launch(Dispatchers.IO) {
                    Thread.sleep(1000)
                }
            }
        }
    }
    println(time) // ~1000
}
```

它是如何工作的？ 想象一个无限的线程池，它最初是空的，但随着我们需要更多的线程，线程被创建并保持活跃，直到一段时间不再使用它们。这样的池子是存在的，但是直接使用它们并不安全，如果有太多的活跃线程，程序性能会以一种缓慢但无止境的方式下降，最终导致内存不足错误。这就是为什么我们创建的 dispatchers 可以同时使用的线程数量是有限的。 `Dispatchers.IO` 的默认限制是 64 个。

```kotlin
suspend fun main() = coroutineScope {
    repeat(1000) {
        launch(Dispatchers.IO) {
            Thread.sleep(200)
            val threadName = Thread.currentThread().name
            println("Running on thread: $threadName")
        }
    }
}
// Running on thread: DefaultDispatcher-worker-1
//...
// Running on thread: DefaultDispatcher-worker-53
// Running on thread: DefaultDispatcher-worker-14
```

正如我们提到的， `Dispatcher.Default` 和 `Dispatchers.IO` 共享相同的线程池。这是一个重要的优化，线程被重用，并且通常不需要被重新调度。假设你的程序在 `Dispatcher.Default` 上运行，然后执行 `withContext(Dispatchers.IO) {...}`。大概率你将会停留在同一个线程上，但区别是线程限制数量不再是 `Dispatcher.Default` 来指定了，而是 `Dispatchers.IO` 来指定。它们的限制是互相独立的，所以它们永远不会让对方饥饿。

```kotlin
suspend fun main(): Unit = coroutineScope {
    launch(Dispatchers.Default) {
        println(Thread.currentThread().name)
        withContext(Dispatchers.IO) {
            println(Thread.currentThread().name)
        }
    }
}
// DefaultDispatcher-worker-2
// DefaultDispatcher-worker-2
```

为了更清楚的了解这一点，假设你最大限度的使用两个 `Dispatchers.Default` 和 `Dispatchers.IO` ，活跃线程的数量将是它们限制的总和。如果你允许 `Dispatcher.IO` 最多有64个线程，而你有8个内核，那么你的共享池将会有 72 个活跃线程。这意味着我们可以高效地重用线程，两个调度器都有很强的独立性。

使用 `Dispatchers.IO` 的最典型情况是需要从库中调用阻塞函数。最佳做法是使用 `withContext(Dispatchers.IO)` 来包装它们，使它们成为挂起函数。使用这些函数无需特别注意，因为它们可以像其他挂起函数一样被正确的处理。

```kotlin
class DiscUserRepository(
    private val discReader: DiscReader
) : UserRepository {

    override suspend fun getUser(): UserData =
        withContext(Dispatchers.IO) {
            UserData(discReader.read("userName"))
        }
}
```

唯一的问题是这些函数可能会阻塞太多线程。调度器限制为64个，一个大规模阻塞线程的服务可能会让所有其它服务暂停等待。为了帮助我们处理这个问题，我们需要再次使用 `limitedParallelism`。

### 带有自定义线程池的 IO 调度器

`Dispatcher.IO` 为 `limitedParallelism` 函数定义了一个特殊的行为，它创建一个具有独立线程池的新调度器。更重要的是，这个池不受限于64个，因为我们可以决定将其限制为我们想要的线程数量。

例如，假设你启动了100个协程，每个协程阻塞一个线程1秒钟，如果在 `Dispatcher.IO` 上运行这些协程，将会花费2秒的时间。但是如果你在 `Dispatcher.IO` 上运行，并且调用了 `limitedParalleism(100)` ，这个程序则只需要1秒。可以同时对这两个调度器的执行时间测时，因为这两个调度器的限制是互相独立的：

```kotlin
suspend fun main(): Unit = coroutineScope {
    launch {
        printCoroutinesTime(Dispatchers.IO)
        // Dispatchers.IO took: 2074
    }
    launch {
        val dispatcher = Dispatchers.IO
            .limitedParallelism(100)
        printCoroutinesTime(dispatcher)
        // LimitedDispatcher@XXX took: 1082
    }
}

suspend fun printCoroutinesTime(dispatcher: CoroutineDispatcher) {
    val test = measureTimeMillis {
        coroutineScope {
            repeat(100) {
                launch(dispatcher) {
                    Thread.sleep(1000)
                }
            }
        }
    }
    println("$dispatcher took: $test")
}
```

从概念上来看，你可以这样想象：

```kotlin
// 没有限制线程的池子
private val pool = ...

Dispatchers.IO = pool.limitedParallelism(64)
Dispatchers.IO.limitedParallelism(x) = pool.limitedParallelism(x)
```

对于可能密集阻塞线程的类，最佳的实践是定义它们自己的调度器。这些调度器有它们自己的独立限制。这个限制应该有多大？你需要自己做决定。太多的线程是对资源的低效利用，而等待可用的线程又不利于性能。最重要的是，这个限制将独立于 Dispatcher 和其它 Dispatcher 的限制，正因如此，一个服务不会阻塞另一个服务。

```kotlin
class DiscUserRepository(
    private val discReader: DiscReader
) : UserRepository {
    private val dispatcher = Dispatchers.IO
        .limitParallelism(5)
    
    override suspend fun getUser(): UserData =
        withContext(dispatcher) {
            UserData(discReader.read("userName"))
        }
}
```

### 带有固定线程池的调度器

一些开发人员喜欢对他们使用的线程池有更多的控制，Java 为此提供了一个强大的Api。例如，我们可以使用 `Executors` 类创建一个固定的或带缓存的线程池。这些池实现了 `ExecutorService` 或 `Executor` 接口，我们可以使用 `asCoroutineDispatcher` 函数将其转换为调度器。

```kotlin
val NUMBER_OF_THREADS = 20
val dispatcher = Executors
    .newFixedThreadPool(NUMBER_OF_THREADS)
    .asCoroutineDispatcher()
```

在 kotlinx-coroutines 1.6 版本以前（没有 `limitedParallelism` 函数），我们经常使用 Executors 类来创建具有独立线程池的调度器。

这种方法最大的问题是，使用 `ExecutorService.asCoroutineDispatcher()` 创建的调度器需要使用 `close` 函数来关闭，而开发人员经常忘记这一点，这将会导致线程泄漏。另一个问题是，当你创建一个固定线程数的线程池时，你很有可能会低效的使用它们。你可能将未使用的线程保持活跃状态，而又不与其它服务共享这些线程。

### 带有单个线程的调度器

对于使用多个线程的调度器，我们需要考虑状态共享的问题。请注意，在下面示例中，有10000个协程给 i 增加1。因此，i 的值应该是10000，但实际上它是一个更小的数字。这是在多个线程上同时修改共享状态（i属性）的结果。

```kotlin
var i = 0

suspend fun main(): Unit = coroutineScope {
    repeat(10_000) {
        launch(Dispatchers.IO) { // or Default
            i++
        }
    }
    delay(1000)
    println(i) // ~9930
}
```

有许多方法可以解决这个问题（大多数方法将在_状态的问题_一章中描述），其中一个选择是只使用单个线程的调度器。如果我们一次只使用一个线程，则不需要任何其它同步。完成此操作的典型做法是使用 `Executor` 来创建这样的调度器：

```kotlin
val dispatcher = Executors.newSingleThreadExecutor()
    .asCoroutineDispatcher()
// 以前的做法:
// val dispatcher = newSingleThreadContext("My name")
```

问题是这个调度器持有一个活跃线程，当它不再使用时需要及时关闭它。一种现代的解决方案是使用 `Dispatcher.Default` 或 `Dispatchers.IO` （如果我们阻塞线程），并限制最大并行数为 1。

```kotlin
var i = 0

suspend fun main(): Unit = coroutineScope {
    val dispatcher = Dispatchers.Default
        .limitedParallelism(1)
        
    repeat(10000) {
        launch(dispatcher) {
            i++
        }
    }
    delay(1000)
    println(i) // 10000
}
```

这样做的最大缺点是，因为我们只有一个线程，假如我们阻塞了它，我们的其他操作将要等待。

```kotlin
suspend fun main(): Unit = coroutineScope {
    val dispatcher = Dispatchers.Default
        .limitedParallelism(1)
    
    val job = Job()
    repeat(5) {
        launch(dispatcher + job) {
            Thread.sleep(1000)
        }
    }
    job.complete()
    val time = measureTimeMillis { job.join() }
    println("Took $time") // Took 5006
}
```

### Unconfined dispatcher

我们要讨论的最后一个调度器就是 `Dispatchers.Unconfined`，这个调度器不同于之前别的调度器，因为它不会改变任何的线程。当它启动时，它在启动它的线程上运行，如果它被恢复，它将在恢复它的线程上运行。

```kotlin
suspend fun main(): Unit =
    withContext(newSingleThreadContext("Thread1")) {
        var continuation: Continuation<Unit>? = null
        
        launch(newSingleThreadContext("Thread2")) {
            delay(1000)
            continuation?.resume(Unit)
        }

        launch(Dispatchers.Unconfined) {
            println(Thread.currentThread().name) // Thread1
            suspendCancellableCoroutine<Unit> {
                continuation = it
            }
            println(Thread.currentThread().name) // Thread2
            delay(1000)
            println(Thread.currentThread().name)
            // kotlinx.coroutines.DefaultExecutor
        }
    }
```

这对于单元测试时有时非常有用。假设你需要测试一个调用 `launch` 的函数。将它们在时间上同步可能并不容易。一种解决方案是使用 `Dispatchers.Unconfined` 而不是其他的调度器。如果它在所有作用域中使用，那么所有操作都运行在相同的线程上，我们可以更容易的控制操作的顺序。如果我们使用 kotinx-coroutines-test 中的 `runTest`，则不需要这个技巧。我们将在本书的后面讨论这个问题。

从性能的角度来看，这个调度器的成本是最低廉的，因为它甚至不需要切换线程。因此，我们选择它可能不需要关心代码运行在哪个线程上。然而，在实践中，轻率地使用它是不好的。如果我们不小心调用了阻塞操作，而我们又在主线程上，这可能会阻塞整个应用程序。

### Dispatchers.Main.immediate

调度协程是有成本的，当调用 `withContext` 时，协程需要挂起，可能要在队列中等待，然后恢复。如果我们已经在这个线程上，调度协程是一个很小但不必要的成本。请看下面函数：

```kotlin
suspend fun showUser(user: User) =
    withContext(Dispatchers.Main) {
        userNameElement.text = user.name
        // ...
    }
```

如果这个函数已经处于主线程上了，我们将会有一个不必要的重新调度的成本。最重要的是， 主线程如果有一个很长的队列，由于 `withContext`，用户数据可能会有一些延迟才显示（这个协程需要等待其他协程完成它们的工作）。为了防止这种情况，有 `Dispatchers.Main.immediate`，它只在需要时才进行调度。因此，如果下面函数在主线程上被调用，它不会被重新调度，而是会被立刻执行：

```kotlin
suspend fun showUser(user: User) =
    withContext(Dispatchers.Main.immediate) {
        userNameElement.text = user.name
        // ...
    }
```

当这个函数可能已经从主调度器调用时，我们更喜欢将 `Dispatchers.Main.immediate` 作为 `withContext` 参数，目前，其他调度器不支持这种立即调度。

### Continuation 拦截器

调度器的运作原理是基于 continuation 的拦截机制，它内置在 Kotlin 语言中，有一个名为 `ContinuationInterceptor` 的协程上下文，它的 `interceptContinuation` 函数用于在协程被挂起时修改一个 continuation。他还有一个 `releaseInterceptedContinuation` 函数，在 continuation 结束后调用该函数。

```kotlin
public interface ContinuationInterceptor :
    CoroutineContext.Element {
    
    companion object Key :
        CoroutineContext.Key<ContinuationInterceptor>
    
    fun <T> interceptContinuation(
        continuation: Continuation<T>
    ): Continuation<T>
    
    fun releaseInterceptedContinuation(  
        continuation: Continuation<*>
    ) {
    }
    //...
}
```

这种封装了 continuation 的能力提供了很多控制。 `Dispatchers` 使用 `interceptContinuation` 用 `DispatchedContinuation` 来包装一个 continuation， `DispatchedContinuation` 运行在一个特定的线程池上。这就是调度器的工作方式。

这里的问题是许多测试库也使用相同的上下文，例如 kotlinx-coroutines-test 中的 `runTest`。上下文中的每个元素都必须要有唯一的键。这就是为什么我们有时会注入调度器，用测试调度器在单元测试中去替换它们。我们将在专门讨论协程测试的章节中深入这个主题。

```kotlin
class DiscUserRepository(
    private val discReader: DiscReader,
    private val dispatcher: CoroutineContext = Dispatchers.IO,
) : UserRepository {
    
    override suspend fun getUser(): UserData =
        withContext(dispatcher) {
            UserData(discReader.read("userName"))
        }
}

class UserReaderTests {
    @Test
    fun `some test`() = runTest {
        val discReader = FakeDiscReader()
        val repo = DiscUserRepository(
            discReader,
            // 一种协程测试实践
            this.coroutineContext[ContinuationInterceptor]!!
        )
        //...
    }
}
```

### 调度器对不同任务的性能

为了展示不同的调度器对不同任务的执行情况，我做了一些基准测试。测试工作是对同一个任务运行100个独立的协程。不同的列代表不同任务：挂起一秒、阻塞一秒、cpu密集型操作和内存密集型操作（其中大部分时间花在访问、分配和释放内存上）。不同的行表示用于这些协程的不同调度器。下表显示了平均执行时间（以毫秒为单位）：

|              | 挂起   | 阻塞     | CPU密集型 | 内存密集型 |
| ------------ | ---- | ------ | ------ | ----- |
| 单线程          | 1002 | 100003 | 39103  | 94358 |
| Default(8线程) | 1002 | 13003  | 8473   | 21461 |
| IO(64线程)     | 1002 | 2003   | 9893   | 20776 |
| 100个线程       | 1002 | 1003   | 16379  | 21004 |

你可以观察到一些重要的事情：

1. 当我们只是挂起时，使用多少线程并不重要
2. 当我们阻塞时，我们使用的线程越多，完成所有这些协程的速度就越快
3. 对于 CPU密集型操作， `Dispatcher.Default` 是最佳的选择
4. 如果我们处理 IO 密集型操作，更多的线程会提供一些改进（但不显著）

下面是被测试的函数样子：

```kotlin
fun cpu(order: Order): Coffee {
    var i = Int.MAX_VALUE
    while (i > 0) {
        i -= if (i % 2 == 0) 1 else 2
    }
    return Coffee(order.copy(customer = order.customer + i))
}

fun memory(order: Order): Coffee {
    val list = List(1_000) { it }
    val list2 = List(1_000) { list }
    val list3 = List(1_000) { list2 }
    return Coffee(
        order.copy(
            customer = order.customer + list3.hashCode()
        )
    )
}

fun blocking(order: Order): Coffee {
    Thread.sleep(1000)
    return Coffee(order)
}

suspend fun suspending(order: Order): Coffee {
    delay(1000)
    return Coffee(order)
}
```

### 总结

Dispatchers 决定协程在哪个线程或线程池上运行（启动和恢复）。最重要的选项有：

* `Dispatchers.Defualt`：被用来执行 CPU 密集型操作
* `Dispatchers.Main`：被我们用来访问主线程，例如在 Android、Swing 或者 JavaFX 上
* `Dispatchers.Main.immediate`：它和 `Dispatchers.Main` 运行在同一个线程上，但如果没有必要，它不会重新调度
* `Dispatchers.IO`：被用来执行一些阻塞线程的操作
* 调用了 `limitParallelism`的 `Dispatchers.IO` 或者带有自定义线程池的 `Dispatcher.IO`：我们用来处理大量的阻塞调用
* 调用了 `limitParallelism` 并设置为1的 `Dispatchers.Default` 或 `Dispatchers.IO`，或具有单个线程的自定义调度器：被用来修改共享状态
* `Dispatchers.Unconfined`：当我们不关心协程在哪个线程上被挂起时使用
