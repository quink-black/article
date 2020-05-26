# Kotlin协程设计

## 译者前言

吾师从《APUE》、《TCP/IP Illustrated》之W.Richard Stevens, 乐读spec之原文，辨细节之是非。今于稚子酣睡之际，译《Kotlin Coroutines》一文。予少有译文，其因有四：

* 英文半通，吾之失也
* 汉语退化，国之失也
* 费我光阴，事小
* 误人子弟，事大

然见诸博客，管中窥豹者多，以讹传讹者众。译文虽亦属二手之资料，尚有原文可溯源，谬误亦可纠也。



原文：https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md



## 摘要

本文描述Kotlin的协程特性。协程也称之为（或包含）：

* generators/yield
* async/await
* composable/delimited continuations

Kotlin协程特性的目标：

* 不依赖第三方库的特定实现，比如Java的Future
* 覆盖async/await和generator blocks的用例需求
* 可以用Kotlin的协程封装现有的异步API（诸如Java NIO，Future等）



## 用例

你可以把协程当成是“可挂起的计算单元”，可以在某个点挂起，之后再恢复执行，可以在当前线程恢复，也可以在其他线程恢复。协程之间的互相调用，构成了__协作多任务__机。（译者：非实时操作系统/非实时调度，进程、线程通常是抢占的，而kotlin的协程之间是协作方式，即如果某个suspend fun是计算密集型任务，或者是阻塞的IO操作，则携程不会挂起，同一个线程的其他suspend fun得不到执行）

### 异步计算

协程的首要应用场景是异步计算（其他编程语言如C#，用async/await实现相同机制）。首先看基于callback的代码实现：

```kotlin
// asynchronously read into `buf`, and when done run the lambda
inChannel.read(buf) {
    // this lambda is executed when the reading completes
    // 第一个callback
    bytesRead ->
    ...
    ...
    process(buf, bytesRead)
    
    // asynchronously write from `buf`, and when done run the lambda
    outChannel.write(buf) {
        // this lambda is executed when the writing completes
        // 第二个callback
        ...
        ...
        outFile.close()          
    }
}
```
注意这里有两个嵌套的callback. 每增加一层callback嵌套，增加一级代码缩进。此场景称之为回调地狱（callback hell)，常见于JavaScript.

改用协程来实现相同逻辑：
```kotlin
launch {
    // suspend while asynchronously reading
    val bytesRead = inChannel.aRead(buf) 
    // we only get to this line when reading completes
    ...
    ...
    process(buf, bytesRead)
    // suspend while asynchronously writing   
    outChannel.aWrite(buf)
    // we only get to this line when writing completes  
    ...
    ...
    outFile.close()
}
```
aRead()和aWrite()是两个可挂起函数。它们可以挂起执行，让出当前线程，当完成读写之后，再恢复执行。基于协程的实现和基于callback的实现，两者功能上是等效的，前者实现可读性更佳。

Kotlin协程设计的一个原则是普适性。上述代码，launch、aRead、aWrite都是库函数：launch是协程builder，构建和启动协程，aRead和aWrite是特定实现的挂起函数。挂起函数有一个隐含的参数：continuation. continuation实质是泛化的callback.

考虑上述代码段循环执行的场景，基于协程的实现方式也更简单

```kotlin
launch {
    while (true) {
        // suspend while asynchronously reading
        val bytesRead = inFile.aRead(buf)
        // continue when the reading is done
        if (bytesRead == -1) break
        ...
        process(buf, bytesRead)
        // suspend while asynchronously writing
        outFile.aWrite(buf) 
        // continue when the writing is done
        ...
    }
}
```

同样，协程方式处理异常也更容易。



### Future

除callback外，另一种表达异步计算的方式是Future(或者叫promise、deferred)，例如：

```kotlin
val future = runAfterBoth(
    loadImageAsync("...original..."), // creates a Future 
    loadImageAsync("...overlay...")   // creates a Future
) {
    original, overlay ->
    ...
    applyOverlay(original, overlay)
}
```

