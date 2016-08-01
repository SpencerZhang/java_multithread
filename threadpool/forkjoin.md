# ForkJoin Pool

在Java7的时候，推出了一个很不一样的thread pool，他实现了[working stealing scheduling](https://en.wikipedia.org/wiki/Work_stealing)。这是什么概念呢? 试想有一个大的task，如果一个人无法完成的话，那我们很直觉的会把大的task，切成很多中型的task，再把他分成几个小的task，让整个团队可以协作完成。这些小的task完成之后，中型的task再把它整合起來，最终完成整合后就完成了大的task。

咦，这不就是[recursion](https://en.wikipedia.org/wiki/Recursion)或是[divide & conquer](https://en.wikipedia.org/wiki/Divide_and_conquer_algorithms)吗? 沒错，很多的算法都是类似这种divide and conquer的形式。这种算法的特性是，要等到所有的subtask都完成之后，才可以算出中间的结果，再来可以算出最后完整的结果。而ForkJoin Pool就是为了解決这种问题平行化的solution。

## Work Stealing

相较於一般的Thread Pool，大家共用一个queue，ForkJoin Pool是**每个thread有一个自己的queue**，而且是双向的queue(deque)。当一个task进来，他会把一部分的工作fork(切)出来先放到queue的最后面，另外一部分开始做。当然可能做进去后，发现task还是太大，就会继续切更小，并再放到queue的最后方。如此一边切一边往下执行，直到task够小可以直接运算为止。

当一个小工作完成之后，他会从最后端的task拿出來(其实这边比较像stack的行为)，继续往下做。当然recursion的程序除了divide以外，还有merge results的动作，这里称之为join。join会取得每个subtask的结果，最终称为目前task的结果返回。

那其他thread呢? 当其他thread身上的queue是空的时候，他会去別的thread的queue最前头steal一个task到自己的queue当中。之后的行为就跟前面一模一样。这有沒有很像一个团队做task的时候的行为? 所以如果我们把thread pool类比为取票机，forkjoin pool就很像做task一样。

## How to use

前面讲的是概念，现在来讲实际怎么实现。我拿费氏数列当作例子，虽然他recursion不是他的最佳解法，但是他的定义很recursive。为了解释方便，我们先不管它的效能。来看经典recursion版本

```java
public int fib(int n) {
    if (n <= 1)
        return n;
    return fib(n-1) + fib(n-2);
}
```

再来看是ForkJoin版本

```java
public class Fibonacci extends RecursiveTask<Integer> {
    final int n;
    Fibonacci(int n) { this.n = n; }

    public Integer compute() {
        if (n <= 1)
            return n;
        Fibonacci f1 = new Fibonacci(n - 1);
        f1.fork();
        Fibonacci f2 = new Fibonacci(n - 2);
        return f2.compute() + f1.join();
    }
}
```

Recursion task要继承於[RecursiveTask](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/RecursiveTask.html)或是[RecursiveAction](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/RecursiveAction.html)。前者是有返回值，后者沒有。继承需要实现compute()`這個method，里面实现divide and conquer的逻辑。在当中我们可以直接调用subtask的`compute()`，也可以调用subtask的`fork()`，代表的是把subtask丟到queue。等到需要他的结果时，再调用`join()`，它会把subtask结果返回，再把所有的result去整合成目前task的result。

实际执行我们需要有一个ForkJoinPool。我们可以直接用大家共用的common forkjoin pool，也就是`ForkJoinPool.commonPool()`。下面是执行这个RecursiveTask的样例：

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    Fibonacci fibonacci = new Fibonacci(10);
    ForkJoinPool.commonPool().execute(fibonacci);
    System.out.println(fibonacci.get());
}
```

# 跟传统Thread Pool比较

其实这是两种不同的执行策略，分別是producer consumer跟recursion。前者适合每个task是独立的，他可以把大事小事都平均分摊在每个thread去执行；后者是通过divide and conquer演算法，用work stealing的方式去执行。所以主要还是要看你的task是哪一种类型居多。

而ForkJoinPool有一个很大的好处是减少thread因为blocking造成context switching。不管是fork, compute, join都机会不會blocking(只有join少数情况会要等待结果)。这可以讲thread一直保持running的状态，一直到时间到了被context switch(上下文切换)，而不是自己卡住了造成的context switch(上下文切换)。

但ForkJoinPool对於不可分割的task，并且处理时间差异很大的场景比较不适合，毕竟每个thread都有一个queue。就很像在大卖场排队结帐，只要运气不好排到一个前面卡比较久的task就要等比较久。但是別排又沒有开到可以把你steal走，那就沒有办法做到先到先处理的特性。

# Java8 Parallel Stream

Java8的[Stream API](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html)的parallel stream事实上也是用ForkJoinPool，他通过[Spliterator](https://docs.oracle.com/javase/8/docs/api/java/util/Spliterator.html)来定义怎么去split(或fork)你的input data。若执行结果需要collect，就会join回最后的result。
