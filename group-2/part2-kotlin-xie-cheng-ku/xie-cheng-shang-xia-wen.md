# 协程上下文

如果查看协程构建器的定义，你将看到它们的第一个参数类型是 `CoroutineContext`。

```kotlin
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    ...
}
```

函数的接收者和最后一个参数的接收者都是 `CoroutineScope` 类型，这个 `CoroutineScope` 似乎是一个重要的概念，所以来看看它的定义：

```kotlin
public interface CoroutineScope {
    public val coroutineContext: CoroutineContext
}
```

它似乎只是 `CoroutineContext` 的包装器，由此你可能会想到 `Continuation` 是如何定义的：

```kotlin
public interface Continuation<in T> {
    public val context: CoroutineContext
    public fun resumeWith(result: Result<T>)
}
```

`Continuation` 也包含着 `CoroutineContext`。既然 Kotlin 最重要的协程元素也用着它，那么它一定是一个非常重要的概念，它是什么呢？

### CoroutineContext 接口

`CoroutineContext` 是一个表示元素或元素集合的接口。它在概念上类似于 `map` 或者 `set` ：它是有索引的 `Element` 实例集，如 `Job`、`CoroutineName`、`CoroutineDispatcher` 等。不同寻常的是，每个 `Element` 也是一个 `CoroutineContext`。因此，集合中的每个元素自己本身就是一个集合。

这个概念很直观，想象一个杯子，它是单个元素，但是也是包含多个元素的集合。当你添加另一个杯子时，你就拥有了一个包含了两个元素的集合。

为了方便地规范和修改上下文，每个 `CoroutineContext` 的元素本身就是一个 `CoroutineContext`，如下面的例子所示（添加上下文和设置协程构建器上下文将在后面解释）。仅仅指定或添加上下文要比显示的创建集合要容易的多：

```kotlin
launch(CoroutineName("Name1")) { ... }
launch(CoroutineName("Name2") + Job()) { ... }
```

这个集合中的每一个元素都有唯一的 Key 来标识它。这些键通过引用进行比较。

例如 `CoroutineName` 或 `Job` 实现了 `CoroutineContext.Element` 接口，而该接口又实现了 `CoroutineContext` 接口。

```kotlin
fun main() {
    val name: CoroutineName = CoroutineName("A name")
    val element: CoroutineContext.Element = name
    val context: CoroutineContext = element

    val job: Job = Job()
    val jobElement: CoroutineContext.Element = job
    val jobContext: CoroutineContext = jobElement
}
```

这与 `SuperviseJob`、`CoroutineExceptionHandler` 和 `Dispatcher` 中的分发器是一样的。这些是最重要的协程上下文。它们将在下一章中解释。

## 在 CoroutineContext 中找到元素

由于 `CoroutineContext` 类似于集合，我们可以使用 `get` 找到具有相同键的元素。另一种选择是使用方括号，因为在 Kotlin 中，`get` 方法是一个操作符，可以使用方括号来调用。就像在 Map 中，当元素在上下文时，它将被返回，否则返回null。

```kotlin
fun main() {
    val ctx: CoroutineContext = CoroutineName("A name")

    val coroutineName: CoroutineName? = ctx[CoroutineName]
    // or ctx.get(CoroutineName)
    println(coroutineName?.name) // A name

    val job: Job? = ctx[Job] // or ctx.get(Job)
    println(job) // null
}
```

要查找 `CoroutineName`，我们只需传入 `CoroutineName`。它不是类或类型，而是一个伴生对象。这是 Kotlin 中的一个特性：一个类的名字可以被用做它的伴生对象的引用，所以 `ctx[CoroutineName]` 是 `ctx[CoroutineName.Key]` 的便捷写法。

```kotlin
data class CoroutineName(
    val name: String
) : AbstractCoroutineContextElement(CoroutineName) {
    
    override fun toString(): String = "CoroutineName($name)"
    
    companion object Key : CoroutineContext.Key<CoroutineName>
}
```

这是 Kotlinx.coroutines 的常见做法，使用伴生对象作为作为同名元素的键。这样更加容易记住。一个键可能指向一个类（CoroutineName），或一个接口（Job），该接口有许多具有相同键的类实现（如 `Job` 和 `SupervisorJob`）：

```kotlin
interface Job : CoroutineContext.Element {
    companion object Key : CoroutineContext.Key<Job>
    // ...
}
```

### 添加上下文

`CoroutineContext` 真正有用的地方在于它能够将两者合并到一起。

当添加两个具有不同键的元素时，最终的上下文将会响应这两个键。

