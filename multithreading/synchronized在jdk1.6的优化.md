在`java`之前，没有什么语言提供加锁得语言级别得支持。也就是说之前`c`、`c++`都是要引用第三方类库来达到加锁得功能。所以`java`一出生就具有这种功能，让它备受瞩目。

但是在`jdk1.6`之前，这种锁运行性能不高。因为`synchronized`是一种**重量级别锁**得实现。而在`jdk1.6`后，`java`就用了一堆其他操作来提供锁性能的操作。 

而在`jdk1.6`，对锁进行处理。认为在某些情况下，是不需要获取锁也可以达到线程安全目的，在特定情况下，再让线程获取锁。这就是**锁的膨胀过程**。

1、加`synchronized`跟不加有什么区别

加`synchronized`就是加锁。对应`java`中，就是对应的**监视器**。对应在字节码上的区别：

```java
public void test(){
    synchronized (this){
        System.out.println("test");
    }
}
```

其字节码上会多出俩个命令

```java
  public test()V
...
    MONITORENTER
   L0
    LINENUMBER 15 L0
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    LDC "test"
    INVOKEVIRTUAL java/io/PrintStream.println (Ljava/lang/String;)V
   L5
    LINENUMBER 16 L5
    ALOAD 1
    MONITOREXIT
...
```

即：`MONITOREXIT`和`MONITORENTER`指令。 其中，`MONITOR`就是监视器。 

2、进入`MONITORENTER`的时候，一定需要获取监视器吗？离开`MONITOREXIT`的时候，一定要释放监视器吗？

答案是不一定。这就是因为`jdk1.6`所做的优化。

在`JVM`中，每个对象包含俩部分：对象头、对象数据。其中对象数据就是我们写代码能看到的属性之类的东西，而在对象头中含有包含同步相关信息。

> [对象头的定义](https://github.com/openjdk/jdk13/blob/7b61cd194c1b5fd06a9ef90ed7a3d51dbee9c859/src/hotspot/share/oops/markOop.hpp)
>
> [锁膨胀的过程](https://github.com/openjdk/jdk13/blob/7b61cd194c1b5fd06a9ef90ed7a3d51dbee9c859/src/hotspot/share/runtime/synchronizer.cpp)

对象头中就包含了记录锁的标志位。如下图：

![](F:\ideaWorks\myBook\java\multithreading\images\对象头锁膨胀.png)

以下是某个对象头锁的膨胀过程：

- 无锁：没有任何线程争抢这个对象，则这个对象头就变成无锁状态；只要有线程来，不需要获取`monitor`，直接用
- 偏向锁：一直以来都是同一个线程获取这个对象，则这个对象头就变成偏向锁；如果下次还是这个线程获取我，也不需要去获取`monitor`
- 轻量锁：有多个线程竞争的情况发生，但在绝大多数情况下，**在任何时刻，都只有一个线程来获取这个锁**。则这个对象头就变成轻量锁；此时，只要你获取这个对象，就直接用，不需要获取`monitor`
- 重量锁：在同一时刻，有多个线程在同时竞争锁。这个时候就必须要获取`monitor`

> 对象的对象头的碰撞过程，其实是多线程环境中，针对不同环境下，对象头变化的过程。

3、锁粗化、锁细化：编译器层面的优化

（1）锁粗坏

如果同一个锁要连续频繁加锁、解锁，就会粗坏为更大范围的锁。比如：

```java
// monitorenter
for(int i = 0; i < 10 ;i++){
    foo();
}
// monitorexit

public static synchronized void foo(){
    System.out.println("");
}
```

（2）锁消除

经过**逃逸分析**，发现不可能有其他线程跟我竞争锁，就会把这个锁消除掉。比如：

```java
public static void foo(){
    Object lock = new Object();
    synchronized (lock){
        ...
    }
}
```

但是如果这个锁对象逃出去的话，就不会产生锁消除。比如：

```java
public static Object foo(){
    Object lock = new Object();
    synchronized (lock){
        ...
    }
    return lock;
}
```

