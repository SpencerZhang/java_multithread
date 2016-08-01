# Thread Pool

前面讲的都是开一个新的thread来完成单一工作，但是开thread是有成本的。如果每一个小的task就开个thread，并且有很多的小task需要处理，那产生thread的overhead是很可观的。因此，比较好的方法是产生一堆threads，称之为thread pool，让这些开好的threads来处理这一堆小工作。

其实thread pool的概念在我们的生活中处处可见。开thread处理task就很像外包一个工作，可能我们会在网络上找一个人过来帮忙处理，等到事情处理完了合约就结束了。但是当事情多了，那就可能直接雇佣一群人当作员工来处理事情，这群正式员工就是thread pool，在此员工就是thread，而工作就是task。

在银行中也会看到thread pool的例子，办理金融业务的可能会有三四个柜台，而当我们有事情要处理会在取票机中提取一个号码。當職員处理完一个人的业务时，就会叫下一个人来处理。在这里职员就是thread，而我们的号码牌就是一个task，而这個取票机就是queue。

对!我讲到一个重点了，thread pool的三大元素就是thread, task, 跟queue。而其实thread pool就是producer consumer pattern的一种形式。consumer就是一堆threads，当queue中一有工作进来，一个空间的thread就会取出来做处理。

我把前面message passing的程序拿过来改一下，马上就做出一个thread pool：

```java
public class ThreadPool implements Runnable{
    private final LinkedBlockingQueue<Runnable> queue;
    private final List<Thread> threads;
    private boolean shutdown;

    public ThreadPool(int numberOfThreads) {
        queue = new LinkedBlockingQueue<>();
        threads = new ArrayList<>();

        for (int i=0; i<numberOfThreads; i++) {
            Thread thread = new Thread(this);
            thread.start();
            threads.add(thread);
        }
    }

    public void execute(Runnable task) throws InterruptedException {
        queue.put(task);
    }

    private Runnable consume() throws InterruptedException {
        return queue.take();
    }

    public void run()  {
        try {
            while (!shutdown) {
                Runnable task = this.consume();
                task.run();
            }
        } catch(InterruptedException e) {
        }
        System.out.println(Thread.currentThread().getName() + " shutdown");
    }

    public void shutdown() {
        shutdown = true;

        threads.forEach((thread) -> {
            thread.interrupt();
        });
    }

    public static void main(String[] args) throws InterruptedException {
        ThreadPool threadPool = new ThreadPool(5);
        Random random = new Random();

        for (int i=0; i<10; i++) {
            int fi = i;
            threadPool.execute(() -> {
                try {
                    Thread.sleep(random.nextInt(1000));
                    System.out.printf("task %d complete\n", fi);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }

        Thread.sleep(3000);
        threadPool.shutdown();
    }
}
```

这段程序跟producer consumer的code非常的接近，都有一个queue，都有produce跟consume methods。不一样的是queue里面放的是一个Runnable，代表的是一個可以执行的task。

另外在constructor中我们有一个参数是`numberOfThreads`，也就是这个thread pool中产生threads的个数，而每个thread的执行內容是一个infinite loop，做的事情就不断的从queue中拿一个task出來，去执行里面的內容。

在main method中创建一个thread pool，里面有五个职员(thread)跟一个取票机(queue)。而我抽了十张号码牌(task)，在当中随机睡了一段时间并且打印信息，最后执行的结果大概会像下面这样：

```
Thread-1: task 1 complete
Thread-0: task 0 complete
Thread-1: task 5 complete
Thread-1: task 7 complete
Thread-4: task 4 complete
Thread-4: task 9 complete
Thread-3: task 3 complete
Thread-1: task 8 complete
Thread-2: task 2 complete
Thread-0: task 6 complete
Thread-0 shutdown
Thread-3 shutdown
Thread-2 shutdown
Thread-1 shutdown
Thread-4 shutdown
```

其实自己做thread pool就是那么简单，沒有什么高深的学问。当然你不需要自己做thread pool，在Java中已经有很多现有的thread pool可以直接拿来使用，不必自己造轮子。接下來介绍一下有哪些thread pool可以拿來使用。
