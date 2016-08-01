# How to use

废话不多说，直接來看Java thread怎么使用：

```java
new Thread(() -> {
    System.out.println("hello thread");
}).start();
```

都已经Java8的时代了，我直接用Java8的语法来介绍Java thread。在上面的程序中创建了一個新的thread，thread的constructor是一个实现java.lang.Runnable`的组件。当呼叫`start()`此method时，则会启动这个thread，并且执行`Runnable#run()`。在Java8中有好用的lambda，我们直接用lambda来实现那个runnable。当然你也可以用传统的方式新建一个新的Class來實作`Runnable`。因为这个文件不是要教你怎么写Java，我就先假设你已经了解并使用过Java，熟悉Java基本常识，如果你到这里还不懂我在写什么，那建议可以先拿一本Java入门书来了解Java最基本的编程知识。

# 使用场景

通常有以下几种情況会用thread

1. IO密集型的task，或称IO bound task。如果同时需要让很多个档案，或是同时要处理很多的sokcet connection，用thread的方法去做blocking read/write可以让程序不會因为IO而导致什么事情都不能做。
2. 执行很耗运算的task，或称CPU bound task。当这种task多，我们会想要使用多个CPU cores的能力。单线程的程式只能用到single core的好处，也就是程序再怎么CPU，最多就用到一个CPU。当使用multi-thraed的时候，就可以把CPU能力拉满不浪费。
3. 非同步执行。其实不管是IO-bound task或是CPU-bound task，我们应该都还是要跟主程序做沟通，所以通常的概念都是开一个thread去做IO bound或是CPU bound task，等到他们执行完，我再把结果拿到我们主程序做后续的处理。当然新开thread不是唯一的方法，我們稍后的章节会陆续提到很多非同步执行的策略跟方法。
4. 排程(定时任务)。常见的排程方法有三种。第一种是delay，例如一秒后执行一个task。第二种是周期性的，例如每一秒钟执行一个task。第三种是指定某个时间执行一个task。当然这些应用会包装在`Timer`或是`ScheduledThreadPoolExecutor`中，但是底层都还是用thread去完成的。
5. Daemon(守护进程)，或是称之为service。有些时候我们需要某个thread专门去等某些event发生的时候才会去做对应的事情。例如写server程序，我们会希望有一个thread专门去听某个port的连接信息。如果我们写一个queue的consumer，我们也会开个thread去监听message收到的event。

我们在写Java程序中，其实我们每天都在跟thread共舞，即便我们沒有直接创建这些thread，但是我们所用的framework可能都已经构建在multi-thread的环境之上，因此我们更需要了解multi-thread的课题。下个章节我们来讨论thread之间的synchronization的问题。
