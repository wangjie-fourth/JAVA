#### 1、 线程的生命周期

一个线程的生命周期有6个状态，其对应的`Thread`的内部类`State`。相关状态转换如下：

![](.\images\线程的6个状态.png)

- `NEW`：Thread还未start时
- `RUNNABLE`：Thread已经start，此时该线程处于系统得线程调度器中；
- `BLOCKED`：线程因为未获取锁而处于阻塞状态；
- `WAITING`：线程处于等待状态，等待其他线程唤醒它；
- `TIMED_WAITING`：因为sleep而处于线程等待状态；
- `TERMINATED`：中断状态，此时线程已经执行完毕；



#### 2、线程对象Thread

1、介绍

在`java`中，`Thread`是唯一代表线程的类。其他`Runnable`、`Callable`都不是线程。每个`Thread`类的实例只要`start()`后，还没结束，就代表`JVM`中的一个线程。

2、多线程的难理解

在`java`中，每次执行`Thread.start()`方法后，`JVM`就会增加一个线程。即：

- 一个代码执行流；
- 一套方法栈；

而不同执行流的同步执行是一切线程问题的根源。

（1）只有`Thread`才代表一个新线程

```java
    public static void main(String[] args) {
        new Thread(() -> {
            while (true){
                System.out.println("这是一个新线程");
            }
        }).start();
        
        while (true){
            System.out.println("这是主线程");
        }
    }
```

这里很容易**明确哪段代码在哪个线程执行**。

（2）线程池的缺点

```java
public class Demo {
    static ExecutorService threadPool = Executors.newFixedThreadPool(5);

    public static void main(String[] args) {
        threadPool.submit(() -> {
            doSomething();// 这段代码被哪个线程执行了？？？
        });

        doSomething();
    }

    private static void doSomething() {
        System.out.println("做了一点事");
    }
}
```

是不是会觉得那段代码会被新线程执行？？其实那段可能会被新线程执行，也有可能还是被主线程执行。

这是因为`Thread.start()`和线程池提交一个任务后，这个任务具体会被哪个线程执行是由**线程调度器**来决定的。默认情况下，如果这个线程池中没有空闲线程的话，那么那段代码还是会被发起的线程来执行的。

（3）多个线程的同时执行 -- 线程安全问题

```java
public class Demo {
    static int i = 0;

    public static void main(String[] args) {
        new Thread(() -> {
            doSomething();
        }).start();

        doSomething();
    }

    private static void doSomething() {// 俩个执行流|人同时在执行这段代码
        i++;
    }
}
```

这也是多线程的可怕地方，程序执行顺序不确定了。

（4）异常处理的反直觉性

```java
public class Demo {
    static int i = 0;

    public static void main(String[] args) {
        try {
            new Thread(() -> {
                throw new RuntimeException();
            }).start();

            doSomething();
        } catch (Exception e) {
            // 这里只能捕捉到当前线程跑出来的异常；上面那个新线程的异常是无法抛到这里的
        }
    }

    private static void doSomething() {
        i++;
    }
}
```

在子线程发生的异常最多只能抛到当前线程的方法栈中。子线程的异常是没法跨线程抛的，也就是说父线程是无法获取子线程的异常。



#### 3、Runnable、Callable

在`jdk1.0`的时候，`Thread`被设计成线程，`Runnable`设计成可以被线程执行的任务。但是`Runnable`有俩个限制：

- 不能直接返回值
- 不能抛出异常

这也是在`jdk1.5`时候引入`Callable`的原因，它就解决了这俩个问题。

1、`Runnable`的俩个限制

`Runnable`之所以有这俩个限制，完全是其抽象方法限制的。

```java
public interface Runnable {
    public abstract void run();
}
```

它本身没有返回值，也不抛出异常。这就导致后续重写这个方法时，也不能返回值，也不能抛出异常。

2、在使用`Runnable`时，如何能获取到线程结果的返回值呢？

设置一个共享`Map`变量，每个线程对应其中的元素。用`线程id`作为`key`，当线程结束后，在这个`Map`对应的`key`中设置`value`。以此来达到获取线程结果返回值的目的。伪代码如下：

```java
        List<Runnable> tasks = new ArrayList<>();
        Map<Long, Object> result = new ConcurrentHashMap<>();

        for (Runnable task : tasks) {
            new Thread(() -> {// 这里指的是每个task的run方法
                // 执行任务
                //...
                //...
                result.put(Thread.currentThread().getId(), "result")
            }).start();
        }
```

