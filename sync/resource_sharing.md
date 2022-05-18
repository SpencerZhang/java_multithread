# Resource Sharing(资源共享)

Resource sharing是multi-thread第一个要遇到的问题，也是最重要的问题。如第一章所言，thread之间本来就是共用同一个address space，所以大家都可以用到在JVM中的所有资源，只要存取得到。那就马上衍生的问题就是，若大家一起共用的时候该怎么管理状态?

在Java中有一个非常好用的keyword叫做`synchronized`，我们用它来处理资源共用的问题。它有以下几种用法：

```java
synchronized(myReource) {
   // do something to my resource.
}
```

上面这段code可以保证同时只有你目前执行中的thread可以跑这段code，当然如果你在要执行这段程序的时候，已经有其他thread在使用，那就要等它执行完才有机会轮到你去使用。

另外一种写法=是包在method的定义前面：

```java
public synchronized void myMethod() {
    //code
}

public static synchronized void myStaticMethod() {
    //code
}
```

其实等同于：

```java
public void myMethod() {
    synchronized(this) {
        //code
    }
}

public static void myStaticMethod() {
     synchronized(MyClass.class) {
         //code
     }
}
```

也就是可以写成method level的描述，来保证这个method同时只有一个thread会去存取目前该method所属的instance或是class。

# Deadlock(死锁)

基本上sychronized就是一种lock，当你执行synchronized(myObject){}`的同时，就是锁定`myObject`资源。当如果有thread先lock **A**再想lock **B**，而另一个thread是先lock **B**再lock **A**，**那就是会造成deadlock**。我们在撰写程序的时候，应该要统一以先取得A再取得B的顺序，也就是前后关系要一致，这样就可以避免掉deadlock。再来可以思考真的要把lock分到**A**跟**B**那么细吗? 还是统一就去取得A就好了，但这就涉及到程序设计的问题了。

# Race Condition(竞态条件)

另外一个极端就是**Race condition**，通常发生在沒有对resource做`synchronized`保护。例如下面这个程序就是会有race condition：

```java
public class MyClass {
    private int i;
    public int getAndIncr() {
        return i++;
    }
}
```

因为i++在java底层是分三个动作: 分別是取得值，值+1，返回变量。如果流程如下：

```
Thread 1: get value: 100
Thread 2: get value: 100
Thread 2: incr value: 101
Thread 2: set value: 101
Thread 1: incr value: 101
Thread 1: set value: 101
```

上面如果连个thread是这样执行就可能造成结果错误。因此如果改成以下这种写法就不会出错：

```java
public class MyClass {
    private int i;
    public synchronized int getAndIncr() {
        return i++;
    }
}
```

也可以直接用[AtomicInteger](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicInteger.html)来解決这个问题： 

```java
public class MyClass {
    private final AtomicInteger i = new AtomicInteger();
    public int getAndIncr() {
        return i.getAndIncrement();
    }
}
```

# Thread Safe(线程安全)

当一个class或method可以在multi-thread环境下不会有race condition，我们可以称此class或method为**thread safe**。此时你可以很放心的在multi-thread的环境操作这些resource而不用再包`synchronized`。但是thread safe并不是一个class或method一定要提供的职责，毕竟让resource可以是thread safe必定有其代价。例如用了很多`synchronized`，但如果你只是在single thread的环境使用，那岂不是增加了执行的overhead? 另一种思考是，让使用library的人去决定`synchronized`包装的granularity(颗粒度)也许会更适合。

因此，在java collection library中，大部分的collection其实都不是thread safe。如果想让你的collection是thread safe，可以通过[Collections](https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html)中的很多helper methods来提供。例如

```java
syncedCol = Collections.synchronizedCollection(myCol);
syncedList = Collections.synchronizedList(myList);
syncedSet = Collections.synchronizedSet(mySet);
syncedMap = Collections.synchronizedMap(myMap);
```

则可以把原本的non-thread-safe的容器包成thread safe的容器。

# Immutable Object(不可变对象)

Immutable(不能修改) object是另外一种resource sharing的策略。概念是resource在唯独的情況下沒有共用的问题，也不需要lock，只要里面资料不会变动。但如果想要修改怎么办? 此时以产生取代修改，也就是会再产生另外一个**immutable object**。在Java中最经典的例子就是`String`，所有的String的资源是不能修改的，如果我们执行以下的code：

```
newStr = str + "2";
```

此时的`newStr`跟`str`会是连个独立的object。这种方式最大的好处是即便很多thread去存取也不用去lock，可以增加平行化的程度。

# Other Utilities(其他工具)

在Java中还有其他部分也跟resource sharing有关，例如：

- [Semaphore](https://docs.oracle.com/javase/8/docs/api/index.html?java/util/concurrent/Semaphore.html): 可以算是countable lock，也就是会有一定数量的resources可以取得。
- [ReadWriteLock](https://docs.oracle.com/javase/8/docs/api/index.html?java/util/concurrent/locks/ReentrantReadWriteLock.html): Lock分read跟write，理论上read access可以multiple thread，write access只能single thread。Read/Write lock把read跟write分开可以让synchronization的成本降低
- [Atomics](https://docs.oracle.com/javase/8/docs/api/index.html?java/util/concurrent/atomic/package-summary.html): 上面已经介绍过，把一些操控primitives的动作变成是atomic(不可分割的) operation。