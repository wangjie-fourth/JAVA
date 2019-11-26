多线程一直是`java`编程的一个难点，因为多线程难理解、难测试、使用困难。

在代码使用上：

- 安全：我们要一直主要线程安全问题
- 协同：`java`原生的协同机制 - 等待唤醒机制

在易用性上：

在`jdk1.5`之前，要想使用多线程做点事情需要自己一点一点的写，不仅费时，而且还很容易写错。



所以`jdk1.5`之后在对多线程做了很多改进。

（1）安全上

- `jdk1.6`优化了`synchronized`，让使用锁不再非常沉重；
- 提供更多线程安全的容器，如：`AtomicXXX`、`ConcurrentHashMap`、`LinkedBlockingQueue`等等

（2）协同上

- 提供新的协同机制：`Lock`与`Condidtion`
- 提高在特定场景可以使用的锁，如：`CountDownLatch`、`Semaphore`、`CyclicBarrier`

- 提高线程池，如：`Executor`、`ThreadPoolExecutor`、`ForkJoinPool`等等

- 提高直接获取线程返回值的`Api`，如：`Callable`、`Future`

上述大多数的`API`都是来自于`JUC`包。



总的来说，`java`后期对多线程做了俩个优化：

- 性能：使用`CAS`、乐观锁来代替之前获取锁；优化`synchronized`
- 使用：提高更多的线程安全容器；增加在特殊场景下的解决方案



## 二、JUC包

#### 1、为什么需要JUC

- `synchronized`性能不高；

主要是针对于`jdk1.6`以前；还有像`HashTable`、`SynchronizedMap`这种粗暴的线程安全实现容器

- 等待唤醒机制太原始了，难用；

比如说多个线程之间在多个事件下的协作，这时候就需要加很多锁。

- 不够灵活

假设你需要在某个类加锁，某个类解锁的话，`synchronized`是无法做到的。因为`synchronized`只能针对一个类的某些代码进行加锁和解锁。

为此，`jdk1.5`的时候，引入了`com.java.util.Concurrent`包。

- 提高了性能：使用了`CAS`
- 提供了在多种场景下更方便的实现

包中大多数使用`CAS`和乐观锁达到同步目的。

#### Atomic类详解

主要是提供线程安全的容器

- AtomicInteger、AtomicBoolean、AtomicLong、AtomicReference
- 全部都是CAS，提高性能
- AtomicXXX的额外用途：当作容器，以规避在java在匿名内部类和Lambda所做的语言限制

##### Atomic所解决的匿名内部类的问题

1、为什么匿名内部类要有final限制？

- 避免用户以为达到什么目的
- 防止破坏线程模型

2、解决的原理





#### Lock与Condition

1、解决的问题 
Lock - 解决同步问题 - 代替synchorized 
Condition - 解决协同问题 - 代替wait、notify

Lock的解决的问题： 
（1）当一个业务流程非常长的时候，我们拆成几个类来完成。而synchronized无法做到在多个类的代码块中同步。 
（2）一个锁只有一个等待队列，也就是说它只能完成一件事情的协同； 
（3）可以实现读写分离；比如有10个线程在进行读写操作，当只有一个线程在执行写操作的时候，就没有必要使用synchorized，但synchorized无法做到这一点； 
（4）tryLock；synchronized无法做到当锁被其他线程获取时，这个线程只能阻塞，不能去做其他人； 
（5）可以方便的实现更加灵活的优先级、公平性；即公平锁和非公平锁



3、常见的实现

- 可重入锁：ReentrantLock

  可多次加锁、解锁；以达到下面这种效果； 



### 锁相关

#### CountDownLatch：倒数拆销锁

1、使用场景 
假设一个主线程会生成五个线程，只有当五个子线程都执行完成后，主线程才接着执行。 

![1574649305587](.\images\倒数拆销锁.png)

2、介绍 
（1）倒数闭锁 
（2）用于协调一组线程的工作 
（3）简单、明了、粗暴的API

- countDown();
- await()

3、面试题：写一个代码，让程序永远阻塞住 
注意：这里不能是死循环

```java
CountDownLatch demo = new CountDownLatch(2);
demo.await();
```





#### CyclicBarrier：可以循环使用的屏障

1、什么是Cyclic？ 
循环的 
2、什么是Barrier？ 
屏障、障碍 
3、使用场景 
让多个线程在某个地方等待，一起结束。比如说让多个线程在某个时刻再一起结束。 



#### Semaphore

信号量。 
1、使用场景 
假设你有10个线程，现在你想同时有俩个线程可以去运行。这时候，你可以释放俩个信号量，让线程去捕捉。

2、信号量的获取和释放 



#### BlockingQueue、BlockingDeque





#### 乐观锁、悲观锁

乐观锁：我现在追不到你，过一会再来问一下。有种实现方式就是自旋锁(spin lock) 
悲观锁：我现在追不到你，沮丧自暴自弃（阻塞）了

> juc包中很多实现都参考CAS和乐观锁。比如：`AtomicInteger`的getAndUpdate方法

多线程中对锁的操作，一般分俩种：乐观锁、悲观锁。

简单来说，这是针对于线程对未获取到锁的态度来命名的。比如，一个线程没有获取到锁而进入到阻塞状态的话，就是悲观锁。一个线程没有获取到锁，它会通过一个循环再次来获取这个锁，这就是乐观锁。

`juc`包下很多都是乐观锁，其实现是根据在写入的时候，会去比较一下当前写入的值，是不是我要准备写入的值。

```java
public final int getAndUpdate(IntUnaryOperator updateFunction) {
    int prev, next;
    do {
        prev = get();
        next = updateFunction.applyAsInt(prev);
    } while (!compareAndSet(prev, next));
    return prev;
}
```

> 这种不断更新的操作，可以称为自旋、spin、乐观锁。





#### ConcurrentHashMap vs SynchronizedMap、HashTable

1、SynchronizedMap、HashTable 
java1.5之后引入了`ConcurrentHashMap`，用于线程安全得`Map`。但在之前使用`HashTable`和`SynchronizedMap`也是线程安全得，这俩个实现线程方式非常简单粗暴，就是在所有方法中添加`synchronized`锁。 
现在后面俩种都不推荐使用，直接用`ConcurrentHashMap`代替即可。后面俩种是典型得悲观锁。 
在`Collections`中得内部类`SynchronizedMap`有一个变量

```java
final Object mutex;// Object on which to synchronize
```

这个`mutex`其实是`mutual` + `exclusive`，即共享 + 排他 = 锁

2、ConcurrentHashMap 
而`ConcurrentHashMap`就是无锁得实现，其使用`CAS`和乐观锁，达到同步得目的。











































