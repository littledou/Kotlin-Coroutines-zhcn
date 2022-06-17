# 构建协程作用域

**系列电子书：**[传送门](https://rikka-2.gitbook.io/kotlin-coroutine-deep-dive-zhcn/)

***

在前几章中，我们已经了解了构建作用域所需的工具。现在是时候总结这一知识，并看看它在实际项目中是如何使用的。我们将展示两个常见的例子，一个用于 Android，一个用于后端开发。

### CoroutineScope 工厂函数

`CoroutineScope` 是一个只包含 `coroutineContext` 属性的接口。

```kotlin
interface CoroutineScope {
    val coroutineContext: CoroutineContext
}
```

因此，我们可以让一个类实现这个接口，并直接使用它来启动协程。

```kotlin
class SomeClass : CoroutineScope {
    override val coroutineContext: CoroutineContext = Job()
    
    fun onStart() {
        launch {
            // ...
        }
    }
}
```

然而，这种方法并不是很流行，虽然它确实很方便，但在这种类中，我们可以直接调用 `CoroutineScope` 其它的方法，例如 `cancel`、 `ensureActive` ，这是有问题的。就算是无意为之，有人也很有可能会在别的地方不小心取消整个协程作用域，协程之后将不再启动。相反，我们通常更喜欢将协程作用域作为一个属性对象，并使用它来启动协程。

```kotlin
class SomeClass {
    val scope: CoroutineScope = ...
    
    fun onStart() {
        scope.launch {
            // ...
        }
    }
}
```

创建协程范围对象最简单的方法是使用 `CoroutineScope` 工厂函数，它用传入的上下文来创建一个作用域（如果上下文中没有 `Job`，则会额外创建一个用于结构化并发的 `Job`）

```kotlin
public fun CoroutineScope(
    context: CoroutineContext
): CoroutineScope = ContextScope(
    if (context[Job] != null) context
    else context + Job()
)
    
internal class ContextScope(
    context: CoroutineContext
) : CoroutineScope {
    override val coroutineContext: CoroutineContext = context
    
    override fun toString(): String =
        "CoroutineScope(coroutineContext=$coroutineContext)"
}
```

### 在 Android 上创建一个作用域

在大多数 Android 应用程序中，我们使用的架构基本都是 MVC 的后代：目前主要是 MVVM 或 MVP，我们将逻辑层提取到名为 ViewModel 或者 Presenter 的对象中。这里通常是协程启动的场所。在其它层中，例如在用例层或者仓库层，我们通常只创建挂起函数。协程可以在 Fragments 或 Activities 中启动。无论协程在 Android 中哪个地方启动，它们的构造方式很可能都是相同的。数据拉取需要在一个协程中操作，而协程需要在某个作用域中调用。我们一般会在 `BaseViewModel` 中构造一个作用域，因此它会为所有的 viewmodel 定义一次。在 `MainViewModel` 中，我们可以使用 `BaseViewModel` 所提供的 `scope` 属性。

```kotlin
abstract class BaseViewModel : ViewModel() {
    protected val scope = CoroutineScope(TODO())
}

class MainViewModel(
    private val userRepo: UserRepository,
    private val newsRepo: NewsRepository,
) : BaseViewModel {
    
    fun onCreate() {
        scope.launch {
            val user = userRepo.getUser()
            view.showUserData(user)
        }
        scope.launch {
            val news = newsRepo.getNews()
                .sortedByDescending { it.date }
            view.showNews(news)
        }
    }
}
```

现在是时候为这个作用域定义上下文了。考虑到 Android 中的许多函数需要在主线程调用，`Dispatchers.Main` 被认为是最好的选择，作为默认的调度器，我们将把它作为 Android 上默认上下文的一部分。

其次，我们需要使我们的作用域可以取消。当用户退出页面并调用 `onDestroy` （或者在 ViewModel 中调用 `onClear` ）时，取消所有未完成的任务是一个常见的做法：

```kotlin
abstract class BaseViewModel : ViewModel() {
    protected val scope = CoroutineScope(Dispatchers.Main + Job())
    
    override fun onCleared() {
        scope.cancel()
    }
}
```

更好的做法是：不取消整个作用域，而只取消其子作用域。因此，只要这个 viewmodel 是活动的，新的协程就可以在它的作用域内启动。

```kotlin
abstract class BaseViewModel : ViewModel() {
    protected val scope =
        CoroutineScope(Dispatchers.Main + Job())
    
    override fun onCleared() {
        scope.coroutineContext.cancelChildren()
    }
}
```

我们还希望这个作用域内启动的不同协程是互相独立的。当我们使用 `Job` 时，如果任何子协程因为异常而被取消，父协程以及其它所有子协程也会被取消。即使在加载用户数据时出现了异常，也不应该阻止我们看到新闻列表。要有这样的独立性，我们应该使用 `SupervisorJob`，而非 `Job`:

```kotlin
abstract class BaseViewModel : ViewModel() {
    protected val scope =
        CoroutineScope(Dispatchers.Main + SupervisorJob())
    
    override fun onCleared() {
        scope.coroutineContext.cancelChildren()
    }
}
```

最后一个重要的功能是处理异常的默认方法。在 Android 上，我们经常定义在出现不同类型的异常情况下应该做些什么。如果我们从 Http 调用中收到 401 Unauthorized，我们可能会拉起登录界面，如果是 503 Service Unavailable，我们可能会显示服务器异常消息。在其他情况下，我们可能会显示对话框、Toast 或者 snackbars。我们通常只定义一次这些异常的处理程序，例如在 BaseActivity 中写下异常处理方法，然后将它们传递给 viewmodel（通过构造函数），如果有未处理的异常，我们可以使用 `CoroutineExpcetionHandler` 来调用这些处理函数。

```kotlin
abstract class BaseViewModel(
    private val onError: (Throwable) -> Unit
) : ViewModel() {

    private val exceptionHandler =
        CoroutineExceptionHandler { _, throwable ->
            onError(throwable)
        }

    private val context =
        Dispatchers.Main + SupervisorJob() + exceptionHandler
    
    protected val scope = CoroutineScope(context)
    
    override fun onCleared() {
        context.cancelChildren()
    }
} 
```

另一种方法是将异常作为 `LiveData` 属性保存，这可以在 `BaseActivity` 或别的视图元素中观察。

```kotlin
abstract class BaseViewModel : ViewModel() {
    private val _failure: MutableLiveData<Throwable> =
        MutableLiveData()
    
    val failure: LiveData<Throwable> = _failure
    
    private val exceptionHandler =
        CoroutineExceptionHandler { _, throwable ->
            _failure.value = throwable
        }

    private val context =
        Dispatchers.Main + SupervisorJob() + exceptionHandler
    
    protected val scope = CoroutineScope(context)
    
    override fun onCleared() {
        context.cancelChildren()
    }
}
```

### viewModelScope 和 lfecycleScope

现在的 Android 应用程序中，除了定义自己的作用域，你还可以使用 `viewModelScope`（需要 androidx.lifecycle:lifecycle-viewmodel-ktx 版本2.2.0或更高）或 `lifecycleScope`（需要 androidx.lifecycle:lifecycle-runtime-ktx 版本2.2.0或更高）。它们的工作方式与我们刚才构造的几乎相同：它们使用 `Dispatcher.Main` 和 `SupervisorJob`。当 viewmodel 或者生命周期所有者被销毁时，它们会取消 job。

```kotlin
// 实现来自 lifecycle-viewmodel-ktx 2.4.0 
public val ViewModel.viewModelScope: CoroutineScope
    get() {
        val scope: CoroutineScope? = this.getTag(JOB_KEY)
        
        if (scope != null) {
            return scope
        }
        
        return setTagIfAbsent(
            JOB_KEY,
            CloseableCoroutineScope(
                SupervisorJob() +
                Dispatchers.Main.immediate
            )
        )
}

internal class CloseableCoroutineScope(
    context: CoroutineContext
) : Closeable, CoroutineScope {
    override val coroutineContext: CoroutineContext = context
    
    override fun close() {
        coroutineContext.cancel()
    }
}
```

如果我们不需要任何特殊的上下文作为作用域的一部分（如 `CoroutineExceptionHandler`），则建议使用 `viewModelScope` 和 `lifecycleScope`。 这就是为什么许多（也许是大多数） Android 应用程序选择这种方法的原因。

```kotlin
class ArticlesListViewModel(
    private val produceArticles: ProduceArticlesUseCase,
) : ViewModel() {
    private val _progressBarVisible =
        MutableStateFlow(false)
    
    val progressBarVisible: StateFlow<Boolean> =
        _progressBarVisible
    
    private val _articlesListState =
        MutableStateFlow<ArticlesListState>(Initial)
    
    val articlesListState: StateFlow<ArticlesListState> =
        _articlesListState
    
    fun onCreate() {
        viewModelScope.launch {
            _progressBarVisible.value = true
            val articles = produceArticles.produce()
            _articlesListState.value =
                ArticlesLoaded(articles)
            _progressBarVisible.value = false
        }
    }
}
```

### 在后端构造协程

许多后端框架内置了对挂起函数的支持。例如 Spring Boot 允许 controller 函数挂起。在 Ktor 中，默认情况下所有处理程序都是挂起函数。正因为如此，我们很少需要自己创建作用域。然而，假设我们这样做了（可能是因为我们需要启动一个任务或使用较老版本的 Spring），我们最可能需要的是：

* 一个带有线程池的自定义 dispatcher（或 `Dispatchers.Default`）
* `SuperviseJob` 让不同的协程相互独立
* 使用 `CoroutineExceptionHandler` 来处理异常，例如发送程序终止讯息，或记录问题

```kotlin
@Configuration
public class CoroutineScopeConfiguration {
    @Bean(name = "coroutineDispatcher")
    fun coroutineDispatcher(): CoroutineDispatcher =
        Dispatchers.IO.limitedParallelism(5)
    
    @Bean(name = "coroutineExceptionHandler")
    fun coroutineExceptionHandler() =
        CoroutineExceptionHandler { _, throwable ->
            FirebaseCrashlytics.getInstance()
                .recordException(throwable)
            }
    @Bean
    fun coroutineScope(
        coroutineDispatcher: CoroutineDispatcher,
        coroutineExceptionHandler: CoroutineExceptionHandler,
    ) = CoroutineScope(
        SupervisorJob() +
            coroutineDispatcher +
            coroutineExceptionHandler
    )
}
```

这样的作用域通常通过构造函数注入到类中。正因如此，作用域可以只定义一次，就能在许多类上使用。并且可以很容易地用不同的作用域替换它，以便测试。

### 构造用于额外调用的作用域

如_协程作用域函数_一章所描述的那样，我们经常需要一些额外的任务创建协程作用域。这些作用域通常通过函数或构造函数注入。如果我们只计划用这些作用域来调用一些挂起函数，那么它们只要有一个 `SupervisorScope` 就足够了。

```kotlin
val analyticsScope = CoroutineScope(SupervisorJob())
```

它们所有的异常只会显示在日志中，如果你想将日志发送到监控系统，请使用 `CoroutineExceptionHandler`。

```kotlin
private val exceptionHandler =
    CoroutineExceptionHandler { _, throwable ->
        FirebaseCrashlytics.getInstance()
            .recordException(throwable)
    }

val analyticsScope = CoroutineScope(
    SupervisorJob() + exceptionHandler
)
```

另一个常见的做法是设置不同的调度器。例如，如果你可能会在这个作用域上有阻塞调用，就是用 `Dispatchers.IO`，或者如果你想修改 Android 上的主视图，就使用 `Dispatchers.Main`（如果我们设置 `Dispatcher.Main`，在 Android 上测试会更加容易）。

```kotlin
val analyticsScope = CoroutineScope(
    SupervisorJob() + Dispatchers.IO
)
```

### 总结

我希望在本章之后，你将知道如何在大多数典型情况下构建作用域。当在实际项目中使用协程时，这些点很重要。对于许多小型和简单应用程序来说，这已经足够了，但是对于那些更正经的应用，我们仍然需要涵盖另外两个主题：适当的数据同步和测试。