```kotlin
fun main() {
    val ctx1: CoroutineContext = CoroutineName("Name1")
    println(ctx1[CoroutineName]?.name) // Name1
    println(ctx1[Job]?.isActive) // null
    
    val ctx2: CoroutineContext = Job()
    println(ctx2[CoroutineName]?.name) // null
    println(ctx2[Job]?.isActive) // true, 因为 “Active” 是job创建后的初始状态

    val ctx3 = ctx1 + ctx2
    println(ctx3[CoroutineName]?.name) // Name1
    println(ctx3[Job]?.isActive) // true
```

当添加具有相同的键的另一个元素时，就像在 `map` 中一样，新元素将替换前一个元素。

```kotlin
fun main() {
    val ctx1: CoroutineContext = CoroutineName("Name1")
    println(ctx1[CoroutineName]?.name) // Name1

    val ctx2: CoroutineContext = CoroutineName("Name2")
    println(ctx2[CoroutineName]?.name) // Name2
    
    val ctx3 = ctx1 + ctx2
    println(ctx3[CoroutineName]?.name) // Name2
}
```

### 空的协程上下文

因为 `CoroutineContext` 就像一个集合，所以我们也有一个空的上下文。这样的上下文本身不返回任何元素，如果我们把它添加到另一个上下文去，最终的行为和被添加的上下文完全一样：

```kotlin
fun main() {
    val empty: CoroutineContext = EmptyCoroutineContext
    println(empty[CoroutineName]) // null
    println(empty[Job]) // null

    val ctxName = empty + CoroutineName("Name1") + empty
    println(ctxName[CoroutineName]) // CoroutineName(Name1)
}
```

### 删去元素

还可以使用 `minusKey` 函数通过传入元素的键，从上下文中删除指定元素。

`CoroutineContext` 的减号操作符没有被重载，我认为这是因为它的含义还不够清晰，正如 《Effective Kotlin》中_第12条：操作符的行为应该与其名称一致_所阐述的那样。

```kotlin
fun main() {
    val ctx = CoroutineName("Name1") + Job()
    println(ctx[CoroutineName]?.name) // Name1
    println(ctx[Job]?.isActive) // true

    val ctx2 = ctx.minusKey(CoroutineName)
    println(ctx2[CoroutineName]?.name) // null
    println(ctx2[Job]?.isActive) // true
    
    val ctx3 = (ctx + CoroutineName("Name2"))
        .minusKey(CoroutineName)
    println(ctx3[CoroutineName]?.name) // null
    println(ctx3[Job]?.isActive) // true
}
```

### 折叠上下文

如果我们需要对上下文中的每个元素执行某些操作，可以使用 `fold` 方法，该方法与集合的 `fold` 功能类似，它具备：

* 累加器初始值
* 根据累加器的当前状态和当前被调用的元素，生成累加器下一个状态的操作

```kotlin
fun main() {
    val ctx = CoroutineName("Name1") + Job()
    
    ctx.fold("") { acc, element -> "$acc$element " }
        .also(::println)
    // CoroutineName(Name1) JobImpl{Active}@dbab622e
    
    val empty = emptyList<CoroutineContext>()
    ctx.fold(empty) { acc, element -> acc + element }
        .joinToString()
        .also(::println)
    // CoroutineName(Name1), JobImpl{Active}@dbab622e
}
```

### 协程上下文和构建器的关系

所以，**`CoroutineContext` 只是保存和传递数据的一种方式**。默认情况下，父协程将上下文传递给子协程，这是父协程与子协程关系产生的一种效果。我们会这样描述：子协程继承了父协程的上下文。

```kotlin
fun CoroutineScope.log(msg: String) {
    val name = coroutineContext[CoroutineName]?.name
    println("[$name] $msg")
}

fun main() = runBlocking(CoroutineName("main")) {
    log("Started") // [main] Started
    val v1 = async {
        delay(500)
        log("Running async") // [main] Running async
        42
    }
    
    launch {
        delay(1000)
        log("Running launch") // [main] Running launch
    }
    log("The answer is ${v1.await()}")
    // [main] The answer is 42
}
```

每个子协程都可以在参数中定义一个特定的上下文，这个上下文会覆盖来自父协程的上下文：

```kotlin
fun main() = runBlocking(CoroutineName("main")) {
    log("Started") // [main] Started
   
    val v1 = async(CoroutineName("c1")) {
        delay(500)
        log("Running async") // [c1] Running async
        42
    }

    launch(CoroutineName("c2")) {
        delay(1000)
        log("Running launch") // [c2] Running launch
    }

    log("The answer is ${v1.await()}")
    // [main] The answer is 42
}
```

一个计算协程上下文的简化公式是：

```kotlin
defaultContext + parentContext + childContext
```

子协程的上下文总是覆盖父协程上下文中具有相同键的元素。默认值仅用于未指定时。目前，当没有设置 `ContinuationInterceprot` 时，默认值为 `Dispatcher.Default` ，并且只有当应用程序在调试模式时才设置 `CoroutineId`。

