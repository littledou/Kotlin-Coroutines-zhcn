# 底层中的协程

有这么一种人，他们不能接受仅仅是只会开车，他们还想要了解引擎盖下面是如何运作了。我就是这样的人，所以我必须弄清楚协程是如何工作的。如果你也是这样的人，你也会喜欢本章的。如果不是的话，你可以跳过本章。

本章不会介绍任何你可能会用到的工具，只是纯粹的阐述。我将尝试在一个令所有人都接受的层级上解释协程是如何工作的。重点如下：

* 挂起函数就像状态机一样，在函数的开始，以及每次挂起协程后，都会有一个不同状态
* 标识状态的数字和本地数据都保存在了 `continuation` 对象中 （注：我们上一章中提到了 `Continuation`，它是一个概念，也是一个类型。它就像是游戏中的存档点，本章我们将着重讨论它）
* 一个函数的 `Continuation` 将它之后的调用行为包装了起来。因此，所有这些 `Continuation` 表示的是恢复函数的调用栈，

如果你有兴趣了解一些内部结构（当然是简化的），请继续阅读一下内容。

### 传递 Continuation 的方式

有几种方式可以实现挂起功能，但 Kotlin 团队决定采用一种称为“传递 `Continuation`”的方式。这意味着 `continuation` 将作为参数从一个函数传递到另外一个函数。按照惯例，`continuation` 位于最后一个参数位置：

```kotlin
suspend fun getUser(): User?
suspend fun setUser(user: User)
suspend fun checkAvailability(flight: Flight): Boolean

// 这些函数在底层的样子
fun getUser(continuation: Continuation<*>): Any?
fun setUser(user: User, continuation: Continuation<*>): Any
fun checkAvailability(
    flight: Flight,
    continuation: Continuation<*>
): Any
```

你可能还注意到，底层函数的返回类型不同于最初的返回类型，它变成了 `Any` 或 `Any?`，为什么会这样呢？原因是挂起函数可能会被挂起，所以这个时候它不能返回被声明的类型。在这种情况下，它返回一个特殊的 `COROUTINE_SUSPENDED` 标记，稍后我们将在实践中看到。现在，只需要注意，因为 `getUser` 可能返回 `User?` 或 `COROUTINE_SUSPENDED`（其类型为 `Any`）两种数据，所以该函数返回类型必须是 `User?` 和 `Any` 的超类，所以就只能是 `Any?` 了。 也许有一天 Kotlin 会引入联合类型，在这种情况下我们就可以使用 `User? | COROUTINE_SUSPENDED` 来代替了。

### 一个非常简单的函数

为了更深入地理解，让我们从一个非常简单的函数开始。该函数在 `delay` 前后打印一些东西：

```kotlin
suspend fun myFunction() {
    println("Before")
    delay(1000) // suspending
    println("After")
}
```

你已经可以推断出 `myFunction` 的函数签名在底层是什么样的了：

```kotlin
fun myFunction(continuation: Continuation<*>): Any
```

接下来，这个函数需要要一个属于它的 `continuation` 来记住它的状态。让我们将其命名为 `MyFunctionContinuation`（实际的 `continuation` 是一个对象表达式，没有名称，但是这样解释会更加容易）。在函数体的开头， `MyFuntion` 会将自己的 `continuation`（参数）包装到自己的 `continuation` （MyFunctionContinuation） 中。

```kotlin
val continuation = MyFunctionContinuation(continuation)
```

上面的行为只有在 `continuation` 还没有包装的情况下才会包装。如果已经包装了的话，我们应该保持 `continuation` 不变，因为那时已经到了函数恢复的过程了（现在可能会让人疑惑，但稍后你会更好地明白这是为什么）。

```kotlin
val continuation =
    if (continuation is MyFunctionContinuation) continuation  // 如果已经包装，就保持不变
    else MyFunctionContinuation(continuation)   // 否则，包装 continuation
```

这个条件表达式可以简化为：

