# Conclusions(结论)

很快的就讲到尾声(其实我写了好几个晚上 XD)，让我们来回顾一下讲了什么

在Java Thread章节中，我们讲了thread最基本的概念，并且知道怎么创建thread，怎么定义thread要做的事情。

有了thread之后，那开始要解决synchronization的问题。其实这个章节讲的是Java最底层的同步机制，说真的除了`synchronized` keyword，大部分几乎不会直接用到。但是通过resource sharing, flow control, message passing三种概念，来帮助勾勒出同步最常见的几种场景。

接下來讲到thread pool，我们在thread pool中提到了single queue/multi consumer这种传统的thread pool，我把它类比成**取票机**。接下来还介绍了queue-per-thread并且通过work stealing方式的`ForkJoinPool`，这种pool对於divide and conquer分治的方式特別适合，我把它类比成**团队总大任务**的概念。

最后讲到`Future`跟`CompletableFuture`。Future是async invocation中caller看到对未来返回值的一个容器，而CompleableFuture是Future的实现，但也扮演了Promise的角色，他代表的是Async invocation的callee在做完时**承诺**要把结果放进去。

而CompletableFuture除了Completable的特性外，其实还有Listenable, Composable, Combinable的特性。并且用monad的map/flatmap来transform这些sync methods变成async methods。

不知道您有沒有发现，每一章节都把前一章节的概念延伸。

1. `synchorinzed` keyword
2. `wait()`/`notify()`包在synchronized中，并且实现producer/consumer
3. 用blockqueue；来做producer/consumer
4. 用producer/consumer的概念来做thread pool
5. 通过thread pool的submit来返回非同步的结果，也就是Future
6. CompletableFuture可以在thread pool中执行

基本上这份文件应该已经涵盖了所有在Java中你所需要的基本multithread知识。而且也看到了整个JDK有关concurrency的演进

- JDK最初版的thread
- JDK1.5的`java.util.concurrent`
- JDK1.7的`ForkJoinPool`
- JDK1.8的`CompletableFuture`

希望这个教学各位会满意。有任何疑问欢迎留言讨论。至于这份文件严禁商业用途，而且不允予许转载。如觉得这份文件很值得分享，欢迎分享原文[Java多线程的基本常识](https://www.gitbook.com/book/popcornylu/java_multithread/details)。
