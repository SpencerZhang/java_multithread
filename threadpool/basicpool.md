# Basic Pool

在Java中，thread pool都会实现一个接口[Executor](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executor.html)，事实上更明确的说是实现[ExecutorService](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ExecutorService.html)这个接口。前者只定义了一个简单的`execute` method，就跟前面一个章节的`execute`定定义一模一样，就是在thread pool中执行一个task。

```java
public interface Executor {
    void execute(Runnable command);
}
```

后者继承了Exectuor接口，定义了更多的method，

```java
public interface ExecutorService extends Executor {
    void shutdown();

    List<Runnable> shutdownNow();

    boolean isShutdown();

    boolean isTerminated();

    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> Future<T> submit(Callable<T> task);

    <T> Future<T> submit(Runnable task, T result);

    Future<?> submit(Runnable task);

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

| Methods              | Description                                                  |
| -------------------- | ------------------------------------------------------------ |
| submit               | 可以调用沒有返回值的task(Runnable)跟有返回值的task(Callable)，并且会返回一个[Future](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Future.html)。这个Future的概念容我稍后介绍。 |
| invokeAll            | 一次执行多个task，并且取得所有task的future objects           |
| invokeAny            | 一次执行多个task，并且取得第一个完成的task的future object    |
| shutdown shutdownNow | 让整个thread pool的threads都停止，简单讲就是打烊了。         |
| awaitTermination     | 等待所有shutdown后的task都执行完成。可以说是打烊并且所有善后都处理完了。 |

另外还有一种较特殊的thread pool称为ScheduledExecutorService](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ScheduledExecutorService.html)。他除了可以做原本submit task到thread pool以外，还可以让这个task不会立刻执行。如下:

```java
public interface ScheduledExecutorService extends ExecutorService {

    public ScheduledFuture<?> schedule(Runnable command,
                                       long delay, TimeUnit unit);

    public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay, TimeUnit unit);


    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit);


    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit);

}
```

| Methods             | Description                                                  |
| ------------------- | ------------------------------------------------------------ |
| schedule            | 让task可以在一段时间之后才执行。                             |
| scheduleAtFixedRate | 除了可以让task在一段时间之后才执行之外，还可以固定周期执行。 |

在看完thread pool的抽象定义之后，我们来看看有哪些现成的实现可以拿來使用。

## Executors

在Java中，大部分的thread pool都是通过[Executors](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executors.html)来产生，里面有非常多的factory method来去产生不同类型的thread pool。

| Name                                                         | Description                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| Executors.newSingleThreadPool()  Executors.newSingleThreadScheduledExecutor() | 产生一个只有一个thread的thread pool。                        |
| Executors.newFixedThreadPool(int nThreads)                   | 产生固定thread数量的thread pool                              |
| Executors.newCachedThreadPool()                              | 产生沒有数量上限的thread pool。Thread的数量会根据使用情况动态的增加或是减少。 |
| Executors.newScheduledThreadPool(int corePoolSize)           | 产生固定数量可以做scheduling的thread pool。                  |
| Executors.newWorkStealingPool()                              | 产生work stealing的thread pool，这种类型的thread pool下一章节会介绍。 |

基本上大部分的情况之下，我们已经不会自己产生thread了，通过thread pool的方式可以更有效的管理我们的threads。再想想前面的银行取票机的例子，也许一个银行需要一般业务的取票机，另外有一个外汇的取票机，还有可能一个专属於VIP的取票机。根据不同的业务需求，我们用不同的thread pool去管理。可以避免较为重要的工作，反而被一些比较不重要但是做比较久的task卡住。在我們的multi thread的环境之下，我们也会有类似的场境。

接下來我们介绍一个比较特別的pool，称之为ForkJoinPool。