```kotlin
val continuation = continuation as? MyFunctionContinuation
    ?: MyFunctionContinuation(continuation)
```

最后，让我们来谈谈函数体：

```kotlin
suspend fun myFunction() {
    println("Before")
    delay(1000) // suspending
    println("After") 
}
```

函数可以从两个地方开始：

1. 开始（在第一次调用的情况下）
2. 被挂起之后（被 `continuation` 恢复的情况下）

为了标识当前状态，我们使用了一个名为 `label` 的字段。初始值是 0，代表函数刚开始调用。然后，函数在执行到每个挂起点之前，`label` 都会被设置到下一个状态， 由此，我们就可以根据 `label` 从恢复后的挂起点开始运行函数了。

```kotlin
// myFunction 在底层简化过后的代码
fun myFunction(continuation: Continuation<Unit>): Any {
    val continuation = continuation as? MyFunctionContinuation
        ?: MyFunctionContinuation(continuation)
    
    if (continuation.label == 0) {
        println("Before")
        continuation.label = 1
        if (delay(1000, continuation) == COROUTINE_SUSPENDED) {
            return COROUTINE_SUSPENDED
        } 
    }
    
    if (continuation.label == 1) {
        println("After")
        return Unit
    }
    error("Impossible") 
}
```

最后一个重点也在上面的代码片段中展示出来。当 `delay` 函数挂起时，它返回 `COROUTINE_SUSPENDED` ，然后 `myFunction` 返回 `COROUTINE_SUSPENDED`，就像递归一样，调用它的函数，调用该函数的函数...直到顶层调用链，所有的函数都要执行先沟通的操作，即返回这个 `COROUTINE_SUSPENDED`。这就是挂起函数如何暂停所有这些函数，并留下线程供其他可运行对象（包括协程）继续工作的方式。

让我们继续分析上面的代码，如果这个 `delay` 没有返回 `COROUTINE_SUSPENDED` 会发生什么呢？如果它只是返回 `Unit` （我们知道它不会，但是让我们假设一下）呢？请注意，如果 `continuation` 只返回了 `Unit`，我们就会进入到下一个状态判断去，函数的行为和其它函数一样。

现在，让我们讨论 `continuation`，它被实现成一个匿名类，简化后，它是这样的：

```kotlin
cont = object : ContinuationImpl(continuation) {
    var result: Any? = null
    var label = 0
    override fun invokeSuspend(`$result`: Any?): Any? {
        this.result = `$result`;
        return myFunction(this);
    }
}
```

为了提高函数可读性，我前面将其表示为一个名为 `MyFunctionContinuation` 的类。我还决定通过内联 `ContinuationImpl` 主体来隐藏继承。为了简化，我跳过了许多优化和机能化，只保留必要的代码部分。

> 在 JVM 中，类型参数在编译期间会被删除，例如 `Continuation<Unit>` 或 `Continuation<String>` 最后将是 `Continuation`。因为我们在这里展示的所有内容都是用 Kotlin 表示的 JVM 字节码，所以你跟不用担心这些类型参数的问题。

下面的代码就是简化了完全的 `myFunction` 在底层的样子：

```kotlin
fun myFunction(continuation: Continuation<Unit>): Any {
    val continuation = continuation as? MyFunctionContinuation
        ?: MyFunctionContinuation(continuation)
    if (continuation.label == 0) {
        println("Before")
        continuation.label = 1
        
        if (delay(1000, continuation) == COROUTINE_SUSPENDED){
            return COROUTINE_SUSPENDED
        }
    }
    if (continuation.label == 1) {
        println("After")
        return Unit
    }
    error("Impossible") 
}
    
class MyFunctionContinuation(
    val completion: Continuation<Unit>
) : Continuation<Unit> {
    override val context: CoroutineContext
        get() = completion.context

    var label = 0
    var result: Result<Any>? = null
    override fun resumeWith(result: Result<Unit>) {
        this.result = result
        val res = try {
            val r = myFunction(this)
            if (r == COROUTINE_SUSPENDED) return
            Result.success(r as Unit) 
        } catch (e: Throwable) {
            Result.failure(e)
        }
        completion.resumeWith(res)
    } 
}
```

