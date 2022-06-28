# 演员

在计算机科学中，有一种并发模型，称为演员模式（actor model），其最重要的概念就是演员，它是一个计算机实体，响应接收到的消息，它可以并发下面行为：

* 向其他演员发送有限数量的信息
* 创造有限数量的新演员
* 指定接收到下一个信息时应该发生行为

演员可以修改自己的私有状态，但只能通过消息传递间接的影响其它演员，因此不需要同步它们。每个演员运行在单个线程上，一个接一个地处理消息。

在 Kotlin 协程中，我们可以很容易地实现这个模型。我们使用 channel 将消息队列同步到演员上，然后我们只需要一个接一个处理这些消息的协程。在下面的代码片段中，你可以看到我们的 `massiveRun` 问题用演员模型解决了：

```kotlin
sealed class CounterMsg
object IncCounter : CounterMsg()
class GetCounter(
    val response: CompletableDeferred<Int>
) : CounterMsg()

fun CoroutineScope.counterActor(): Channel<CounterMsg> {
    val channel = Channel<CounterMsg>()
    launch {
        var counter = 0
        for (msg in channel) {
            when (msg) {
                is IncCounter -> {
                    counter++
                }
                is GetCounter -> {
                    msg.response.complete(counter)
                }
            }
        }
    }
    return channel
}

suspend fun main(): Unit = coroutineScope {
    val counter: SendChannel<CounterMsg> = counterActor()
    massiveRun { counter.send(IncCounter) }
    val response = CompletableDeferred<Int>()
    counter.send(GetCounter(response))
    println(response.await()) // 1000000
    counter.close()
}
```

这里不存在同步问题，因为演员在单个线程上工作。

为了简化这个模型，有一个 `actor` 协程构建器，它做了上述所示的事情（创建一个 channel 并启动一个协程），并为异常处理提供了更好的支持（例如，出现异常时会自行关闭 channel）。

```
fun CoroutineScope.counterActor() = actor<CounterMsg> {
    var counter = 0
    for (msg in channel) {
        when (msg) {
            is IncCounter -> counter++
            is GetCounter -> msg.response.complete(counter)
        }
    }
}
```

### 总结

演员模型是并行处理的一个重要模型。它目前在 Kotlin 中并不流行，但值得我们去了解，因为有一些案例完全适合使用它们，在这个模型中，最重要的概念是演员，它是响应消息的计算实体。因为它只操作一个协程，所以访问它的状态并没有冲突。我们可以用可能提供持久性或优先级的标准功能包装演员，从而创造许多有趣的可能。

曾经有一段时间，角色模型作为设计后端应用程序的一种主流方法（就像 Akka 一样），但在我看来，这似乎不是现在的趋势，我的观点是，这是因为大多数开发者更喜欢使用特别队列或流软件，如 Kafka 或 RabbitMQ。