改用协程实现

```kotlin
val future = future {
    val original = loadImageAsync("...original...") // creates a Future
    val overlay = loadImageAsync("...overlay...")   // creates a Future
    ...
    // suspend while awaiting the loading of the images
    // then run `applyOverlay(...)` when they are both loaded
    applyOverlay(original.await(), overlay.await())
}
```

同样，基于协程的实现代码缩进更少，构造逻辑更自然，也不需要特殊的关键字来支持future：kotlin的future{}和await()都是库函数，不是kotlin语言的关键字。而C# JS等用async和await来支持future.

### Generators

另一个用例是懒计算序列（lazily computed sequences)，C#、python等语言用yield关键字实现。看起来是执行串行的代码，实际运行时是按需计算：

```kotlin
// inferred type is Sequence<Int>
val fibonacci = sequence {
    yield(1) // first Fibonacci number
    var cur = 1
    var next = 1
    while (true) {
        yield(next) // next Fibonacci number
        val tmp = cur + next
        cur = next
        next = tmp
    }
}
```

上述代码创建的是斐波那契数列的懒序列，可以无限执行。我们可以取出计算结果的一部分，通过`take()`函数:

```kotlin
println(fibonacci.take(10).joinToString())
```

输出结果：

```
1, 1, 2, 3, 5, 8, 13, 21, 34, 55
```

generator的优势在于支持任意控制流，例如while（就像上述代码），if，try/catch/finally

```kotlin
val seq = sequence {
    yield(firstItem) // suspension point

    for (item in input) {
        if (!item.isValid()) break // don't generate any more items
        val foo = item.toFoo()
        if (!foo.isGood()) continue
        yield(foo) // suspension point        
    }
    
    try {
        yield(lastItem()) // suspension point
    }
    finally {
        // some finalization code
    }
} 
```

### 异步UI

通常UI框架用单个线程处理UI操作的事件分发，不允许从其他线程修改UI状态。所有UI框架都提供一种机制，把要执行的代码块放到UI线程执行。例如，Swing有`SwingUtilities.invokeLater`, JavaFX有`Platform.runLater`, Android有`Activity.runOnUiThread`等等。Swing代码示例：

```
makeAsyncRequest {
    // this lambda is executed when the async request completes
    result, exception ->
    
    if (exception == null) {
        // display result in UI
        SwingUtilities.invokeLater {
            display(result)   
        }
    } else {
       // process exception
    }
}
```

和异步计算一样，这也是一种回调地狱。改用协程处理：

```kotlin
launch(Swing) {
    try {
        // suspend while asynchronously making request
        val result = makeRequest()
        // display result in UI, here Swing context ensures that we always stay in event dispatch thread
        display(result)
    } catch (exception: Throwable) {
        // process exception
    }
}
```

可见异常处理的形式也非常自然。

### 更多用例

协程还有其他使用场景，例如：

* 基于channel的并发（goroutines和channel）
* 基于actor的并发
* 偶尔需要用户交互的后台进程，例如展示模态对话框
* 通信协议：actor可以不用状态机，而是改用串行执行的方式实现
* web应用工作流：用户注册、邮箱验证、登录（挂起的协程可以序列化存储到数据库中）



## 协程概述

本章节介绍支撑协程的语言机制以及相关的标准库。

### 术语

* 协程：可挂起执行单元的一个实例。概念上和线程相似，同样是取一个代码块来执行，并且有生命周期——创建和启动。但要注意，协程不是绑定在特定线程上。协程可以在某个线程上挂起，再从另一个线程恢复执行。此外，和future以及promise一样，协程执行完后可以返回一个结果（可以是value也可以是exception)

* 可挂起函数：带suspend修饰符的函数。它可以挂起，然后让当前线程去执行其他挂起函数。挂起函数只能被其他挂起函数或挂起lambda调用。前文提到的await()和yield()就是挂起函数。kotlin标准库提供原挂起函数(primitive suspending function)，用来调用执行其他挂起函数。（译者注：挂起函数只能被挂起函数调用，所以一定有个源头，由标准库来实现的源头）