如果你想自己分析挂起函数的底层运作，在Intellij IDEA 中打开 Tool -> Kotlin -> Show Koltin bytecode， 然后点击 “Decompile” 按钮，结果你将看到这段代码被反编译成 Java（这些代码或多或少使用 Java 编写的样子）。

![在这里插入图片描述](https://img-blog.csdnimg.cn/89ec38e8b96b4b4a9c39c8b29121efb7.png) ![在这里插入图片描述](https://img-blog.csdnimg.cn/3cf6111c81e244f2b139e449dac94e70.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/7a923946d5a344c68c27566baae05323.png)

### 带有状态的函数

如果函数有一些状态（比如局部变量或参数）需要在挂起后恢复。那么这个状态需要在函数的 `continuation` 中持有，让我们思考下面这个函数：

```kotlin
suspend fun myFunction() {
    println("Before")
    var counter = 0 // myFunction 的一个状态（局部变量）
    delay(1000) // suspending
    counter++
    println("Counter: $counter")
    println("After") 
}
```

这里 `counter` 在两种状态下都是被需要的（对于等于0和等于1的 `label` ），因此需要在 `continuation` 中保留它。它会在挂起之前被存储起来，函数开始后则会恢复这些属性。下面是（简化后的）函数在底层的样子：

```kotlin
fun myFunction(continuation: Continuation<Unit>): Any {
    val continuation = continuation as? MyFunctionContinuation
        ?: MyFunctionContinuation(continuation)
    var counter = continuation.counter
    
    if (continuation.label == 0) {
        println("Before")
        counter = 0
        continuation.counter = counter
        continuation.label = 1
        
        if (delay(1000, continuation) == COROUTINE_SUSPENDED){
            return COROUTINE_SUSPENDED
        }
    }
    
    if (continuation.label == 1) {
        counter = (counter as Int) + 1
        println("Counter: $counter")
        println("After")
        return Unit
    }
    error("Impossible") 
}

class MyFunctionContinuation(
    val completion: Continuation<Unit>
) : Continuation<Unit> {
    
    override val context: CoroutineContext
        get() = completion.context
    
    var result: Result<Unit>? = null
    var label = 0

    var counter = 0  // 这里将状态存储起来了
    
    override fun resumeWith(result: Result<Unit>) {
        this.result = result
        val res = try {
            val r = myFunction(this)
            if (r == COROUTINE_SUSPENDED) return
            Result.success(r as Unit)
        } catch (e: Throwable) {
            Result.failure(e)
        }
        completion.resumeWith(res)
    }
}
```

### 带值恢复的函数

如果我们期望从挂起函数获取一些数据，情况会略有不同，让我们来分析下面函数：

```kotlin
suspend fun printUser(token: String) {
    println("Before")
    val userId = getUserId(token) // suspending
    println("Got userId: $userId")
    val userName = getUserName(userId, token) // suspending
    println(User(userId, userName))
    println("After") 
}
```

这里有两个挂起函数： `getUserId` 和 `getUserName`，我们还添加了一个参数 `token`，我们的挂起函数还返回了一些值。下面这些信息都需要存储在 `Continuation` 中：

* token， 因为在状态 0 和 状态 1 都使用了它
* userId， 因为在状态 1 和 状态 2 都使用了它
* `Result` 类型的 result，表示函数如何恢复

如果函数恢复了一个值，结果将是 `Result.Success(vaule)`，在这种情况下，我们可以获得并使用这个值。 如果在异常情况下，结果将是 `Result.Failure(exception)`。在这种情况下，将会抛出此异常。

```kotlin
fun printUser(
    token: String,
    continuation: Continuation<*>
): Any {
    val continuation = continuation as? PrintUserContinuation
        ?: PrintUserContinuation(
            continuation as Continuation<Unit>,token)
        
    var result: Result<Any>? = continuation.result
    var userId: String? = continuation.userId
    val userName: String
    
    if (continuation.label == 0) {
        println("Before")
        continuation.label = 1
        val res = getUserId(token, continuation)
        if (res == COROUTINE_SUSPENDED) {
            return COROUTINE_SUSPENDED
        }
        result = Result.success(res)
    }
    
    if (continuation.label == 1) {
        userId = result!!.getOrThrow() as String
        println("Got userId: $userId")
        continuation.label = 2
        continuation.userId = userId
        val res = getUserName(userId, continuation)
        if (res == COROUTINE_SUSPENDED) {
            return COROUTINE_SUSPENDED
        }
        result = Result.success(res)
    }
    
    if (continuation.label == 2) {
        userName = result!!.getOrThrow() as String
        println(User(userId as String, userName))
        println("After")
        return Unit
    }
    error("Impossible") 
}

class PrintUserContinuation(
    val completion: Continuation<Unit>,
    val token: String
) : Continuation<String> {
    override val context: CoroutineContext
        get() = completion.context
    var label = 0
    var result: Result<Any>? = null
    var userId: String? = null
    override fun resumeWith(result: Result<String>) {
        this.result = result
        val res = try {
            val r = printUser(token, this)
            if (r == COROUTINE_SUSPENDED) return
            Result.success(r as Unit)
        } catch (e: Throwable) {
            Result.failure(e)
        }
    completion.resumeWith(res)
    }
}
```

### 调用栈

当函数 a 调用函数 b 时，虚拟机需要将函数 a 的状态存储在某个地方，同样还有函数 b 完成时执行返回的地址。所有这些都存储在一个成为**调用栈**的结构中。 问题是，当我们挂起时，我们释放了一个线程，我们因此清空了调用栈。所以，当我们恢复时，调用栈是没有用的。相反， `continuation` 充当了调用栈。每个 `continuation` 都会在我们挂起时（作为 `label`）保留函数的局部变量和参数，以及对调用该函数的函数的 `continuation` 的引用。一个 `continuation` 引用另一个 `continuation`，后者又引用另一个 `continutaion`，从此往复。 因此，我们的 `continuation` 就像一个巨大的洋葱：它保留了常规保存在调用栈中的所有内容。看看下面的例子：

```kotlin
suspend fun a() {
    val user = readUser()
    b()
    b()
    b()
    println(user)
}

suspend fun b() {
    for (i in 1..10) {
        c(i)
    }
}

suspend fun c(i: Int) {
    delay(i * 100L)
    println("Tick") 
}
```

i 等于4时，一个 `continuation` 引用可以表示为：

```kotlin
CContinuation(
    i = 4,
    label = 1,
    completion = BContinuation(
        i = 4,
        label = 1,
        completion = AContinuation(
            label = 2,
            user = User@1234,
            completion = ...
        )
    ) 
)
```

看看上面的表示，“Tick”会被打印了多少次（假设 readuUser 不是一个挂起函数）？ （答案是13次）

当一个 `continuation` 被恢复时，每个 `continuation` 首先调用它的函数，完成此操作后，`continutaion` 将恢复调用该函数的函数的 `continutaion`。这个 `continutaion` 再调用它的函数，这个过程循环往复，直到到达调用链的顶部。

```kotlin
override fun resumeWith(result: Result<String>) {
    this.result = result
    val res = try {
        val r = printUser(token, this)
        if (r == COROUTINE_SUSPENDED) return
        Result.success(r as Unit)
    }
    catch (e: Throwable) {
        Result.failure(e)
    }
    completion.resumeWith(res)
}
```

例如，思考这样一个情况： 函数 a 调用函数 b，函数 b 调用挂起函数 c。在恢复的过程中， c 的`continutaion` 首先恢复 c 函数，一旦这个函数完成， c 的 `continutaion` 将会调用 b 函数的 `continutaion`。完成后， b 的 `continutaion` 调用 a 的 `continutaion`，后者调用 a 的函数。

整个过程可以用下面这张图片来展示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/5cafdfa036524f6aad84bb77d8b6db40.png)与异常看起来相似：未捕获的异常在 `resumeWith` 中被捕获，然后用 `Result.failure(e)` 包装，然后再用这个结果恢复调用我们函数的函数。

