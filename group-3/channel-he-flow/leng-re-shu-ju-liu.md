# 冷热数据流

在这里插入图片描述]\(https://img-blog.csdnimg.cn/d2893ae770c34a7b924a3372997cbafb.png) Kotlin 协程最初只有 Channel，但创造者意识到这个是不够的。 Channel 是一种热数据流，但是我们经常需要冷数据流。

![···图片··](https://img-blog.csdnimg.cn/c9959653a0084828b9e26e7ced9d055a.png)

了解热数据流和冷数据流之间的区别可以帮助我们更好的学习 Flow 和其他相关的技术，因为你日常使用的大多数数据源都属于这两类之一。集合（List、Set）是热的，而序列和 Java Stream 是冷的。Channel 是热的，而 Flow 和 RxJava流是冷的。

| 热数据流                   | 冷数据流                 |
| ---------------------- | -------------------- |
| Collections(List, Set) | Sequence, Stream     |
| Channel                | Flow, RxJava streams |

### 热 vs 冷

热数据流是急切、即时的，和消费者互相独立，生产和存储元素。冷数据流是懒惰的，按需执行操作，不需要存储任何东西

我们可以在使用列表（热）和序列（冷）时了解到这些差异。热数据流的构建器和操作都是立即开始的，而在冷数据流中，直到元素被需要使用时才会去生产它们。

```kotlin
@OptIn(ExperimentalStdlibApi::class)
fun main() {
    val l = buildList {
        repeat(3) {
            add("User$it")
            println("L: Added User")
        }
    }

    val l2 = l.map {
        println("L: Processing")
        "Processed $it"
    }

    val s = sequence {
        repeat(3) {
            yield("User$it")
            println("S: Added User")
        }
    }

    val s2 = s.map {
        println("S: Processing")
        "Processed $it"
    }
}
// L: Added User
// L: Added User
// L: Added User
// L: Processing
// L: Processing
// L: Processing
```

因此，冷数据流（如 `Sequence`、`Stream` 或 `Flow`）:

* 可以是无限的
* 使用尽量少的操作次数
* 使用更少的内存（不需要分配所有中间集合产物）

序列处理做的操作更少，因为它延迟处理元素。它的工作也方式非常简单：每个中间操作（如 `map` 或 `filter`）只是用于装饰前面的序列。终端操作才是作为结束点来完成所有工作。请看下面的例子，使用序列，`find` 会查询 `map` 后的第一个符合条件的元素。它从 `sequenceOf`（返回1）返回的序列，然后将其映射（1\*1到1），并将结果返回给过滤器`find`。过滤器检查该元素是否满足其条件。如果元素不满足条件，过滤器就会一次又一次地询问。直到找到合适的元素为止。

这与列表的处理非常不同，列表处理在每一个中间步骤都要进行计算并返回一个完全处理过的中间集合产物。这就是集合处理元素的顺序不同、需要更多的内存、可能需要更多操作的原因（如下面例子所示）。

```kotlin
fun m(i: Int): Int {
    print("m$i ")
    return i * i
}

fun f(i: Int): Boolean {
    print("f$i ")
    return i >= 10
}

fun main() {
    listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
        .map { m(it) }
        .find { f(it) }
        .let { print(it) }
    // m1 m2 m3 m4 m5 m6 m7 m8 m9 m10 f1 f4 f9 f16 16
    
    sequenceOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
        .map { m(it) }
        .find { f(it) }
        .let { print(it) }
    
    // m1 f1 m2 f4 m3 f9 m4 f16 16
}
```

这意味着列表是元素的集合，而序列只是针对元素的操作符集的定义。热数据流：

* 随时可以使用（每个操作都是终端操作）
* 多次使用时不需要重新计算结果

```kotlin
fun m(i: Int): Int {
    print("m$i ")
    return i * i
}

fun main() {
    val l = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
        .map { m(it) } // m1 m2 m3 m4 m5 m6 m7 m8 m9 m10
    
    println(l) // [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
    println(l.find { it > 10 }) // 16
    println(l.find { it > 10 }) // 16
    println(l.find { it > 10 }) // 16
    
    val s = sequenceOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
        .map { m(it) }
    
    println(s.toList())
    // [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
    
    println(s.find { it > 10 }) // m1 m2 m3 m4 16
    println(s.find { it > 10 }) // m1 m2 m3 m4 16
    println(s.find { it > 10 }) // m1 m2 m3 m4 16
}
```

Java流与 Kotlin 序列具有相同的特征，它们都是冷的数据流。

### 热的 channel，冷的 flow

是时候回到协程了。创建 flow 最简单的方式是使用构建器，它类似于 `produce` 函数，这个函数被称为 `flow`：

```kotlin
val channel = produce {
    while (true) {
        val x = computeNextValue()
        send(x)
    }
}

val flow = flow {
    while (true) {
        val x = computeNextValue()
        emit(x)
    }
}
```

这些构建器在概念上是等价的，但由于 channel 和 flow 的行为有很大的区别，这两个函数之间也有重要的区别。看看下面的例子，channel 是热的，所以它们会立即开始计算。这个计算需要从一个单独的协程开始。这就是为什么 `produce` 需要成为一个协程构建器，它被定义为 `CoroutineScope` 上的扩展函数。计算立即开始，但由于默认的缓冲区大小是0（`RENDEZVOUS`），所以它很快就会被挂起，直到下面例子中的消费者准备好。注意，当没有接收者时，停止生产和按需生产是有区别的。channel 作为热数据流，独立于其消费者去产生元素，然后存储它们。它们不会关心有多少消费者，因为每个元素只能被接收一次。在第一个接收端消费了所有元素后，第二个接收端就会找一个空的、已经关闭的 channel，这就是它根本不会接收到任何元素的原因。

```kotlin
private fun CoroutineScope.makeChannel() = produce {
    println("Channel started")
    for (i in 1..3) {
        delay(1000)
        send(i)
    }
}

suspend fun main() = coroutineScope {
    val channel = makeChannel()
    delay(1000)
    println("Calling channel...")
    for (value in channel) { println(value) }
    println("Consuming again...")
    for (value in channel) { println(value) }
}
// Channel started
// (1 sec)
// Calling channel...
// 1
// (1 sec)
// 2
// (1 sec)
// 3
// Consuming again...
```

使用 Flow 进行相同的处理是非常不同的。由于它是一个冷数据流。所以生产是按需的。这意味着 flow 不是一个协程构建器，不需要进行任何处理。它只是一个定义在使用终端操作（如 `collect`）时元素应该如何被生产的指令集。它在执行它终端操的作用域上运行（它从挂起函数的 continuation 中获取作用域，就像 coroutineScope 和其他协程作用域函数一样），这就 `flow` 构建器不需要一个 `CoroutineScope` 的原因。Flow 上的每个终端操作都将从头开始。请你认真比较上面和下面的例子，因为它们展示了 channel 和 flow 之间的关键区别：

```kotlin
private fun makeFlow() = flow {
    println("Flow started")
    for (i in 1..3) {
        delay(1000)
        emit(i)
    }
}

suspend fun main() = coroutineScope {
    val flow = makeFlow()
    delay(1000)
    println("Calling flow...")
    flow.collect { value -> println(value) }
    println("Consuming again...")
    flow.collect { value -> println(value) }
}
// (1 sec)
// Calling flow...
// Flow started
// (1 sec)
// 1
// (1 sec)
// 2
// (1 sec)
// 3
// Consuming again...
// Flow started
// (1 sec)
// 1
// (1 sec)
// 2
// (1 sec)
// 3
```

RxJava 流与 Kotlin 的 Flow 有很多相似之处，有些人甚至说， Flow 可以被称为 “RxCoroutines”。

### 总结

大多数数据源不是热的就是冷的：

* 热数据很迫切，它们尽可能快的生产元素并存储它们。它们创造的元素独立于它们的消费者，它们是集合（`List` 、`Set`）和 channel
* 冷数据流是惰性的，它们在终端操作上按需处理元素，所有中间函数知识定义应该做什么（通常是用装饰模式），它们通常不存储元素，而是根据需要创建元素，它们的运算次数很少，可以是无限的，它们创建、处理元素的过程通常和消费过程紧挨着。这些元素是 `Sequence`、`Java Stream`，`Flow`和 `RxJava 流`（`Observable`、`Single` 等）

这解释了 Channel 和 Flow 的本质区别。现在是时候来讨论后者所支持的特性了。