* 可挂起lambda：只能在协程里执行的一段代码块。它看起来和普通的lambda表达式一样，只是带了suspend修饰符。常规的lambda表达式是匿名局部函数的简短表达形式，类似的，可挂起lambda是匿名可挂起函数的简短表达形式。同样，可挂起lambda能够让出当前线程，去执行其他的挂起函数。前文代码示例，`launch`, `future`, `sequence`后面跟的花括号就是可挂起lambda。

* 协程builder：以可挂起lambda为参数，创建协程，返回执行结果。例如，`launch{}`, `future{}`, `sequence{}`都是协程builder。标准库提供原协程builder来实现其他协程builder.

* 挂起点（suspension point): 协程执行过程中可以挂起的地方。从语法上来说，调用挂起函数的地方就是挂起点，但实际的挂起发生在挂起函数调用系统原函数的地方。

* Continuation：被挂起的协程的状态，能够完整描述从挂起点恢复后如何执行的上下文。例如：

  ```kotlin
  sequence {
      for (i in 1..10) yield(i * i)
      println("over")
  }  
  ```

  每次调用yield时被挂起，yield之后的状态用continuation来表示。

如前文所述，Kotlin协程设计的一大目标是柔性：可以支持多种现存的异步API模式和用例，同时尽可能少往编译器添加新东西。Kotlin协程最终的实现方式，只需要编译器支持挂起函数、挂起lambda和挂起函数类型，其他的功能都以库函数的方式实现。

### Continuation接口

标准库的`Continuation`接口如下（见Kotlin.coroutines包），代表一个泛化的callback

```kotlin
interface Continuation<in T> {
   val context: CoroutineContext
   fun resumeWith(result: Result<T>)
}
```

关于`context`详见后文，用来表示任意用户自定义的、和协程关联的上下文。`resumeWith`是协程执行完调用的callback，可以返回成功执行的结果，也可以返回失败时的异常。

为了方便使用，提供了两个扩展函数，分别用来返回结果和异常：

```kotlin
fun <T> Continuation<T>.resume(value: T)
fun <T> Continuation<T>.resumeWithException(exception: Throwable)
```

### 挂起函数

挂起函数await()可以用如下方式实现：

```kotlin
suspend fun <T> CompletableFuture<T>.await(): T =
    suspendCoroutine<T> { cont: Continuation<T> ->
        whenComplete { result, exception ->
            if (exception == null) // the future has been completed normally
                cont.resume(result)
            else // the future has completed with an exception
                cont.resumeWithException(exception)
        }
    }
```

`suspend`修饰符代表此函数可以挂起协程。`await()`作为`CompletableFuture<T>`的扩展函数，提高了代码的可读性，比如如下代码，一目了然：

```kotlin
doSomethingAsync(...).await()
```

`suspend`修饰符可以用于多种函数类型：顶层函数、扩展函数、成员函数、局部函数以及操作符函数等等。

> `getter/setter`，构造函数，部分操作符函数不支持`suspend`修饰符，以后可能放宽此类限制。

挂起函数可以调用普通函数，但要真正能够被挂起，必须调用kotlin标准库提供的挂起函数，例如`await()`调用的Kotlin标准库函数`suspendCoroutine<T>`

```kotlin
suspend fun <T> suspendCoroutine(block: (Continuation<T>) -> Unit): T
```

`suspendCoroutine`被调用时，会把协程的执行状态包装成一个`Continuation<T>`，作为参数传给`block`. 要把协程从挂起点恢复执行，`block`调用`continuation.resumeWith()` (或者调用扩展函数 `continuation.resume()` 、 `continuation.resumeWithException()`)。可以在当前线程调用，也可以从其他线程调用。` suspendCoroutine`在运行时可能执行多次。如果` suspendCoroutine`返回前执行了`continuation.resumeWith()`，则认为没有发生挂起；反之如果