有一个特殊的上下文叫做 `Job`，它是可变的，用于父协程与子协程的通信。接下来的章节将会专门讨论这种通信的影响。

### 在挂起函数中访问上下文

`CoroutineScope` 有一个可用于访问上下文的 `coroutineContext` 属性，但是在一个普通的挂起函数中，是如何拥有上下文呢？你可能还记得，在_底层的协程_中一章说到，上下文被 `continuation` 引用， `continuation` 被传递给每个挂起函数。因此，可以在挂起函数中访问父协程的上下文。为此，我们可以直接使用 `coroutineContext` 属性，该属性可以用于每个挂起的作用域。

```kotlin
suspend fun printName() {
    println(coroutineContext[CoroutineName]?.name)
}

suspend fun main() = withContext(CoroutineName("Outer")) {
    printName() // Outer
    launch(CoroutineName("Inner")) {
        printName() // Inner
    }
    delay(10)
    printName() // Outer
}
```

### 创建我们专属的上下文

这不是一个常见的需求，但是我们可以很容易地创建自己专属的协程上下文。为此，最简单的方法是创建一个实现了 `CoroutinContext.Element` 接口的类。这样的类需要 `CoroutineContext.Key<*>` 类型的属性作为键。此键将用作标识上下文的键。通常的做法是使用该类的伴生对象作为键。下面是一个非常简单的实现协程上下文的方式：

```kotlin
class MyCustomContext : CoroutineContext.Element {
    override val key: CoroutineContext.Key<*> = Key

    companion object Key :
        CoroutineContext.Key<MyCustomContext>
}
```

这样的上下文非常像 `CoroutineName`：它将父协程传递到子协程，但任何子协程都可以用相同键的不同上下文覆盖它。要在实践中了解这一点，下面可以看到一个用于打印连续数字的上下文示例：

```kotlin
class CounterContext(
    private val name: String
) : CoroutineContext.Element {
    override val key: CoroutineContext.Key<*> = Key
    private var nextNumber = 0
    
    fun printNext() {
        println("$name: $nextNumber")
        nextNumber++
    }
    
    companion object Key:CoroutineContext.Key<CounterContext>
}

suspend fun printNext() {
    coroutineContext[CounterContext]?.printNext()
}

suspend fun main(): Unit = withContext(CounterContext("Outer")) {
    printNext() // Outer: 0
    launch {
        printNext() // Outer: 1
        launch {
            printNext() // Outer: 2
        }
        launch(CounterContext("Inner")) {
            printNext() // Inner: 0
            printNext() // Inner: 1
            launch {
                printNext() // Inner: 2
            }
        }
    }
    printNext() // Outer: 3
}
```

我有看到自定义上下文被用作一种依赖注入的方式 —— 在生产环境中比测试环境中更容易注入不同的值，然而，我不认为这将成为标准的做法：

```kotlin
data class User(val id: String, val name: String)

abstract class UuidProviderContext : CoroutineContext.Element {
    abstract fun nextUuid(): String
    
    override val key: CoroutineContext.Key<*> = Key
    companion object Key :CoroutineContext.Key<UuidProviderContext>
}

class RealUuidProviderContext : UuidProviderContext() {
    override fun nextUuid(): String = UUID.randomUUID().toString()
}

class FakeUuidProviderContext(
    private val fakeUuid: String
) : UuidProviderContext() {
    override fun nextUuid(): String = fakeUuid
}

suspend fun nextUuid(): String =
    checkNotNull(coroutineContext[UuidProviderContext]) {
        "UuidProviderContext not present" 
    }
    .nextUuid()
    
// 下面是测试函数
suspend fun makeUser(name: String) = User(
    id = nextUuid(),
    name = name
)
        
suspend fun main(): Unit {
    // 生产环境中的用例
    withContext(RealUuidProviderContext()) {
        println(makeUser("Michał"))
        // e.g. User(id=d260482a-..., name=Michał)
    }

    // 测试用例
    withContext(FakeUuidProviderContext("FAKE_UUID")) {
        val user = makeUser("Michał")
        println(user) // User(id=FAKE_UUID, name=Michał)
        assertEquals(User("FAKE_UUID", "Michał"), user)
    }
}
```

### 总结

`CoroutineContext` 在概念上类似于集合或映射。它是有索引的 `Element` 实例集，其中每个 `Element` 也是一个 `CoroutinContext`，它里面的每个元素都有唯一的 `Key` 用来标识它。这样， `CoroutineContext` 就是一种通用的将对象分组并传递给协程的方法。这些对象由协程保存，并可以决定这些协程应该如何运行（它们的状态是什么，在哪个线程，等等）。在下一章中，我们将讨论 Kotlin 协程库中最重要的协程上下文。