我希望这些都能让你们对挂起时发生的事情有一些了解。状态需要存储在 `continutaion` 中，需要支持挂起机制，恢复时，需要从 `continutaion` 恢复状态，并使用结果或抛出异常。

![···图片···](https://img-blog.csdnimg.cn/3731c0c15fe44449825e7c59c79ba41d.png)

### 实际代码

`continutaion` 和挂起函数编译后的实际代码更为复杂，因为它包含了一些优化和额外的机制，比如：

* 构建了更好的异常堆栈跟踪
* 增加了协程挂起拦截的特性（我们将在后面讨论这个特性）
* 不同级别的优化（如删除未使用的变量，或尾调用优化）

下面是来自 Kotlin 1.5.30 版本的 `BaseContinuationImpl` 的一部分，它展示了实际 `resumeWith` 代码（还有其他方法和一些跳过的注释）：

```kotlin
internal abstract class BaseContinuationImpl(
    val completion: Continuation<Any?>?
) : Continuation<Any?>, CoroutineStackFrame, Serializable {
    // This implementation is final. This fact is used to
    // unroll resumeWith recursion.
    final override fun resumeWith(result: Result<Any?>) {
        // This loop unrolls recursion in
        // current.resumeWith(param) to make saner and
        // shorter stack traces on resume
        var current = this
        var param = result
        while (true) {
            // Invoke "resume" debug probe on every resumed
            // continuation, so that a debugging library
            // infrastructure can precisely track what part
            // of suspended call stack was already resumed
            probeCoroutineResumed(current)
            with(current) {
                val completion = completion!! // fail fast
                // when trying to resume continuation
                // without completion
                val outcome: Result<Any?> =
                try {
                    val outcome = invokeSuspend(param)
                    if (outcome === COROUTINE_SUSPENDED)
                    return
                    Result.success(outcome)
                } catch (exception: Throwable) {
                    Result.failure(exception)
                }
                releaseIntercepted()
                // this state machine instance is terminating
                if (completion is BaseContinuationImpl) {
                    // unrolling recursion via loop
                    current = completion
                    param = outcome
                } else {
                    // top-level completion reached --
                    // invoke and return
                    completion.resumeWith(outcome)
                    return
                }
            } 
        } 
    }
    // ...
}
```

如你所见，它使了循环而非递归，这一更改允许实际代码进行一些优化和简化

### 挂起函数的性能

使用挂起函数而不是常规函数的成本是多少呢？当我们看到其内部结构时，许多人可能会产生这样的印象：成本是巨大的。但并非这样的。将函数划分为多个状态的代价是很低的，因为数字比较和执行跳转的成本几乎为0，在 `continuation` 中保存状态也很廉价，因为我们不复制局部变量：我们让新变量指向内存中相同引用处。唯一需要的开销是创建一个 `continuation` 对象。但这仍然不是什么大问题，如果你不担心 RxJava 或回调函数的性能，那你也不需要担心挂起函数的性能。

### 总结

实际上，协程的内部结构要比我所描述的复杂的多，但我希望你对协程的内部结构有一些了解。关键的点是：

* 挂起函数就像状态机一样，在函数的开始，以及每次挂起协程后，都会有一个不同状态
* 标识状态的数字和本地数据都保存在了 `continuation` 对象中 （注：我们上一章中提到了 `Continuation`，它是一个概念，也是一个类型。它就像是游戏中的存档点，本章我们将着重讨论它）
* 一个函数的 `Continuation` 将它之后的调用行为包装了起来。因此，所有这些 `Continuation` 表示的是恢复函数的调用栈，