`continuation.resumeWith()`没执行，则认为发生了挂起。

传给`continuation.resumeWith()`的`result`，会成为`suspendCoroutine<T>`的返回值，最终成为`await()`的返回值。

`continuation.resumeWith()`不允许多次调用，多次调用抛异常`IllegalStateException`.

### 协程builder

普通函数不能调用挂起函数，由Kotlin标准库提供协程的创建方式。下面是协程builder `launch{}`的简单实现方式：

```kotlin
fun launch(context: CoroutineContext = EmptyCoroutineContext, block: suspend () -> Unit) =
    block.startCoroutine(Continuation(context) { result ->
        result.onFailure { exception ->
            val currentThread = Thread.currentThread()
            currentThread.uncaughtExceptionHandler.uncaughtException(currentThread, exception)
        }
    })
```

`Continuation(context){...}`函数返回一个`Continuation`接口对象。这个`Continuation`接口对象作为参数，传给了`block.startCoroutine(...)`

> 译者注：`Continuation(context){...}`函数的实现很简单，如下：

```kotlin
/**
 * Creates a [Continuation] instance with the given [context] and implementation of [resumeWith] method.
 */
@SinceKotlin("1.3")
@InlineOnly
public inline fun <T> Continuation(
    context: CoroutineContext,
    crossinline resumeWith: (Result<T>) -> Unit
): Continuation<T> =
    object : Continuation<T> {
        override val context: CoroutineContext
            get() = context

        override fun resumeWith(result: Result<T>) =
            resumeWith(result)
    }
```

协程执行完，通过resumeWith返回成功或失败的结果。因为`launch`创建的是“即发即弃”(fire-and-forget)类型的协程，所以挂起函数的返回值是Unit，`Continuation`通过`resumeWith`返回的结果被丢弃了。如果`resumeWith`接收的是一个异常，交由当前线程的未捕获异常handler处理。

> 这里的`launch`实现是个简单示例，实际的实现更复杂，会返回一个可以cancel的`Job`对象。

`startCoroutine`是标准库里的扩展函数，创建协程并在当前线程立即执行，直到第一个挂起点，然后返回。挂起函数指定了挂起点、协程恢复执行的时机、恢复执行的方式。

> continuation拦截器（来自context)可以把协程分发到其他线程执行。

### 协程上下文

协程上下文是用户自定义的绑定到协程的对象。可以用来处理协程的线程策略，协程执行的记录、安全性和事务处理，协程的辨识和命名等。可以用如下的类比方式来理解协程和协程上下文的关系。把协程当成轻量的线程，协程上下文即线程局部变量的集合。但要注意，线程局部变量是可修改的，协程上下文是常量。因为协程非常轻量，当遇到要修改协程上下文的场景时，只需要重新发起一个新的协程。

标准库没有限定协程上下文必须包含什么。标准库定义了一些接口和抽象类，你可以通过自由复合的方式定义协程上下文。来自不同第三方库的协程上下文可以同时存在，不会造成冲突。

概念上，协程上下文是一堆可索引的Element的集合，每个Element有唯一的key。它是set和map的混合体。它的Element和map一样，有key值，但它的key值和Element直接关联，更像是set。

```kotlin
interface CoroutineContext {
    operator fun <E : Element> get(key: Key<E>): E?
    fun <R> fold(initial: R, operation: (R, Element) -> R): R
    operator fun plus(context: CoroutineContext): CoroutineContext
    fun minusKey(key: Key<*>): CoroutineContext

    interface Element : CoroutineContext {
        val key: Key<*>
    }

    interface Key<E : Element>
}
```

`CoroutineContext`有4个核心的操作

1. `get`提供了类型安全的通过key取出Element的操作。使用时可以用`[]`符号来调用
2. `fold`和`Collection.fold`类似，可以用来遍历所有的`Element`
3. `plus`类似`Set.plus`，返回两个上下文的叠加；如果`plus`左右两端存在相同的key，对应的`Element`取右侧的
4. `minusKey`返回不包含指定key值的上下文

