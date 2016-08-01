# Message Passing

Message passing简单来讲就是一个thread想把资料丟给另外一個thread去做处理。其实在前面flow control的章节在讲producer/consumer pattern时，其实就已经点出了一个概念。但是可以看得出我们的实现过夜简单，producer跟consumer共用了同一个message变量，当producer/consumer多了，势必会造成前面所说的**race condition**。那有沒有辦法用更high-level的方式来来传递资讯呢?

工厂生产的时候，有一个名词是生产线(pipeline)，每一站都把前一个站的产出变成下一站的來源，而最后一站的产出就是这个工厂的产品。其实这个概念可以想像每一站就是一个thread，而这些站跟站之间的半成品就可以称它为message。而站跟站之间，会通过一个输送带当作管道传递，而这个管道我们称为pipe或是queue。

## Blocking Queue

在Java中有一个非常好用并且现成的东西帮我们做Consumer/Producer Pattern，那就是[BlockingQueue](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/BlockingQueue.html)，我们可以通过它轻易做到之前所说的queue满了让producer block，以及queue空了让conusmer block的结果。我把前面一个章节有关producer consumer的code改用BlockingQueue来实现。

```java
public class ProducerConsumer {
    private LinkedBlockingQueue queue = new LinkedBlockingQueue<>(5);

    public void produce(String message) throws InterruptedException {
        System.out.println("produce message: " + message);
        queue.put(message);
    }

    public void consume() throws InterruptedException {
        System.out.println("consume message: " + queue.take());
    }

    public static void main(String[] args) throws InterruptedException {
        final ProducerConsumer producerConsumer = new ProducerConsumer();
        // Create consumer thread
        new Thread(()->{
            Random random = new Random();
            try {
                while (true) {
                    producerConsumer.consume();
                    Thread.sleep(random.nextInt(1000));
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();

        // Create producer thread
        new Thread(()->{
            Random random = new Random();
            try {
                int counter = 0;
                while (true) {
                    producerConsumer.produce("message" + counter++);
                    Thread.sleep(random.nextInt(1000));
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }
}
```

可以看得到我们在produce中调用了`BlockingQueue#put()`，它会放一个message进queue中，如果满了会block；同样的在consume的地方调用 `BlockingQueue#take()`，它会从queue中取出一个messge，如果queue是空的则会block。为了刻意让queue比较容易满，我把queue的constructor传入`5`，这代表的是它的容量。我们可以改变main`中的sleep time，来模拟**供过于求**跟**供不应求**的状况。

## Pipe

有使用Unix/Linux的读者应该很熟悉[pipe](https://en.wikipedia.org/wiki/Pipeline_(software))也就是`|`这個shell operation，这是Unix当初在设计的时候非常重要的[philosophy](https://en.wikipedia.org/wiki/Unix_philosophy):

- Write programs that do one thing and do it well.
- Write programs to work together.
- Write programs to handle text streams, because that is a universal interface.

下面就是一个常见的案例：

```bash
tail accesslog | awk '{print $1}' | sort | uniq -c  | sort -nr
```

我们把前面process的stdout当作下一个process的stdin。通过这行指令，我们马上建立一个pipeline，最后把结果印出到terminal之上，有沒有很像我刚刚说的工厂的场景?

在Java中thread之间也可以这么做(虽然这么做不常见 ^^)，利用的是[PipedOutputStream](https://docs.oracle.com/javase/8/docs/api/java/io/PipedOutputStream.html)跟[PipedInputStream](https://docs.oracle.com/javase/8/docs/api/java/io/PipedInputStream.html)这两个class，两者可组合成一个Pipe，其中outputstream的产出就会是inputstream的输入。下面是个样例：

```java
public class Pipe {
    private PipedInputStream in;
    private PipedOutputStream out;

    public Pipe() throws IOException {
        in = new PipedInputStream(4096);
        out = new PipedOutputStream(in);
    }

    public String readAll() throws IOException {
        StringBuilder stringBuilder = new StringBuilder();
        try (BufferedReader reader = new BufferedReader(new InputStreamReader(in))) {
            String line;
            while ((line = reader.readLine()) != null) {
                stringBuilder.append(line);
            }
        }
        return stringBuilder.toString();
    }

    public void writeAll(String string) {
        try (PrintStream printStream = new PrintStream(out)){
            printStream.print(string);
        }
    }

    public static void main(String[] args) throws IOException {
        Pipe pipe = new Pipe();

        // produce a message
        new Thread(() -> {
            pipe.writeAll("hello pipe!!");
        }).start();

        // consume the message
        new Thread(() -> {
            try {
                System.out.println(pipe.readAll());
            } catch (IOException e) {
                e.printStackTrace();
            }
        }).start();
    }
}
```

## Other Utility

- [Pipe](https://docs.oracle.com/javase/8/docs/api/java/nio/channels/Pipe.html): nio的版本。其实跟`PipedInputStream`跟`PipedOutputStream`类似。
- [Future](https://docs.oracle.com/javase/8/docs/api/index.html?java/util/concurrent/Future.html) and [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/index.html?java/util/concurrent/CompletableFuture.html): 其实两个都同时扮演了flow control跟message passing。同样是留在后面的章节再做讨论。

## Recap

到这里我们讨论了resource sharing, flow control, message passing。这可以说是multi-threading中最重要的三个问题。但是mult-thread程序不好撰写，即使是知道了上面这些问题，还是很容易出错。因此有必要把更high-level，更一般化的问题包裝成更好用的介面。后面章节我们会把thread pool跟asynchronous invokation这两个常用的场景做更进一步介绍。
