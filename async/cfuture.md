# CompletableFuture

Java8在推出了lambda之后，也伴随推出了支持lambda的函式库。其中我们把[Stream API](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html), [Optional API](https://docs.oracle.com/javase/8/docs/api/java/util/Optional.html), 跟[CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html)称为Java8三神器。这三個都有Functional Language里[Monad](https://en.wikipedia.org/wiki/Monad_(functional_programming))的精神，而CompletableFuture也就是Monadic Future。这边我还是先不讨论太多Functional Language，让我们来直接看看CompletableFuture怎么使用。

先來看一個简单的例子

```java
CompletableFuture<Void> future =
CompletableFuture.runAsync(() -> {
    try {
        Thread.sleep(1000);
        System.out.println("hello");
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
});
future.get();
System.out.println("world");
```

输出：

```shell
hello
world
```



通过lambda的特性，在CompletableFuture中已定义几个static methods，可以帮助我们快速创建

| Method                                                       | Description                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [runAsync(Runnable runnable)](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#runAsync-java.lang.Runnable-) | 非同步的执行一个沒有返回值的task，并且在缺省的thread pool中执行。缺省为： [ForkJoinPool.commonPool()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html#commonPool--) |
| [runAsync(Runnable runnable, Executor executor)](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#runAsync-java.lang.Runnable-java.util.concurrent.Executor-) | 非同步的执行一個沒有返回值的task，并且在指定的thread pool之中执行。 |
| [supplyAsync(Suppliersupplier)](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#supplyAsync-java.util.function.Supplier-) | 非同步的执行一個有返回值的task，并且在缺省的thread pool之中执行。 |
| [supplyAsync(Suppliersupplier, Executor executor)](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#supplyAsync-java.util.function.Supplier-java.util.concurrent.Executor-) | 非同步的执行一個有返回值的task，并且在指定的thread pool之中执行。 |



Completable是什么意思? 事实上CompletableFuture是一个Future的实现，至于Completable，以下四个特性来讨论

1. Completable
2. Listenable
3. Composible
4. Combinable

## Completable

所谓的Completable就是这个future可以被complete。其实这要先讨论Future跟Promise这两个概念。

1. Future: 是一个未来会完成的一个结果，算是这个结果的容器。Caller(调用方)通过Future来等非同步执行的结果。
2. Promise: 是可以被改变可以被完成的值，通常是非同步执行的结果。Callee(被调用方)通过Promise来告知非同步完成的结果。

基本上就是一体两面。对于asynchronous invocation(异步调用)，对于caller看到就是future，对于callee就是看到promise。而CompletableFuture就同时扮演了Future跟Promise两种角色。

所以CompletableFuture会被下面这样使用：

1. 在非同步调用时，会先创建一个CompletableFuture，并且返回给Caller(调用方)
2. 这个CompletableFuture会连童async task一起传到worker thread中。
3. 当执行完这个async task，Callee(被调用方)会呼叫CompletableFuture的`complete()`
4. 此时Caller(调用方)可以通过CompletableFuture的`get()`获取結果的值。

其实这个跟Thread的`wait()`/`notify()`相似，不一样的就是这不只是流程同步，还带有返回值。除了complete以外，当执行错误的时候，也可以调用`completeExceptionally()`。

在completable这个特性里，我们把关于caller/consumer用的Future介面，以及callee/provider用的Completable放在一起，我們来看看一下有哪些跟Completable相关：

| Method                                                       | Description                         |
| ------------------------------------------------------------ | ----------------------------------- |
| [complete(T t)](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#complete-T-) | 完成非同步执行，并且返回结果        |
| [completeExceptionally(Throwable ex)](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#completeExceptionally-java.lang.Throwable-) | 非同步执行不正常的結束，抛throwable |

有了以上的概念，我们很快的可以很快地写出`CompletableFuture.runAsync()`可能的逻辑

```java
public static CompletableFuture<Void> runAsync(Runnable runnable) {
    CompletableFuture<Void> future = new CompletableFuture<>();
    ForkJoinPool.commonPool().execute(() -> {
        try {
            runnable.run();
            future.complete(null);
        } catch (Throwable throwable) {
            future.completeExceptionally(throwable);
        }
    });
    return future;
}
```

在Google的[Guava library](https://github.com/google/guava)中也可以看到completable的踪影，那就是[SettableFuture](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/SettableFuture.html)。

## Listenable

对于asynchronous invocation(异步调用)的caller來讲，`Future`只提供了一个pulling result的方法，更多时候我们想要的是**好了叫我**这种语义。因此*Listenable*的特性，就是我们可以注册一個callback，让我可以listen执行完成的event。

在CompletableFuture主要是通过[whenComplete()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#whenComplete-java.util.function.BiConsumer-)跟[handle()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#handle-java.util.function.BiFunction-)這兩個method。

| Method                                                       | Description                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [whenComplete()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#whenComplete-java.util.function.BiConsumer-) | 当完成时，把result或exception带到callback function中。       |
| [handle()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#handle-java.util.function.BiFunction-) | 当完成时，把result或exception带到callback function中，并且返回最后的结果。 |

我再把最上面的例子改写成用listener的方式

```java
CompletableFuture.runAsync(() -> {
    try {
        Thread.sleep(1000);
        System.out.println("hello");
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}).whenComplete((result, throwable) -> {
    System.out.println("world");
});
```

这两個method以及包含后面会提到的method都有三种变形，分別是：

- xxxx(function): function会用前个执行的thread去调用。
- xxxxAsync(function): function会用非同步的方式调用，使用缺省的thread pool。
- xxxxAsync(function, executor): function会用非同步的方式调用，使用指定的thread pool。

由于基本逻辑相似，之后就不再赘述。

同样在Guava library中也可以看到listenable的踪影，那就是[ListenableFuture](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/ListenableFuture.html)。

## Composible

有了Listenable的特性之后，我们就可以做到当完成时，在做下一件事情。如果接下來又是一个非同步的工作，那就可能会串成非常多层，我们称之为callback hell(回调地狱)。如下例：

```java
public static void sleep(long time) {
    try {
        System.out.printf("sleep for %d milli\n", time);
        Thread.sleep(time);
        System.out.printf("wake up\n");
    } catch (InterruptedException e) {
    }
}

public static void main(String[] args) throws InterruptedException {
    CompletableFuture<Void> future = 
    CompletableFuture
    .runAsync(() -> sleep(1000))
    .whenComplete((result, throwable) -> {
        if (throwable != null) {
            return;
        }

        CompletableFuture
        .runAsync(() -> sleep(1000))
        .whenComplete((result2, throwable2) -> {
            if (throwable2 != null) {
                return;
            }

            CompletableFuture
            .runAsync(() -> sleep(1000))
            .whenComplete((result3, throwable3) -> {
                if (throwable2 != null) {
                    return;
                }
                System.out.println("Done");
            });
        });
    });
```

这个程序这样三层可能已经受不了了，如果更多层应该会有恶心到的感觉。这还不打紧，如果再加上异常处理，那可能更是晕头转向。

对于这种一连串的invocation(调用)，如果可以把这些async function组起來，变成单个的future，可能会舒服许多。先来看最后的结果，我们再来讨论细节。

```java
CompletableFuture
.runAsync(() -> sleep(1000))
.thenRunAsync(() -> sleep(1000))
.thenRunAsync(() -> sleep(1000))
.whenComplete((r, ex) -> System.out.println("done"));
```

有沒有觉得清爽许多?这就是Composible(可组合的)的魅力。

在CompletableFuture中，它提供了非常多的compose methods来帮助我们组合各种sync methods变成async methods。我来列举一下：

| Method                                                       | Trasnformer                         | To Type                   |
| ------------------------------------------------------------ | ----------------------------------- | ------------------------- |
| [thenRun()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#thenRun-java.lang.Runnable-) | `Runnable`                          | `CompletableFuture<Void>` |
| [thenAccept()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#thenAccept-java.util.function.Consumer-) | `Consumer<T>`                       | `CompletableFuture<Void>` |
| [thenApply()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#thenApply-java.util.function.Function-) | `Function<T, U>`                    | `CompletableFuture<U>`    |
| [thenCompose()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#thenCompose-java.util.function.Function-) | `Function<T, CompletableFuture<U>>` | `CompletableFuture<U>`    |

泛型的部分我有稍微调整一下，让它比较容易读。但是我们都可以看到他们都有一个特性，就是把原本某个CompletableFuture的type parameter，经过一个transformer后，转成另外一个Type的CompletableFuture，这就是Monad中的**map**。而最后一个因为他的返回值本来就是CompletableFuture，这种转换我们称之为flatmap**。其实同样的概念在Optional API跟Stream API都找得到，有兴趣可以去看看。

这些method也都有`xxx()`, `xxxAsync(func)`, `xxxAsync(func, executor)`三个版本，就如前面所述。

经过这样的转换过程，我们把很多的future合并成单个的future。这些转换我们沒有看到任何的exception处理，因为在任何一个阶段出现exception，对于整个组合起來的future就是exception。所以我们就希望把每一个小的async invocation **compose**成一个大的async invocation。

同样在guava library中，我们可以看到composible的踪影，他是放在[Futures](https://google.github.io/guava/releases/19.0/api/docs/com/google/common/util/concurrent/Futures.html)下面的`transformXXX()`相关的methods。

## Combinable

最后，async的流程有些时候不会是单一条路的，有时候更像是[DAG](https://en.wikipedia.org/wiki/Directed_acyclic_graph)(Directed Acyclic Graph)。例如做一个爬虫程序(Crawler)，我们爬一个文章的时候，可能会爬到很多外部链接，这时候就会继续打开更多非同步的task。等到到了某个停止条件，我们就要等所有爬虫的task完成，最终等待执行完这个大的async task。

这时候我们会希望把多个future完成时当作一个future的complete，这就是combinable(可组合)的概念。跟composible的概念不同的是，composible是一个串一个，比较像是串连的感觉；相对的combinable，就比较像是并联。

来看看CompletableFuture针对这种应用有哪些method，假设原始类型CompletableFuture<T>`

| Method                                                       | With                   | Transformer         | Return Type               |
| ------------------------------------------------------------ | ---------------------- | ------------------- | ------------------------- |
| [runAfterBoth()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#runAfterBoth-java.util.concurrent.CompletionStage-java.lang.Runnable-) | `CompletableFuture<?>` | `Runnable`          | `CompletableFuture<Void>` |
| [runAfterEither()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#runAfterEither-java.util.concurrent.CompletionStage-java.lang.Runnable-) | `CompletableFuture<?>` | `Runnable`          | `CompletableFuture<Void>` |
| [thenAcceptBoth()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#thenAcceptBoth-java.util.concurrent.CompletionStage-java.util.function.BiConsumer-) | `CompletableFuture<U>` | `BiConusmer<T,U>`   | `CompletableFuture<Void>` |
| [acceptEither()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#acceptEither-java.util.concurrent.CompletionStage-java.util.function.Consumer-) | `CompletableFuture<T>` | `Conusmer<T>`       | `CompletableFuture<Void>` |
| [applyToEither()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#applyToEither-java.util.concurrent.CompletionStage-java.util.function.Function-) | `CompletableFuture<T>` | `Function<T,U>`     | `CompletableFuture<U>`    |
| [thenCombine()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#thenCombine-java.util.concurrent.CompletionStage-java.util.function.BiFunction-) | `CompletableFuture<U>` | `BiFunction<T,U,V>` | `CompletableFuture<V>`    |

跟Composible那边的method不一样的是多了一个*with*，代表的是combine的对象。这些method都有可以把两個future **combine**成一个future的特色。而**both**跟**either**，代表的是两个都完成才算完成，还是其中一个完成则算完成。

除了两两combine的这些method以外，CompletableFuture还有提供两个static methods来做combine多个futures。

| Method                                                       | Description                                                |
| ------------------------------------------------------------ | ---------------------------------------------------------- |
| [allOf(...)](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#allOf-java.util.concurrent.CompletableFuture...-) | 返回一个future，其中所有的future都完成此future才算完成。   |
| [anyOf(...)](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#anyOf-java.util.concurrent.CompletableFuture...-) | 返回一个future，其中任何一个future完成则此future就算完成。 |

## 总结

CompletableFuture跟lambda的組合，在Java8中带來了非同步的生力軍。Lambda让之前的annoymous inner class来实现async task会变成简洁非常多，而Completable future又多了composible跟combinable，让复杂的非同步流程变得非常的简洁。

再來就如前面讲的，大部分的method都有**async**，以及**async with executor**的版本。所以我们可以很明确指定到底我的task是在哪一个thread pool跑。对于UI程序，常常有一个pattern就是先async到worker thread pool去执行，处理完再到UI thread去update UI并且呈现，这个流程在新的CompletableFuture下变得更为简洁。