`Element`本身的类型也是协程上下文。它的`Element`只有一个，即它自身。如果一个库定义了`auth` Element，另一个库定义了`threadPool` 协程上下文，你可以这样组合使用:

```
launch(auth + threadPool) { ... }
```

> 译者注：
>
> ```kotlin
>     /**
>      * An element of the [CoroutineContext]. An element of the coroutine context is a singleton context by itself.
>      */
>     public interface Element : CoroutineContext {
>         /**
>          * A key of this coroutine context element.
>          */
>         public val key: Key<*>
> 
>         public override operator fun <E : Element> get(key: Key<E>): E? =
>             @Suppress("UNCHECKED_CAST")
>             if (this.key == key) this as E else null
> 
>         public override fun <R> fold(initial: R, operation: (R, Element) -> R): R =
>             operation(initial, this)
> 
>         public override fun minusKey(key: Key<*>): CoroutineContext =
>             if (this.key == key) EmptyCoroutineContext else this
>     }
> ```

> kotlinx.coroutines提供了多个上下文`Element`，包括`Dispatchers.Default`，用来把协程分发到一个共享的线程池。

标准库提供了`EmptyCoroutineContext`，`Element`为空的`CoroutineContext`实例。

第三方库的`Element`应当继承自标准库的`AbstractCoroutineContextElement`.推荐用如下形式：

```kotlin
class AuthUser(val name: String) : AbstractCoroutineContextElement(AuthUser) {
    companion object Key : CoroutineContext.Key<AuthUser>
}
```

注意`AuthUser Key`的实现方式，在从context查找Element时十分便捷。使用`AuthUser`的示例代码：

```kotlin
suspend fun doSomething() {
    val currentUser = coroutineContext[AuthUser]?.name ?: throw SecurityException("unauthorized")
    // do something user-specific
}
```

`coroutineContext`是一个顶层property，用来提取当前协程的上下文。

### Continuation拦截器

让我们再回顾下异步UI的用例。异步UI要求协程自身必须在UI线程执行，尽管协程内的挂起函数可以在其他线程恢复。这一要求由协程拦截器来保证。首先，我们要理解协程的生命周期。考虑如下使用`launch{}`协程builder的代码段：

```kotlin
launch(Swing) {
    initialCode() // execution of initial code
    f1.await() // suspension point #1
    block1() // execution #1
    f2.await() // suspension point #2
    block2() // execution #2
}
```

协程从`initialCode`开始执行，直到第一个挂起点。挂起一定时间后，协程恢复执行`block1`，然后再次挂起和恢复执行`block2`，之后结束。

协程拦截器可以拦截和封装`initialCode`,`block1`, `block2`从恢复点到下个挂起点的continuation.协程的开头可以认为是从初始continuation的恢复点。标准库提供了` ContinuationInterceptor`接口：

```kotlin
interface ContinuationInterceptor : CoroutineContext.Element {
    companion object Key : CoroutineContext.Key<ContinuationInterceptor>
    fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T>
    fun releaseInterceptedContinuation(continuation: Continuation<*>)
}
```

`interceptContinuation`函数负责包装协程的continuation. 当协程挂起时，协程框架用如下代码包装实际的continuation，用于后续的恢复执行

```kotlin
val intercepted = continuation.context[ContinuationInterceptor]?.interceptContinuation(continuation) ?: continuation
```

协程框架缓存对应`continuation` 的`intercepted` continuation，并在适当时机通过`releaseInterceptedContinuation(intercepted)`来释放。细节详见后文。

来看一个`Swing` 拦截器把代码段抛到Swing UI事件分发线程的例子。先看`SwingContinuation`包装类，通过`SwingUtilities.invokeLater`把实际的`cont`分发到UI线程：

