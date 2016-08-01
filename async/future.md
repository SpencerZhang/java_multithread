# Future

[Future](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Future.html)是代表一个非同步调用的返回结果，而这个结果会在未来某一个时间点可以取得。这样讲有点抽象，那举个例子好了，送洗衣服就是一个非同步的掉，因为衣服是交给別人洗而非自己洗，而洗好的衣服是一个**未來**会发生的结果，这个结果被Future这个class包装起來。

我们再更具体一点的把它变成实际的代码，`Future`是一有一个类型参数的class，使用上大概会像這样：

```java
Future<Clothes> future = laundryService.serviceAsync(myClothes);
```

洗衣店提供了一个非同步的服务，所以返回一个`Future`代表的是非同步的结果(Asynchronous Result)。因为是非同步，所以这个method不会卡住。拿到这个future之后，我们可以选择做別的事情，或是马上blocking取得。

```java
// block until result is available
Clothes clothes = future.get();
```

因为是非同步，所以我们可以一次执行很多个非同步的task，让他们可以并行的去处理，最后再一次等所有的非同步结果。这在生活中也经常发生，例如我去夜市买鸡排豆花跟珍奶，我也不会呆呆的在每一个摊位前面等他一个一个做好，我可能会跟老板说等等过来拿，让这几个摊位可以**并行**的处理我的tasks。

Future是一个interface，所以需要具体的实现。但通常不需要自己实现，还记得前面[ThreadPool](async/basicpool.md)的章节就有提到`Future`了吗? 事实上[ExecutorService#submit](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ExecutorService.html#submit-java.util.concurrent.Callable-)就提供了一个非同步执行的实现，并且返回一个`Future`，一般来讲我们只要使用这个method来实现我们的非同步执行就可以了。

所以上面的代码的具体实现可能长这样：

```java
public class LaundaryService {
    private ExecutorService executorService = /*...*/;

    public Future<Clothes> serviceAsync(Clothes dirtyClothes) {
        return executorService.submit(()-> service(dirtyClothes));
    }

    public Clothes service(Clothes dirtyClothes) {
        return dirtyClothes.wash();
    }
}
```

如此一來，这个LaundaryService当中有一个threadpool帮忙洗衣服，当调用serviceAsync`，则会产生一个task在threadpool中执行，并且先返回一个Future代表这个非同步的结果，就很像是一个取货单一样。caller(调用方)在之后再通过future取得最后完成的结果。