```kotlin
private class SwingContinuation<T>(val cont: Continuation<T>) : Continuation<T> {
    override val context: CoroutineContext = cont.context

    override fun resumeWith(result: Result<T>) {
        SwingUtilities.invokeLater { cont.resumeWith(result) }
    }
}
```

再看Swing对象，它是协程上下文的Element，同时实现了`ContinuationInterceptor`接口：

```kotlin
object Swing : AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor {
    override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =
        SwingContinuation(continuation)
}
```

接下来就可以把`Swing`作为参数传给协程builder`launch{}`，让协程在Swing UI事件分发线程执行：

```kotlin
launch(Swing) {
  // 这里的代码可能挂起，但一定是在Swing UI线程恢复执行
}
```

### 有限制的挂起

要实现`sequence{}`和`yield()`以满足generator的用例需求，需要另一类协程builder和挂起函数。实现`sequence{}`协程builder的示例代码：

```kotlin
fun <T> sequence(block: suspend SequenceScope<T>.() -> Unit): Sequence<T> = Sequence {
    SequenceCoroutine<T>().apply {
        nextStep = block.createCoroutine(receiver = this, completion = this)
    }
}
```

`sequence`用了用了标准库的另一个原函数`createCoroutine`. `createCoroutine`和`startCoroutine`很相似，不同之处在于前者只创建协程，但不启动协程。它返回的是初始的*continuation*:

```kotlin
fun <T> (suspend () -> T).createCoroutine(completion: Continuation<T>): Continuation<Unit>
fun <R, T> (suspend R.() -> T).createCoroutine(receiver: R, completion: Continuation<T>): Continuation<Unit>
```

另一个区别是，挂起lambda `block`是以`SequenceScope<T>`为receiver的扩展lambda. `SequenceScope`接口提供了generator代码块的`scope`:

```
interface SequenceScope<in T> {
    suspend fun yield(value: T)
}
```

为了避免创建多个对象，`SequenceCoroutine<T>`类同时实现了`SequenceScope<T>`接口和`Continuation<Unit>`接口，所以可同时作为`createCoroutine`的receiver和`completion`参数。简单实现如下：

```kotlin
private class SequenceCoroutine<T>: AbstractIterator<T>(), SequenceScope<T>, Continuation<Unit> {
    lateinit var nextStep: Continuation<Unit>

    // AbstractIterator implementation
    override fun computeNext() { nextStep.resume(Unit) }

    // Completion continuation implementation
    override val context: CoroutineContext get() = EmptyCoroutineContext

    override fun resumeWith(result: Result<Unit>) {
        result.getOrThrow() // bail out on error
        done()
    }

    // Generator implementation
    override suspend fun yield(value: T) {
        setNext(value)
        return suspendCoroutine { cont -> nextStep = cont }
    }
}
```

`yield`的实现用了`suspendCoroutine`挂起函数来挂起协程，捕获它的continuation. Continuation存放在nextStep变量中，computeNext()被调用时执行`nextStep.resume()`.

但要注意，`sequence{}`和`yield()`的工作方式是同步的，并非任意挂起函数可以捕获它们的continuation。continuation的捕获方式、存放方式和恢复时机是受到严格控制的，构成了有限制的挂起域。通过在scope的类或接口添加`@RestrictsSuspension`注解可以限制挂起方式，例如：

```kotlin
@RestrictsSuspension
interface SequenceScope<in T> {
    suspend fun yield(value: T)
}
```

此注解为`sequence{}`或类似的同步协程builder内的挂起函数添加了一些限制条件。以`@RestrictsSuspension` scope为receiver的扩展函数或扩展lambda称为有限制的挂起函数。有限制的挂起函数只能调用同一个有限制scope内的其他挂起函数。

特别要提及的是，`SequenceScope`的扩展函数不能调用`suspendContinuation`或其他通用挂起函数。想要挂起`sequence`协程的执行，必须调用`SequenceScope.yield`. `yield`自身是`SequenceScope`的成员函数，不受前述限制条件影响（只有扩展函数和扩展lambda受限制）。

像`sequnce`这类有限制的协程builder，支持任意的协程上下文是没意义的，scope类或接口（例如`SequenceScope`）就可以看成是协程的上下文。因此，有限制的协程必须使用`EmptyCoroutineContext`作为它们的上下文，`SequenceScope.context`返回的就是`EmptyCoroutineContext`. 用其他context创建有限制的协程，会抛出`IllegalArgumentException`异常。



## 协程实施细节

本章节简要介绍协程实施的细节。这些细节藏在前文所提及的基础构件中。只要不影响API和ABI的兼容，实施细节可能随着时间而改变。

### Continuation passing style

挂起函数是通过Continuation-Passing-Stype (CPS)实现的。每个挂起函数和挂起lambda都有一个隐含的类型为Continuation的参数。例如，`await`的函数原型是这样的：

```kotlin
suspend fun <T> CompletableFuture<T>.await(): T
```

实际上，经过CPS变换之后的函数原型是这样的：

```kotlin
fun <T> CompletableFuture<T>.await(continuation: Continuation<T>): Any?
```

注意`Continuation`的类型参数也是`T`. 返回值类型`Any?`用来反映挂起函数的行动。当挂起函数挂起协程时，返回值是一个特殊标记值`COROUTINE_SUSPENDED`. 当挂起函数没有挂起，继续执行时，返回处理结果或异常。也就是说，`await`的返回值是`COROUTINE_SUSPENDED`和`T`的联合(union)，只能用`Any?`类型来表示。

实际的实现方式，挂起函数不允许直接在栈上调用continuation，因为如果协程太深容易导致栈溢出。标准库众的`suspendCoroutine`负责处理这些细节问题，追踪continuation的调用执行，并且保证不管以何种方式在何时调用continuation，仍然满足前文所述的设计方式。

### 状态机

用尽可能少的对象和类高效的实现协程是至关重要的。许多编程语言是用状态机的形式来实现协程，Kotlin也采用此方案。每个挂起lambda只编译生成一个class，不论内部有多少个挂起点。

核心思想：每个挂起函数编译生成一个状态机，每个挂起点对应一个状态。例如下面代码有两个挂起点：

```kotlin
val a = a()
val y = foo(a).await() // 挂起点 #1
b()
val z = bar(a, y).await() // 挂起点 #2
c(z)
```

上述代码有三个状态：

* 初始状态（所有挂起点之前）
* 第一个挂起点之后
* 第二个挂起点之后

每个状态对应一个continuation。

上述代码编译成一个状态机匿名类。此匿名类用一个成员变量来记录当前状态，协程的局部变量也变成了匿名类的成员变量，目的是能够在多个状态中共享访问这些局部变量。下面是一段伪代码，描述编译后的挂起函数

```kotlin
class <anonymous_for_state_machine> extends SuspendLambda<...> {
    // 状态机的当前状态
    int label = 0
    
    // 协程局部变量
    A a = null
    Y y = null
    
    void resumeWith(Object result) {
        if (label == 0) goto L0
        if (label == 1) goto L1
        if (label == 2) goto L2
        else throw IllegalStateException()
        
      L0:
        // result is expected to be `null` at this invocation
        a = a()
        label = 1
        result = foo(a).await(this) // 'this' is passed as a continuation 
        if (result == COROUTINE_SUSPENDED) return // return if await had suspended execution
      L1:
        // external code has resumed this coroutine passing the result of .await() 
        y = (Y) result
        b()
        label = 2
        result = bar(a, y).await(this) // 'this' is passed as a continuation
        if (result == COROUTINE_SUSPENDED) return // return if await had suspended execution
      L2:
        // external code has resumed this coroutine passing the result of .await()
        Z z = (Z) result
        c(z)
        label = -1 // No more steps are allowed
        return
    }          
}    
```

伪代码描述的是编译后字节码的逻辑，所以用了goto和label.



