## 死锁

Java 死锁产生的四个必要条件：

1. 互斥条件，即当资源被一个线程使用（占有）时，别的线程不能使用
2. 不可剥夺条件，资源请求者不能强制从资源占有者手中夺取资源，资源只能由资源占有者主动释放
3. 请求和保持条件，即当资源请求者在请求其他的资源的同时保持对原有资源的占有
4. 循环等待条件，即存在一个等待循环队列：p1 要 p2 的资源，p2 要 p1 的资源，形成了一个等待环路



定位死锁：

检测死锁可以使用 **jconsole**工具，或者使用 **jps 定位进程 id，再用 jstack 定位死锁**

Linux 下可以通过 top 先定位到 CPU 占用高的 Java 进程，再利用 `top -Hp 进程id` 来定位是哪个线程，最后再用 jstack 的输出来看各个线程栈





## synchronized

Java 中的每一个对象都可以作为锁。 具体表现为以下 3 种形式：

1. 对于普通同步方法，锁是当前实例对象。 
2. 对于静态同步方法，锁是当前类的 Class 对象。 
3. 对于同步方法块，锁是 Synchonized 括号里配置的对象。



JVM 基于进入和退出 Monitor 对象来实现方法同步和代码块同步，代码块同步是使用 monitorenter 和 monitorexit 指令实现的，方法的同步同样可以使用这两个指令来实现。monitorenter 指令是在编译后插入到同步代码块的开始位置，而 monitorexit 是插入到方法结束处和异常处，JVM 要保证每个 monitorenter 必须有对应的 monitorexit 与之配对。任何对象都有一个 monitor 与之关联，当且一个monitor 被持有后，它将处于锁定状态。线程执行到 monitorenter 指令时，将会尝试获取对象所对应的 monitor 的所有权，即尝试获得对象的锁。



### 对象头

在`JVM`中，一个`Java`对象在内存的布局，会分为三个区域：对象头、实例数据以及对齐填充。

synchronized 用的锁是存在 Java 对象头里的。如果对象是数组类型，则虚拟机用 3个字宽（Word）存储对象头，如果对象是非数组类型，则用 2 字宽存储对象头。在 32 位虚拟机中，1 字宽等于 4 字节，即 32bit；64位虚拟机下，一个字宽大小为 8 字节。

![](.\JUC\对象头.jpeg)

Java 对象头里的 Mark Word 里默认存储对象的 HashCode、分代年龄和锁标记位。

![](.\JUC\对象头存储结构.jpeg)

在运行期间，Mark Word 里存储的数据会随着锁标志位的变化而变化。

![](.\JUC\锁状态变化.jpeg)

当对象状态为偏向锁时，Mark Word存储的是偏向的线程ID。

当状态为轻量级锁时，Mark Word 存储的是指向线程栈中 Lock Record 的指针。

`LockRecord`是什么呢？由于`MarkWord`的空间有限，随着对象状态的改变，原本存储在对象头里的一些信息，如`HashCode`、对象年龄等，就没有足够的空间存储。这时为了保证这些数据不丢失，就会拷贝一份原本的`MarkWord`放到线程栈中，这个拷贝过去的`MarkWord`叫作`Displaced Mark Word`，同时会配合一个指向对象的指针，形成`LockRecord`（锁记录），而原本对象头中的`MarkWord`，就只会存储一个指向`LockRecord`的指针。



### 锁升级

锁一共有 4 种状态，级别从低到高依次是：无锁状态、偏向锁状态、轻量级锁状态和重量级锁状态，这几个状态会随着竞争情况逐渐升级。锁可以升级但不能降级，意味着偏向锁升级成轻量级锁后不能降级成偏向锁。这种锁升级却不能降级的策略，目的是为了提高获得锁和释放锁的效率。

#### 偏向锁

大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁。

当一个线程访问同步块并获取锁时，会在对象头和栈帧中的锁记录里存储锁偏向的线程 ID，以后该线程在进入和退出同步块时不需要进行 CAS 操作来加锁和解锁，只需简单地测试一下对象头的 Mark Word 里是否存储着指向当前线程的偏向锁。如果测试成功，表示线程已经获得了锁。如果测试失败，则需要再测试一下 Mark Word 中偏向锁的标识是否设置成 1 （表示当前是偏向锁）：如果没有设置，则使用 CAS 竞争锁；如果设置了，则尝试使用 CAS 将对象头的偏向锁指向当前线程。



**偏向锁的撤销**：

偏向锁使用了一种等到竞争出现才释放锁的机制，所以当其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁。

1. 在一个安全点（在这个时间点上没有正在执行的字节码）停下拥有锁的线程；

2. 遍历线程栈，如果存在锁记录的话，需要修复锁记录和Mark Word，使其变成无锁状态。

3. 唤醒当前线程，将当前锁升级成轻量级锁。

![](.\JUC\偏向锁的获得和撤销.jpeg)

**关闭偏向锁**：

偏向锁是默认是延迟的，不会在程序启动时立即生效，如果想避免延迟，可以加 VM 参数 `-XX:BiasedLockingStartupDelay=0` 来禁用延迟。JDK 8 延迟 4s 开启偏向锁原因：在刚开始执行代码时，会有好多线程来抢锁，如果开偏向锁就会造成偏向锁不断的进行锁撤销和锁升级的操作，效率反而降低。

禁用偏向锁，运行时在添加 VM 参数 `-XX:-UseBiasedLocking=false` 禁用偏向锁。



#### 轻量级锁

线程在执行同步块之前，JVM 会先在当前线程的栈桢中创建用于存储锁记录的空间，并将对象头中的 Mark Word 复制到锁记录中，称为 `Displaced Mark Word`。然后线程尝试使用 CAS 将对象头中的 Mark Word 替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁。

1. 如果自旋后还未等到锁，则说明目前竞争较重，进入锁膨胀过程
2. 如果是自己执行了 synchronized 锁重入，那么再添加一条 mark word 设置为null 的锁记录作为重入的计数



轻量级解锁时，会使用原子的 CAS 操作将 Displaced Mark Word 替换回到对象头， 如果成功，则表示没有竞争发生。如果失败，表示当前锁存在竞争，锁就会膨胀成重量级锁。

![](.\JUC\轻量级锁.jpeg)



| 锁       | 优点                                                         | 缺点                                           | 适用场景                                 |
| -------- | ------------------------------------------------------------ | ---------------------------------------------- | ---------------------------------------- |
| 偏向锁   | 加锁解锁不需要额外消耗，和执行非同步方法相比仅存在纳秒级差距 | 如果线程间存在锁竞争，会带来额外的锁撤销的消耗 | 适用于只有一个线程访问同步块场景         |
| 轻量级锁 | 竞争的线程不会阻塞，提高了响应速度                           | 始终得不到锁的线程，使用自旋会消耗CPU          | 追求响应时间，同步块执行速度非常快的场景 |
| 重量级锁 | 线程竞争不会使用自旋，不会消耗CPU                            | 线程阻塞，响应时间慢                           | 追求吞吐量，同步块执行速度较慢           |





## 原子操作

CAS 实现原子操作的三大问题：

1. ABA 问题。因为 CAS 需要在操作值的时候，检查值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是 A，变成了 B，又变成了 A，那么使用 CAS 进行检查时会发现它的值没有发生变化，但是实际上却变化了。

   ABA 问题的解决思路就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加 1，那么 A→B→A 就会变成 1A→2B→3A。从 Java 1.5 开始，JDK 的 Atomic 包里提供了一个类 AtomicStampedReference 来解决 ABA 问题。这个类的 compareAndSet 方法的作用是首先检查当前引用是否等于预期引用，并且检查当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。

2. 循环时间长开销大。自旋 CAS 如果长时间不成功，会给 CPU 带来非常大的执行开销。如果 JVM 能支持处理器提供的 pause 指令，那么效率会有一定的提升。pause 指令有两个作用：第一，它可以延迟流水线执行指令（de-pipeline），使 CPU 不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟时间是零；第二， 它可以避免在退出循环的时候因内存顺序冲突（Memory Order Violation）而引起 CPU 流水线被清空（CPU Pipeline Flush），从而提高 CPU 的执行效率。

3. 只能保证一个共享变量的原子操作。当对一个共享变量执行操作时，我们可以使用循环 CAS 的方式来保证原子操作，但是对多个共享变量操作时，循环 CAS 就无法保证操作的原子性，这个时候就可以用锁。还有一个取巧的办法，就是把多个共享变量合并成一个共享变量来操作。从 Java 1.5 开始， JDK 提供了 AtomicReference 类来保证引用对象之间的原子性，就可以把多个变量放在一个对象里来进行 CAS 操作。



## Java内存模型

线程之间的共享变量存储在主内存（Main Memory）中，每个线程都有一个私有的本地内存（Local Memory），本地内存中存储了该线程以读/写共享变量的副本。本地内存是 JMM 的一个抽象概念，并不真实存在。它涵盖了缓存、写缓冲区、寄存器以及其他的硬件和编译器优化。

![](.\JUC\JMM.png)

### 三大特性

**可见性**

指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。

单线程程序不会出现内存可见性问题。但在多线程环境中可就不一定了，由于线程对共享变量的操作，都是拷贝到各自的工作内存运算的，运算完成后才刷回主内存中。另外指令重排以及编译器优化也可能导致可见性问题。

JMM 通过限制编译器和处理器的重排序来为程序员提供内存可见性保证。

**原子性**

一个或者多个操作在 CPU 执行的过程中不被中断的特性称为原子性。

在Java中使用了原子类和synchronized 来确保原子性。

**有序性**

JMM 允许编译器和 CPU 优化指令顺序，但通过内存屏障机制和 `volatile` 等关键字可以保证线程间的执行顺序。

有序性通过 `happens-before` 规则来保证。

### volatile

volatile 是轻量级的 synchronized，它在多处理器开发中保证了共享变量的**可见性**。可见性的意思是当一个线程修改一个共享变量时，另外一个线程能读到这个修改的值。**主要作用是保证可见性和禁止指令重排优化。**

使用 volatile 修饰的共享变量，底层通过汇编 lock 前缀指令进行缓存锁定：

1. 将当前处理器（线程）缓存行的数据写回到系统内存。
2. 这个写回内存的操作会使在其他 CPU 里缓存了该内存地址的数据无效。

为了提高处理速度，处理器不直接和内存进行通信，而是先将系统内存的数据读到内部缓存后再进行操作，但操作完不知道何时会写到内存。如果对声明了 volatile 的变量进行写操作，JVM 就会向处理器发送一条 Lock 前缀的指令，将这个变量所在缓存行的数据写回到系统内存。但是，就算写回到内存，如果其他处理器缓存的值还是旧的，再执行计算操作就会有问题。所以，在多处理器下，为了保证各个处理器的缓存是一致的，就会实现缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器对这个数据进行修改操作的时候，会重新从系统内存中把数据读到处理器缓存里。



当写一个 volatile 变量时，JMM 会把该线程对应的本地内存中的共享变量值刷新到主内存。

当读一个 volatile 变量时，JMM 会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量。

线程 A 写一个 volatile 变量，随后线程 B 读这个 volatile 变量，这个过程实质上是线程 A 通过主内存向线程 B 发送消息。



volatile重排序规则：

![](JUC\volatile重排序规则表.jpeg)

* 当第二个操作是 volatile 写时，不管第一个操作是什么，都不能重排序。这个规则确保 volatile 写之前的操作不会被编译器重排序到 volatile 写之后。 
* 当第一个操作是 volatile 读时，不管第二个操作是什么，都不能重排序。这个规则确保 volatile 读之后的操作不会被编译器重排序到 volatile 读之前。 

为了实现 volatile 的内存语义，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。

* 在每个 volatile 写操作的前面插入一个 StoreStore 屏障。

* 在每个 volatile 写操作的后面插入一个 StoreLoad 屏障。  
* 在每个 volatile 读操作的后面插入一个 LoadLoad 屏障。
* 在每个 volatile 读操作的后面插入一个 LoadStore 屏障。

| 屏障类型   | 指令示例                    | 说明                                                         |
| ---------- | --------------------------- | ------------------------------------------------------------ |
| LoadLoad   | Load1; LoadLoad; Load2;     | 确保`Load1`指令数据的装载，发生于`Load2`及后续所有装载指令的数据装载之前。 |
| StoreStore | Store1; StoreStore; Store2; | 确保`Store1`数据的存储对其他处理器可见（刷新到内存中），并发生于`Store2`及后续所有存储指令的数据写入之前。 |
| LoadStore  | Load1; LoadStore; Store2;   | 确保`Load1`指令数据的装载，发生于`Store2`及后续所有存储指令的数据写入之前。 |
| StoreLoad  | Store1; StoreLoad; Load2;   | 确保`Store1`数据的存储对其他处理器可见（刷新到内存中），并发生于`Load2`及后续所有装载指令的数据装载之前。`StoreLoad Barriers`会使该屏障之前的所有内存访问指令（存储和装载）完成之后，才执行该屏障之后的内存访问指令。 |



synchronized 无法禁止指令重排和处理器优化，为什么可以保证有序性可见性？

1. 加了锁之后，只能有一个线程获得到了锁，获得不到锁的线程就要阻塞，所以同一时间只有一个线程执行，相当于单线程，由于数据依赖性的存在，单线程的指令重排是没有问题的

2. 线程加锁前，将清空工作内存中共享变量的值，使用共享变量时需要从主内存中重新读取最新的值；线程解锁前，必须把共享变量的最新值刷新到主内存中



### 锁

当线程释放锁时，JMM 会把该线程对应的本地内存中的共享变量刷新到主内存中。

当线程获取锁时，JMM 会把该线程对应的本地内存置为无效。从而使得被监视器保护的临界区代码必须从主内存中读取共享变量。

线程 A 释放锁，随后线程 B 获取这个锁，这个过程实质上是线程 A 通过主内存 向线程 B 发送消息。



ReentrantLock 的实现依赖于 Java 同步器框架 AbstractQueuedSynchronizer（ AQS）。AQS 使用一个整型的 volatile 变量（命名为 state）来维护同步状态。

公平锁和非公平锁释放时，最后都要写一个 volatile 变量 state。 

公平锁获取时，首先会去读 volatile 变量。 

非公平锁获取时，首先会用 CAS 更新 volatile 变量，这个操作同时具有 volatile 读 和 volatile 写的内存语义。

根据 volatile 的 happens- before 规则，释放锁的线程在写 volatile 变量之前可见的共享变量，在获取锁的线程读取同一个 volatile 变量后将立即变得对获取锁的线程可见。

### final

对于 final 域，编译器和处理器要遵守两个重排序规则：

1. 在构造函数内，编译器不能将 `final` 域的赋值操作重排序到构造函数之外。
2. 在构造函数外，读取包含 `final` 域的对象引用和读取该对象的 `final` 域之间不能重排序。这意味着当一个线程获得一个对象引用时，它能够立即读取到 `final` 域的值。

这两个重排序规则确保：

1. 构造函数在初始化 `final` 域后才发布对象引用，避免线程看到未初始化的 `final` 值。
2. 线程在获得对象引用后，能够立即看到 `final` 域的正确值，避免由于重排序导致读取到错误的 `final` 值。

### happens-before

happens-before 原则的诞生是为了程序员和编译器、处理器之间的平衡。只要不改变程序的执行结果（单线程程序和正确执行的多线程程序），编译器和处理器怎么进行重排序优化都行。对于会改变程序执行结果的重排序，JMM 要求编译器和处理器必须禁止这种重排序。

如果 A happens-before B，那么 Java 内存模型将向程序员保证——A 操作的结果将对 B 可见，且 A 的执行顺序排在 B 之前。

happens-before规则：

1. 程序顺序原则：指在一个线程中，按照程序顺序，前面的操作对后续的任意操作可见。
2. 监视器锁规则：解锁于后续对这个锁的加锁可见。
3. volatile 变量规则：对一个 volatile 变量的写操作相对于后续对这个 volatile 变量的读操作可见。
4. 传递性规则：指如果 A Happens-Before B，且 B Happens-Before C，那么 A Happens-Before C。
5. 线程 start() 规则：指主线程 A 启动子线程 B 后，子线程 B 能够看到主线程A在启动子线程 B 前的操作。
6. 线程 join() 规则：指主线程 A 等待子线程 B 完成（主线程 A 通过调用子线程 B 的 join() 方法），当子线程 B 完成后（主线程 A 中 join() 方法返回），主线程能够看到子线程B的操作。



### 双检锁单例模式

```java
public class Singleton {
    private volatile static Singleton uniqueInstance;

    private Singleton() {
    }

    public  static Singleton getUniqueInstance() {
       //先判断对象是否已经实例过，没有实例化过才进入加锁代码
        if (uniqueInstance == null) {
            //类对象加锁
            synchronized (Singleton.class) {
                if (uniqueInstance == null) {
                    uniqueInstance = new Singleton();
                }
            }
        }
        return uniqueInstance;
    }
}
```

采用 `volatile` 关键字修饰也是很有必要的，`uniqueInstance = new Singleton();` 这段代码其实是分为三步执行：

1. 为 `uniqueInstance` 分配内存空间
2. 初始化 `uniqueInstance`
3. 将 `uniqueInstance` 指向分配的内存地址

但是由于 JVM 具有指令重排的特性，执行顺序有可能变成 1->3->2。指令重排在单线程环境下不会出现问题，但是在多线程环境下会导致一个线程获得还没有初始化的实例。例如，线程 T1 执行了 1 和 3，此时 T2 调用 `getUniqueInstance`() 后发现 `uniqueInstance` 不为空，因此返回 `uniqueInstance`，但此时 `uniqueInstance` 还未被初始化。



## Java线程

### 创建和运行线程

**方法一：继承Thread**

```java
// 创建线程对象
Thread t = new Thread() {
    public void run() {
        // 要执行的任务
    }
};
// 启动线程
t.start();
```

**方法二：实现Runnable 接口**

```java
Runnable runnable = new Runnable() {
    public void run(){
        // 要执行的任务
    }
};
// 创建线程对象
Thread t = new Thread( runnable );
// 启动线程
t.start();
```

- 方法1 是把线程和任务合并在了一起，方法2 是把线程和任务分开了 
- 用 Runnable 更容易与线程池等高级 API 配合 
- 用 Runnable 让任务类脱离了 Thread 继承体系，更灵活

**方法三：实现Callable接口与FutureTask** 

java.util.concurrent.Callable接口类似于Runnable，但Callable的call()方法可以有返回值并且可以抛出异常。要执行Callable任务，需将它包装进一个FutureTask，因为Thread类的构造器只接受Runnable参数，而FutureTask实现了Runnable接口。

```java
// 创建任务对象
FutureTask<Integer> task3 = new FutureTask<>(() -> {
    log.debug("hello");
    return 100;
});

// 参数1 是任务对象; 参数2 是线程名字，推荐
new Thread(task3, "t3").start();

// 主线程阻塞，同步等待 task 执行完毕的结果
Integer result = task3.get();
log.debug("结果是:{}", result);
```

**方法四：使用线程池**





### 查看进程线程方法

**windows** 

- 任务管理器可以查看进程和线程数，也可以用来杀死进程 
- `tasklist` 查看进程 
- `taskkill` 杀死进程 



**linux** 

- `ps -fe` 查看所有进程 
- `ps -fT -p <PID>` 查看某个进程（PID）的所有线程 
- `kill`杀死进程 
- `top` 按大写 H 切换是否显示线程 
- `top -H -p <PID>` 查看某个进程（PID）的所有线程 



**Java** 

- `jps` 命令查看所有 Java 进程 
- `jstack <PID>` 查看某个 Java 进程（PID）的所有线程状态 
- `jconsole` 来查看某个 Java 进程中线程的运行情况（图形界面）



jconsole 远程监控配置 

- 需要以如下方式运行你的 java 类

```plain
java -Djava.rmi.server.hostname=`ip地址` -Dcom.sun.management.jmxremote -
Dcom.sun.management.jmxremote.port=`连接端口` -Dcom.sun.management.jmxremote.ssl=是否安全连接 -
Dcom.sun.management.jmxremote.authenticate=是否认证 java类
```

- 修改 /etc/hosts 文件将 127.0.0.1 映射至主机名

如果要认证访问，还需要做如下步骤 

- 复制 jmxremote.password 文件 
- 修改 jmxremote.password 和 jmxremote.access 文件的权限为 600 即文件所有者可读写 
- 连接时填入 controlRole（用户名），R&D（密码）



### 线程上下文切换（Thread Context Switch） 

因为以下一些原因导致 cpu 不再执行当前的线程，转而执行另一个线程的代码 

- 线程的 cpu 时间片用完 
- 垃圾回收 
- 有更高优先级的线程需要运行 
- 线程自己调用了 sleep、yield、wait、join、park、synchronized、lock 等方法 主动让出CPU



当 Context Switch 发生时，需要由操作系统保存当前线程的状态，并恢复另一个线程的状态，Java 中对应的概念就是程序计数器（Program Counter Register），它的作用是记住下一条 jvm 指令的执行地址，是线程私有的 

- 状态包括程序计数器、虚拟机栈中每个栈帧的信息，如局部变量、操作数栈、返回地址等 
- Context Switch 频繁发生会影响性能



### 线程方法

Thread 类 API：

| 方法              | 说明                                                         | 注意                                                         |
| ----------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| start()           | 启动一个新线程，Java虚拟机调用此线程的 run 方法              | start方法只是让线程进入就绪，里面的代码不一定立刻运行。每个线程的start方法只能调用一次。 |
| run()             | 线程启动后调用该方法                                         | 如果构造Thread对象时传递了Runnable参数，线程启动时会调用Runnable中的run方法，否则默认不执行任何操作。但可以创建Thread子类对象重写run方法 |
| sleep(long time)  | 让当前线程休眠多少毫秒，休眠时让出CPU的时间片给其他线程      |                                                              |
| yield()           | 提示线程调度器让出当前线程对 CPU 的使用                      |                                                              |
| interrupt()       | 打断这个线程，异常处理机制                                   | 正在sleep、wait、join会抛出异常并清除打断标记；如果打断正在运行的线程则会设置打断标记 |
| interrupted()     | 判断当前线程是否被打断                                       | 清除打断标记                                                 |
| isInterrupted()   | 判断当前线程是否被打断                                       | 不清除打断标记                                               |
| join()            | 等待线程结束                                                 |                                                              |
| join(long millis) | 等待这个线程结束，最多 millis 毫秒，0 意味着永远等待         |                                                              |
| wait()            | 当前线程进入等待状态，直到被 `notify()` 或 `notifyAll()` 唤醒。必须在同步块或同步方法中调用。 |                                                              |
| notify()          | 醒一个正在等待该对象监视器的线程。被唤醒的线程会进入 **Runnable** 状态，但不会立即获得锁。 |                                                              |
| notifyAll()       | 唤醒所有正在等待该对象监视器的线程。                         |                                                              |

### **run和start**

- 直接调用 run 是在主线程中执行了 run，没有启动新的线程 
- 使用 start 是启动新的线程，通过新的线程间接执行 run 中的代码

### **sleep和yield**

sleep

- 调用 sleep 会让当前线程从 *Running*进入 *Timed Waiting* 状态（阻塞） 
- 其它线程可以使用 interrupt 方法打断正在睡眠的线程，这时 sleep 方法会抛出 InterruptedException 

* 睡眠结束后的线程未必会立刻得到执行 

* 建议用 TimeUnit 的 sleep 代替 Thread 的 sleep 来获得更好的可读性 

yield 

- 调用 yield 会让当前线程从 *Running* 进入 *Runnable* 就绪状态，然后调度执行其它线程 
- 具体的实现依赖于操作系统的任务调度器 



### 限制CPU使用

在没有利用 cpu 来计算时，不要让 while(true) 空转浪费 cpu，这时可以使用 yield 或 sleep 来让出 cpu的使用权给其他程序

**sleep实现**

```java
while(true) {
    try {
        Thread.sleep(50);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

- 可以用 wait 或 条件变量达到类似的效果 
- 不同的是，后两种都需要加锁，并且需要相应的唤醒操作，一般适用于要进行同步的场景 
- sleep 适用于无需锁同步的场景 

**wait实现**

```java
synchronized(锁对象) {
    while(条件不满足) {
        try {
            锁对象.wait();
        } catch(InterruptedException e) {
            e.printStackTrace();
        }
    }
    // do sth...
}
```

**条件变量实现**

```java
lock.lock();
try {
    while(条件不满足) {
        try {
            条件变量.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    // do sth...
} finally {
    lock.unlock();
}
```



### 停止线程

**异常法停止**：线程调用interrupt()方法后，在线程的run方法中判断当前对象的interrupted()状态，如果是中断状态则抛出异常，达到中断线程的效果。

**在沉睡中停止**：先将线程sleep，然后调用interrupt标记中断状态，interrupt会将阻塞状态的线程中断。会抛出中断异常，达到停止线程的效果。

**stop()暴力停止**：强制让线程停止有可能使一些请理性的工作得不到完成，如果线程持有 JUC 的互斥锁可能导致锁来不及释放，造成死锁。不推荐

**使用return停止线程**：调用interrupt标记为中断状态后，在run方法中判断当前线程状态，如果为中断状态则return，能达到停止线程的效果。



在一个线程 T1 中如何“优雅”终止线程 T2？

错误思路：

1. 使用线程对象的 stop() 方法停止线程 ：stop 方法会真正杀死线程，如果这时线程锁住了共享资源，那么当它被杀死后就再也没有机会释放锁，其它线程将永远无法获取锁 

2. 使用 System.exit(int) 方法停止线程 ：目的仅是停止一个线程，但这种做法会让整个程序都停止



正确思路：两阶段终止模式

1. 利用 isInterrupted 

   ```java
   public class Test {
       public static void main(String[] args) throws InterruptedException {
           TwoPhaseTermination tpt = new TwoPhaseTermination();
           tpt.start();
           Thread.sleep(3500);
           tpt.stop();
       }
   }
   class TwoPhaseTermination {
       private Thread monitor;
       // 启动监控线程
       public void start() {
           monitor = new Thread(new Runnable() {
               @Override
               public void run() {
                   while (true) {
                       Thread thread = Thread.currentThread();
                       if (thread.isInterrupted()) {
                           System.out.println("后置处理");
                           break;
                       }
                       try {
                           Thread.sleep(1000);					// 睡眠
                           System.out.println("执行监控记录");	// 在此被打断不会异常
                       } catch (InterruptedException e) {		// 在睡眠期间被打断，进入异常处理的逻辑
                           e.printStackTrace();
                           // 重新设置打断标记，打断 sleep 会清除打断状态
                           thread.interrupt();
                       }
                   }
               }
           });
           monitor.start();
       }
       // 停止监控线程
       public void stop() {
           monitor.interrupt();
       }
   }
   ```

2. 利用停止标记

   ```java
   // 停止标记用 volatile 是为了保证该变量在多个线程之间的可见性
   // 我们的例子中，即主线程把它修改为 true 对 t1 线程可见
   class TPTVolatile {
       private Thread thread;
       private volatile boolean stop = false;
       
       public void start(){
           thread = new Thread(() -> {
               while(true) {
                   //Thread current = Thread.currentThread();
                   if(stop) {
                       log.debug("料理后事");
                       break;
                   }
                   try {
                       Thread.sleep(1000);
                       log.debug("将结果保存");
                   } catch (InterruptedException e) {
   
                   }
                   // 执行监控操作
               }
           },"监控线程");
           thread.start();
       }
       
       public void stop() {
           stop = true;
           thread.interrupt();
       }
   }
   ```

   


### 不推荐

不推荐使用的方法，这些方法已过时，容易破坏同步代码块，造成线程死锁：

- `public final void stop()`：停止线程运行

  废弃原因：方法粗暴，除非可能执行 finally 代码块以及释放 synchronized 外，线程将直接被终止，如果线程持有 JUC 的互斥锁可能导致锁来不及释放，造成其他线程永远等待的局面

- `public final void suspend()`：挂起（暂停）线程运行

  废弃原因：如果目标线程在暂停时对系统资源持有锁，则在目标线程恢复之前没有线程可以访问该资源，如果恢复目标线程的线程在调用 resume 之前会尝试访问此共享资源，则会导致死锁

- `public final void resume()`：恢复线程运行



### 线程状态

| 线程状态                   | 导致状态发生条件                                             |
| -------------------------- | ------------------------------------------------------------ |
| NEW（新建）                | 线程刚被创建，但是并未启动，还没调用 start 方法，只有线程对象，没有线程特征 |
| Runnable（可运行）         | 线程可以在 Java 虚拟机中运行的状态，可能正在运行自己代码，也可能没有，这取决于操作系统处理器，调用了 t.start() 方法 |
| Blocked（阻塞）            | 当一个线程试图获取一个对象锁，而该对象锁被其他的线程持有，则该线程进入 Blocked 状态；当该线程持有锁时，该线程将变成 Runnable 状态。 |
| Waiting（无限等待）        | 一个线程在等待另一个线程执行一个（唤醒）动作时，该线程进入 Waiting 状态，进入这个状态后不能自动唤醒，必须等待另一个线程调用 notify 或者 notifyAll 方法才能唤醒。在这种状态下，线程将不会消耗CPU资源 |
| Timed Waiting （限期等待） | 有几个方法有超时参数，调用将进入 Timed Waiting 状态，这一状态将一直保持到超时期满或者接收到唤醒通知。带有超时参数的常用方法有 Thread.sleep 、Object.wait |
| Teminated（终止）          | run 方法正常退出而死亡，或者因为没有捕获的异常终止了 run 方法而死亡 |

<img src=".\JUC\线程状态.png" style="zoom:80%;" />

**NEW → RUNNABLE**：

当调用 t.start() 方法时，由 NEW → RUNNABLE

**RUNNABLE <--> WAITING**：

1. t 线程用 `synchronized(obj)` 获取了对象锁后，

   1. 调用 obj.wait() 方法时： RUNNABLE --> WAITING 

   2. 调用 obj.notify()、obj.notifyAll()、t.interrupt()：

      1. 竞争锁成功，t 线程从 WAITING → RUNNABLE

      2. 竞争锁失败，t 线程从 WAITING → BLOCKED

2. 当前线程调用 t.join() 方法时，**当前线程**从 RUNNABLE --> WAITING ；t 线程运行结束，或调用了当前线程的 interrupt() 时，当前线程从 WAITING --> RUNNABLE

3. 当前线程调用 LockSupport.park() 方法会让**当前线程**从 RUNNABLE --> WAITING ；调用 LockSupport.unpark(目标线程) 或调用了线程 的 interrupt() ，会让目标线程从 WAITING -->RUNNABLE 

**RUNNABLE <--> TIMED_WAITING**：

1. t 线程用 synchronized(obj) 获取了对象锁后 ，

   1. 调用 obj.wait(long n) 方法时，t 线程从 RUNNABLE --> TIMED_WAITING 

   2. t 线程等待时间超过了 n 毫秒，或调用 obj.notify() ， obj.notifyAll() ， t.interrupt() 时 

      1. 竞争锁成功，t 线程从TIMED_WAITING --> RUNNABLE 

      2. 竞争锁失败，t 线程从TIMED_WAITING --> BLOCKED

2. 当前线程调用 t.join(long n) 方法时，**当前线程**从 RUNNABLE --> TIMED_WAITING ; 当前线程等待时间超过了 n 毫秒，或t 线程运行结束，或调用了当前线程的 interrupt() 时，当前线程从 TIMED_WAITING --> RUNNABLE

3. 当前线程调用 Thread.sleep(long n) ，**当前线程**从 RUNNABLE --> TIMED_WAITING ; 当前线程等待时间超过了 n 毫秒，当前线程从TIMED_WAITING --> RUNNABLE

4. 当前线程调用 LockSupport.parkNanos(long nanos) 或 LockSupport.parkUntil(long millis) 时，**当前线程**从 RUNNABLE --> TIMED_WAITING；调用 LockSupport.unpark(目标线程) 或调用了线程 的 interrupt() ，或是等待超时，会让目标线程从 TIMED_WAITING--> RUNNABLE

**RUNNABLE <--> BLOCKED**：

t 线程用 synchronized(obj) 竞争对象锁失败时，从RUNNABLE --> BLOCKED 

持 obj 锁线程的同步代码块执行完毕，会唤醒该对象上所有 BLOCKED的线程重新竞争，如果其中 t 线程竞争成功，从 BLOCKED --> RUNNABLE ，其它失败的线程仍然BLOCKED

**RUNNABLE --> TERMINATED** ：

当前线程所有代码运行完毕，进入 TERMINATED 

## 管程/监视器（Monitor）

所谓**管程，指的是管理共享变量以及对共享变量的操作过程，让他们支持并发**



### 竞态条件 Race Condition 

多个线程在临界区内执行，由于代码的**执行序列不同**而导致结果无法预测，称之为发生了**竞态条件**

为了避免临界区的竞态条件发生，有多种手段可以达到目的： 

- 阻塞式的解决方案：synchronized，Lock 
- 非阻塞式的解决方案：原子变量 



### 变量的线程安全

**成员变量和静态变量是否线程安全？** 

- 如果它们没有共享，则线程安全 
- 如果它们被共享了，根据它们的状态是否能够改变，又分两种情况 
  - 如果只有读操作，则线程安全 
  - 如果有读写操作，则这段代码是临界区，需要考虑线程安全 


**局部变量是否线程安全？** 

- 局部变量是线程安全的 
- 但局部变量引用的对象则未必 
  - 如果该对象没有逃离方法的作用范围，它是线程安全的 
  - 如果该对象逃离方法的作用范围，需要考虑线程安全


```java
public static void test1() {
    int i = 10;
    i++; 
}
```

每个线程调用 test1() 方法时局部变量 i，会在每个线程的栈帧内存中被创建多份，因此不存在共享



### 常见线程安全类

- String 
- Integer 
- StringBuffer 
- Random 
- Vector 
- Hashtable 
- java.util.concurrent 包下的类

String、Integer 等都是不可变类，因为其内部的状态不可以改变，因此它们的方法都是线程安全的



### 案例分析

案例一：

```java
public class MyServlet extends HttpServlet {
    // 是否安全？  不安全 servlet是运行在tomcat下，只有一个实例，所以这些成员变量也只有一份，需要考虑线程安全
    Map<String,Object> map = new HashMap<>();
    // 是否安全？  安全
    String S1 = "...";
    // 是否安全？  安全
    final String S2 = "...";
    // 是否安全？  不安全
    Date D1 = new Date();
    // 是否安全？  不安全 引用不能变，但是属性值是可变的
    final Date D2 = new Date();

    public void doGet(HttpServletRequest request, HttpServletResponse response) {
        // 使用上述变量
    }
}
```

案例二：

```java
public class MyServlet extends HttpServlet {
    // 是否安全？  不安全 userService也只有一份
    private UserService userService = new UserServiceImpl();

    public void doGet(HttpServletRequest request, HttpServletResponse response) {
        userService.update(...);
    }
}

public class UserServiceImpl implements UserService {
    // 记录调用次数
    private int count = 0;

    public void update() {
        // ...
        count++;
    }
}
```

案例三：

```java
@Aspect
@Component
public class MyAspect {
    // 是否安全？ 不安全 Spring的对象默认是单例的，所以成员变量是共享的，存在线程安全问题。
    // 使用环绕通知将其变成局部变量解决线程安全问题。
    private long start = 0L;

    @Before("execution(* *(..))")
    public void before() {
        start = System.nanoTime();
    }

    @After("execution(* *(..))")
    public void after() {
        long end = System.nanoTime();
        System.out.println("cost time:" + (end-start));
    }
}
```

案例四：

```java
public class MyServlet extends HttpServlet {
    // 是否安全  安全
    private UserService userService = new UserServiceImpl();

    public void doGet(HttpServletRequest request, HttpServletResponse response) {
        userService.update(...);
    }
}

public class UserServiceImpl implements UserService {
    // 是否安全 安全
    private UserDao userDao = new UserDaoImpl();

    public void update() {
        userDao.update();
    }
}

public class UserDaoImpl implements UserDao {
    public void update() {
        String sql = "update user set password = ? where username = ?";
        // 是否安全 安全
        try (Connection conn = DriverManager.getConnection("","","")){
            // ...
        } catch (Exception e) {
            // ...
        }
    }
}
```

案例五：

```java
public class MyServlet extends HttpServlet {
    // 是否安全 不安全
    private UserService userService = new UserServiceImpl();

    public void doGet(HttpServletRequest request, HttpServletResponse response) {
        userService.update(...);
    }
}

public class UserServiceImpl implements UserService {
    // 是否安全 不安全
    private UserDao userDao = new UserDaoImpl();

    public void update() {
        userDao.update();
    }
}

public class UserDaoImpl implements UserDao {
    // 是否安全 不安全 这里Connection放在了成员变量中，由于UserDaoImpl是独一份的，所以Connection是可能被多个线程修改的，所以是不安全的
    private Connection conn = null;
    public void update() throws SQLException {
        String sql = "update user set password = ? where username = ?";
        conn = DriverManager.getConnection("","","");
        // ...
        conn.close();
    }
}
```

案例六：

```java
public class MyServlet extends HttpServlet {
    // 是否安全 安全
    private UserService userService = new UserServiceImpl();

    public void doGet(HttpServletRequest request, HttpServletResponse response) {
        userService.update(...);
    }
}

public class UserServiceImpl implements UserService {
    public void update() {
        UserDao userDao = new UserDaoImpl();
        userDao.update();
    }
}

public class UserDaoImpl implements UserDao {
    // 是否安全 安全
    private Connection = null;
    public void update() throws SQLException {
        String sql = "update user set password = ? where username = ?";
        conn = DriverManager.getConnection("","","");
        // ...
        conn.close();
    }
}
```



### Monitor

**Java对象头**

普通对象：

![](JUC\普通对象头.png)

Mark Word 主要用来存储对象自身的运行时数据

Klass Word 指向Class对象

数组对象：

![](JUC\数组对象头.png)

**Mark Word 结构**

![](JUC\mark_word结构.png)

**Monitor 结构**

Monitor 被翻译为监视器或管程

每个 Java 对象都可以关联一个 Monitor 对象，如果使用 synchronized 给对象上锁（重量级）之后，该对象头的Mark Word 中就被设置指向 Monitor 对象的指针



![](JUC\monitor.png)

- 刚开始 Monitor 中 Owner 为 null 
- 当 Thread-2 执行 synchronized(obj) 就会将 Monitor 的所有者 Owner 置为 Thread-2，Monitor中只能有一个 Owner 
- 在 Thread-2 上锁的过程中，如果 Thread-3，Thread-4，Thread-5 也来执行 synchronized(obj)，就会进入EntryList BLOCKED 
- Thread-2 执行完同步代码块的内容，然后唤醒 EntryList 中等待的线程来竞争锁，竞争的时是非公平的 
- 图中 WaitSet 中的 Thread-0，Thread-1 是之前获得过锁，但条件不满足进入 WAITING 状态的线程（wait-notify 机制）

**注意：** 

- synchronized 必须是进入同一个对象的 monitor 才有上述的效果 
- 不加 synchronized 的对象不会关联监视器，不遵从以上规则



### **synchronized**

**对象锁，保证了临界区内代码的原子性**，采用互斥的方式让同一时刻至多只有一个线程能持有对象锁，其它线程获取这个对象锁时会阻塞，保证拥有锁的线程可以安全的执行临界区内的代码，不用担心线程上下文切换

synchronized 实际是用**对象锁**保证了**临界区内代码的原子性**，临界区内的代码对外是不可分割的，不会被线程切换所打断。

互斥和同步都可以采用 synchronized 关键字来完成，区别：

- 互斥是保证临界区的竞态条件发生，同一时刻只能有一个线程执行临界区代码
- 同步是由于线程执行的先后、顺序不同、需要一个线程等待其它线程运行到某个点

语法：

```java
synchronized(对象) // 线程1， 线程2(blocked)
{
 	临界区
}
```

**方法上的synchronized**

```java
// 锁住 this 对象，只有在当前实例对象的线程内是安全的，如果有多个实例就不安全
class Test{
    public synchronized void test() {

    }
}
等价于
class Test{
    public void test() {
        synchronized(this) {

        }
    }
}
```

```java
// 锁住类对象，所有类的实例的方法都是安全的，类的所有实例都相当于同一把锁
class Test{
    public synchronized static void test() {
        
    }
}
等价于
class Test{
    public static void test() {
        synchronized(Test.class) {

        }
    }
}
```



### synchronized 原理

synchronized 的原理其实就是基于一个锁对象和锁对象相关联的一个 **monitor 对象**。

synchronized 同步语句块的情况：

使用synchronized之后，会在编译之后在同步的代码块前后加上monitorenter和monitorexit字节码指令，执行monitorenter指令时会尝试获取对象锁，如果对象没有被锁定或者已经获得了锁，锁的计数器+1。执行monitorexit指令时则会把计数器-1，当计数器值为0时，则锁释放，处于等待队列中的线程再继续竞争锁。

synchronized 修饰方法的的情况：

`synchronized` 修饰的方法并没有 `monitorenter` 指令和 `monitorexit` 指令，取得代之的确实是 `ACC_SYNCHRONIZED` 标识，该标识指明了该方法是一个同步方法。JVM 通过该 `ACC_SYNCHRONIZED` 访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。



![](JUC\synchronized.webp)

1. 当多个线程进入同步代码块时，首先进入entryList
2. 有一个线程获取到monitor锁后，就赋值给当前线程，并且计数器+1
3. 如果线程调用wait方法，将释放锁，当前线程置为null，计数器-1，同时进入waitSet等待被唤醒，调用notify或者notifyAll之后又会进入entryList竞争锁
4. 如果线程执行完毕，同样释放锁，计数器-1，当前线程置为null





#### JVM对Synchornized的优化

锁膨胀

锁消除

锁粗话

自适应自旋锁



#### 轻量级锁

1. 创建 锁记录（Lock Record）对象，每个线程的栈帧都会包含一个锁记录的结构，内部可以存储锁定对象的Mark Word 

2. 让锁记录中 Object reference 指向锁对象，并尝试用 cas 替换 Object 的 Mark Word，将 Mark Word 的值存入锁记录

   1. 如果 cas 替换成功，对象头中存储了 `锁记录地址和状态 00 `，表示由该线程给对象加锁，这时图示如下
   2. 如果 cas 失败，有两种情况 
      1. 如果是其它线程已经持有了该 Object 的轻量级锁，这时表明有竞争，进入锁膨胀过程 
      2. 如果是自己执行了 synchronized 锁重入，那么再添加一条 Lock Record 作为重入的计数

3. 当退出 synchronized 代码块（解锁时）如果有取值为 null 的锁记录，表示有重入，这时重置锁记录，表示重入计数减一

4. 当退出 synchronized 代码块（解锁时）锁记录的值不为 null，这时使用 cas 将 Mark Word 的值恢复给对象头 
   1. 成功，则解锁成功 

   2. 失败，说明轻量级锁进行了锁膨胀或已经升级为重量级锁，进入重量级锁解锁流程


| <img src="E:\笔记\notes\JUC\轻量级锁2.png" style="zoom: 67%;" /> | <img src="E:\笔记\notes\JUC\轻量级锁3.png" style="zoom: 67%;" /> |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| <img src=".\JUC\轻量级锁4.png" style="zoom:67%;" />          | <img src=".\JUC\轻量级锁5.png" style="zoom:67%;" />          |



#### 锁膨胀

如果在尝试加轻量级锁的过程中，CAS 操作无法成功，这时一种情况就是有其它线程为此对象加上了轻量级锁（有竞争），这时需要进行锁膨胀，将轻量级锁变为重量级锁。

1. 当 Thread-1 进行轻量级加锁时，Thread-0 已经对该对象加了轻量级锁
2. Thread-1 加轻量级锁失败，进入锁膨胀流程：为 Object 对象申请 Monitor 锁，**通过 Object 对象头获取到持锁线程**，将 Monitor 的 Owner 置为 Thread-0，将 Object 的对象头指向重量级锁地址，然后自己进入 Monitor 的 EntryList BLOCKED
3. 当 Thread-0 退出同步块解锁时，使用 CAS 将 Mark Word 的值恢复给对象头失败，这时进入重量级解锁流程，即按照 Monitor 地址找到 Monitor 对象，设置 Owner 为 null，唤醒 EntryList 中 BLOCKED 线程

| ![](.\JUC\锁膨胀0.png) | ![](.\JUC\锁膨胀.png) |
| ---------------------- | --------------------- |



#### 自旋优化

**重量级锁**竞争时，尝试获取锁的线程不会立即阻塞，可以使用**自旋**（默认 10 次）来进行优化，采用循环的方式去尝试获取锁

注意：

- 自旋占用 CPU 时间，单核 CPU 自旋就是浪费时间，因为同一时刻只能运行一个线程，多核 CPU 自旋才能发挥优势
- 自旋失败的线程会进入阻塞状态

优点：不会进入阻塞状态，**减少线程上下文切换的消耗**

缺点：当自旋的线程越来越多时，会不断的消耗 CPU 资源

自旋锁说明：

- 在 Java 6 之后自旋锁是自适应的，比如对象刚刚的一次自旋操作成功过，那么认为这次自旋成功的可能性会高，就多自旋几次；反之，就少自旋甚至不自旋，比较智能
- Java 7 之后不能控制是否开启自旋功能，由 JVM 控制



#### 偏向锁



**撤销偏向锁： **

**1. 调用对象 hashCode** 

调用了对象的 hashCode，但偏向锁的对象 MarkWord 中存储的是线程 id，如果调用 hashCode 无法存储hashCode，会导致偏向锁被撤销 

- 轻量级锁会在锁记录中记录 hashCode 
- 重量级锁会在 Monitor 中记录 hashCode 

**2. 没有锁竞争下，其它线程使用对象** 

当有其它线程使用偏向锁对象时，会将偏向锁升级为轻量级锁

```java

private static void test2() throws InterruptedException {
    
    Dog d = new Dog();
    
    Thread t1 = new Thread(() -> {
        
        log.debug(ClassLayout.parseInstance(d).toPrintableSimple(true));
        synchronized (d) {
            log.debug(ClassLayout.parseInstance(d).toPrintableSimple(true));
        }
        log.debug(ClassLayout.parseInstance(d).toPrintableSimple(true));
        
        synchronized (TestBiased.class) {
            TestBiased.class.notify();
        }
    }, "t1");
    t1.start();
    
    Thread t2 = new Thread(() -> {
        synchronized (TestBiased.class) {
            try {
                TestBiased.class.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        
        log.debug(ClassLayout.parseInstance(d).toPrintableSimple(true));
        synchronized (d) {
            log.debug(ClassLayout.parseInstance(d).toPrintableSimple(true));
        }
        log.debug(ClassLayout.parseInstance(d).toPrintableSimple(true));
        
    }, "t2");
    t2.start();
}
```

**3. 调用 wait/notify**

重量级锁才支持 wait/notify



**批量撤销**：

如果对象被多个线程访问，但没有竞争，这时偏向了线程 T1 的对象仍有机会重新偏向 T2，重偏向会重置对象的 Thread ID

- 批量重偏向：当撤销偏向锁阈值超过 20 次后，JVM 会觉得是不是偏向错了，于是在给这些对象加锁时重新偏向至加锁线程
- 批量撤销：当撤销偏向锁阈值超过 40 次后，JVM 会觉得自己确实偏向错了，根本就不该偏向，于是整个类的所有对象都会变为不可偏向的，新建的对象也是不可偏向的

```java
private static void test3() throws InterruptedException {
    
    Vector<Dog> list = new Vector<>();
    
    Thread t1 = new Thread(() -> {
        for (int i = 0; i < 30; i++) {
            Dog d = new Dog();
            list.add(d);
            synchronized (d) {
                log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintableSimple(true));
            }
        }
        synchronized (list) {
            list.notify();
        }
    }, "t1");
    t1.start();

    Thread t2 = new Thread(() -> {
        synchronized (list) {
            try {
                list.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        log.debug("===============> ");
        for (int i = 0; i < 30; i++) {
            Dog d = list.get(i);
            log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintableSimple(true));
            synchronized (d) {
                log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintableSimple(true));
            }
            log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintableSimple(true));
        }
    }, "t2");
    t2.start();
}

```



#### 锁消除

锁消除是指对于被检测出不可能存在竞争的共享数据的锁进行消除，这是 JVM **即时编译器的优化**

锁消除主要是通过**逃逸分析**来支持，如果堆上的共享数据不可能逃逸出去被其它线程访问到，那么就可以把它们当成私有数据对待，也就可以将它们的锁进行消除



#### 锁粗化

对相同对象多次加锁，导致线程发生多次重入，频繁的加锁操作就会导致性能损耗，可以使用锁粗化方式优化

如果虚拟机探测到一串的操作都对同一个对象加锁，将会把加锁的范围扩展（粗化）到整个操作序列的外部



### wait / notify

![](.\JUC\wait-notify.png)

- Owner 线程发现条件不满足，调用 wait 方法，即可进入 WaitSet 变为 WAITING 状态 
- BLOCKED 和 WAITING 的线程都处于阻塞状态，不占用 CPU 时间片 
- BLOCKED 线程会在 Owner 线程释放锁时唤醒 
- WAITING 线程会在 Owner 线程调用 notify 或 notifyAll 时唤醒，但唤醒后并不意味者立刻获得锁，仍需进入EntryList 重新竞争

```java
public final void notify():唤醒正在等待对象监视器的单个线程。
public final void notifyAll():唤醒正在等待对象监视器的所有线程。
public final void wait():导致当前线程等待，直到另一个线程调用该对象的 notify() 方法或 notifyAll()方法。
public final native void wait(long timeout):有时限的等待, 到n毫秒后结束等待，或是被唤醒
```

它们都是线程之间进行协作的手段，都属于 Object 对象的方法。**必须先获得此对象的锁，才能调用这几个方法**



**`sleep(long n)` 和 `wait(long n)` 的区别**：

1.  sleep 是 Thread 方法，而 wait 是 Object 的方法 

2.  sleep 不需要强制和 synchronized 配合使用，但 wait 需要和 synchronized 一起用 

3.  **`sleep()` 方法没有释放锁，而 `wait()` 方法释放了锁** ，但都会释放CPU

4.  它们状态 TIMED_WAITING

5.  `wait()` 通常被用于线程间交互/通信，`sleep()`通常被用于暂停执行。

6.  `wait()` 方法被调用后，线程不会自动苏醒，需要别的线程调用同一个对象上的 `notify()`或者 `notifyAll()` 方法。`sleep()`方法执行完成后，线程会自动苏醒，或者也可以使用 `wait(long timeout)` 超时后线程会自动苏醒。



```java
synchronized(lock) {
    while(条件不成立) {
        lock.wait();
    }
    // 干活
}

//另一个线程
synchronized(lock) {
    lock.notifyAll();
}
```



### Park & Unpark 

 LockSupport 类中的方法：每个线程都有自己的一个(C代码实现的) Parker 对象

```java
// 暂停当前线程
LockSupport.park(); 
// 恢复某个线程的运行
LockSupport.unpark(暂停线程对象)
```



```java
public static void main(String[] args) {
    Thread t1 = new Thread(() -> {
        System.out.println("start...");	//1
		Thread.sleep(1000);// Thread.sleep(3000)
        // 先 park 再 unpark 和先 unpark 再 park 效果一样，都会直接恢复线程的运行
        System.out.println("park...");	//2
        LockSupport.park();
        System.out.println("resume...");//4
    },"t1");
    t1.start();
   	Thread.sleep(2000);
    System.out.println("unpark...");	//3
    LockSupport.unpark(t1);
}
```

先调用park，后调用unpark：

| ![](.\JUC\park.png)                                          | ![](.\JUC\unpark.png)                                        |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 1. 当前线程调用 Unsafe.park() 方法 <br>2. 检查 _counter ，本情况为 0，这时，获得 _mutex 互斥锁 <br>3.  线程进入 _cond 条件变量阻塞 <br>4. 设置 _counter = 0 | 1. 调用 Unsafe.unpark(Thread_0) 方法，设置 _counter 为 1 <br>2. 唤醒 _cond 条件变量中的 Thread_0 <br>3. Thread_0 恢复运行 <br>4. 设置 _counter 为 0 |

先调用unpark 再调用park：

1. 调用 Unsafe.unpark(Thread_0) 方法，设置 _counter 为 1 

2. 当前线程调用 Unsafe.park() 方法 

3. 检查 _counter ，本情况为 1，这时线程无需阻塞，继续运行 

4. 设置 _counter 为 0 

![](.\JUC\unpark-park.png)



与 Object 的 wait & notify 相比 ：

- wait，notify 和 notifyAll 必须配合 Object Monitor 一起使用，而 park，unpark 不必
- park & unpark 是以线程为单位来【阻塞】和【唤醒(指定)】线程，而 notify 只能随机唤醒一个等待线程，notifyAll是唤醒所有等待线程
- park & unpark 可以先 unpark，而 wait & notify 不能先 notify 
- wait 会释放锁资源进入WAITING队列，**park 不会释放锁资源**，只负责阻塞当前线程，会释放 CPU



### ReentrantLock

`ReentrantLock` 实现了 `Lock` 接口，是一个可重入且独占式的锁，和 `synchronized` 关键字类似。不过，`ReentrantLock` 更灵活、更强大，增加了轮询、超时、中断、公平锁和非公平锁等高级功能。



ReentrantLock 相对于 synchronized 具备如下特点：

1. 锁的实现：synchronized 是 JVM 实现的，而 ReentrantLock 是 JDK 实现的
2. 性能：新版本 Java 对 synchronized 进行了很多优化，synchronized 与 ReentrantLock 大致相同
3. 使用：ReentrantLock 需要手动解锁，synchronized 执行完代码块自动解锁
4. **可中断**：ReentrantLock 可中断，而 synchronized 不行。这里的可中断是指在等待锁的过程中，可以被中断（即取消获取锁的请求）
5. **公平锁**：公平锁是指多个线程在等待同一个锁时，必须按照申请锁的时间顺序来依次获得锁
   - ReentrantLock 可以设置公平锁，synchronized 中的锁是非公平的
   - 不公平锁的含义是阻塞队列内公平，队列外非公平
6. **锁超时**：尝试获取锁，超时获取不到直接放弃，不进入阻塞队列
   - ReentrantLock 可以设置超时时间，synchronized 会一直等待
7. 锁绑定多个条件：一个 ReentrantLock 可以同时绑定多个 Condition 对象，更细粒度的唤醒线程
8. 两者都是可重入锁

```java
// 获取锁
reentrantLock.lock();
try {
    // 临界区
} finally {
    // 释放锁
    reentrantLock.unlock();
}
```



#### 可重入

可重入是指同一个线程如果首次获得了这把锁，那么它是这把锁的拥有者，因此有权利再次获取这把锁，如果不可重入锁，那么第二次获得锁时，自己也会被锁挡住，直接造成死锁。

原理：通过state计数的增加减少来实现可重入

```java
static final class NonfairSync extends Sync {
    // ...
    
    // Sync 继承过来的方法, 方便阅读, 放在此处
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 如果已经获得了锁, 线程还是当前线程, 表示发生了锁重入
        else if (current == getExclusiveOwnerThread()) {
            // state++
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
    
    // Sync 继承过来的方法, 方便阅读, 放在此处
    protected final boolean tryRelease(int releases) {
        // state-- 
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        // 支持锁重入, 只有 state 减为 0, 才释放成功
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }
}
```



#### 可打断

打断即取消获取锁的请求

`public void lockInterruptibly()`：获得可打断的锁

- 如果没有竞争此方法就会获取 lock 对象锁
- 如果有竞争就进入阻塞队列，可以被其他线程用 interrupt 打断
- 如果是不可中断模式，那么即使使用了 interrupt 也不会让等待中断

```java
ReentrantLock lock = new ReentrantLock();

Thread t1 = new Thread(() -> {
    log.debug("启动...");
    
    try {
        //没有竞争就会获取锁
        //有竞争就进入阻塞队列等待,但可以被打断
        lock.lockInterruptibly();
        //lock.lock(); //不可打断 
    } catch (InterruptedException e) {
        e.printStackTrace();
        log.debug("等锁的过程中被打断");
        return;
    }
    
    try {
        log.debug("获得了锁");
    } finally {
        lock.unlock();
    }
}, "t1");

lock.lock();
log.debug("获得了锁");
t1.start();

try {
    sleep(1);
    log.debug("执行打断");
    t1.interrupt();
} finally {
    lock.unlock();
}

/*
18:02:40.520 [main] c.TestInterrupt - 获得了锁
18:02:40.524 [t1] c.TestInterrupt - 启动... 
18:02:41.530 [main] c.TestInterrupt - 执行打断
java.lang.InterruptedException 
at 
java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireInterruptibly(AbstractQueuedSynchr
onizer.java:898) 
    at 
    java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireInterruptibly(AbstractQueuedSynchron
    izer.java:1222) 
    at java.util.concurrent.locks.ReentrantLock.lockInterruptibly(ReentrantLock.java:335) 
    at cn.itcast.n4.reentrant.TestInterrupt.lambda$main$0(TestInterrupt.java:17) 
    at java.lang.Thread.run(Thread.java:748) 
18:02:41.532 [t1] c.TestInterrupt - 等锁的过程中被打断
/*
```

#### 锁超时

`public boolean tryLock()`：尝试获取锁，获取到返回 true，获取不到直接放弃，不进入阻塞队列

`public boolean tryLock(long timeout, TimeUnit unit)`：在给定时间内获取锁，获取不到就退出

注意：tryLock 期间也可以被打断



#### 公平锁

ReentrantLock 默认是不公平的

公平锁一般没有必要，会降低并发度

如果是公平锁，唤醒的策略就是谁等待的时间长，就唤醒谁，很公平；如果是非公平锁，则不提供这个公平保证，有可能等待时间短的线程反而先被唤醒。



#### 条件变量

synchronized 中也有条件变量，就是我们讲原理时那个 waitSet 休息室，当条件不满足时进入 waitSet 等待 

ReentrantLock 的条件变量比 synchronized 强大之处在于，它是支持多个条件变量的，这就好比 

- synchronized 是那些不满足条件的线程都在一间休息室等消息 
- 而 ReentrantLock 支持多间休息室，有专门等烟的休息室、专门等早餐的休息室、唤醒时也是按休息室来唤醒 



ReentrantLock 类获取 Condition 对象：`public Condition newCondition()`

Condition 类 API：

- `void await()`：当前线程从运行状态进入等待状态，释放锁
- `void signal()`：唤醒一个等待在 Condition 上的线程，但是必须获得与该 Condition 相关的锁



使用要点： 

- **await / signal 前需要获得锁**
- await 执行后，会释放锁，进入 conditionObject 等待 
- await 的线程被唤醒（或打断、或超时）去重新竞争 lock 锁 
- 竞争 lock 锁成功后，从 await 后继续执行 

```java
static ReentrantLock lock = new ReentrantLock();

static Condition waitCigaretteQueue = lock.newCondition();
static Condition waitbreakfastQueue = lock.newCondition();

static volatile boolean hasCigrette = false;
static volatile boolean hasBreakfast = false;

public static void main(String[] args) {
    
    new Thread(() -> {
        try {
            lock.lock();
            while (!hasCigrette) {
                try {
                    waitCigaretteQueue.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            log.debug("等到了它的烟");
        } finally {
            lock.unlock();
        }
    }).start();
    
    new Thread(() -> {
        try {
            lock.lock();
            while (!hasBreakfast) {
                try {
                    waitbreakfastQueue.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            log.debug("等到了它的早餐");
        } finally {
            lock.unlock();
        }
    }).start();
    
    sleep(1);
    sendBreakfast();
    sleep(1);
    sendCigarette();
}

private static void sendCigarette() {
    lock.lock();
    try {
        log.debug("送烟来了");
        hasCigrette = true;
        waitCigaretteQueue.signal();
    } finally {
        lock.unlock();
    }
}

private static void sendBreakfast() {
    lock.lock();
    try {
        log.debug("送早餐来了");
        hasBreakfast = true;
        waitbreakfastQueue.signal();
    } finally {
        lock.unlock();
    }
}
```





## 无锁

### CAS

CAS 的全称是 Compare-And-Swap，是 CPU 并发原语，作为一条 CPU 指令，CAS 指令本身是能够保证原子性的，线程安全。

作用：比较当前工作内存中的值和主物理内存中的值，如果相同则执行指定操作，否则继续比较直到主内存和工作内存的值一致为止

CAS 特点：

- CAS 体现的是**无锁并发、无阻塞并发**，线程不会陷入阻塞，线程不需要频繁切换状态（上下文切换，系统调用）
- CAS 是基于乐观锁的思想

CAS 缺点：

- 执行的是循环操作，如果比较不成功一直在循环，最差的情况某个线程一直取到的值和预期值都不一样，就会无限循环导致饥饿，**使用 CAS 线程数不要超过 CPU 的核心数**，采用分段 CAS 和自动迁移机制
- 只能保证一个共享变量的原子操作
  - 对于一个共享变量执行操作时，可以通过循环 CAS 的方式来保证原子操作
  - 对于多个共享变量操作时，循环 CAS 就无法保证操作的原子性，这个时候**只能用锁来保证原子性**
-  ABA 问题：主线程仅能判断出共享变量的值与最初值 A 是否相同，不能感知到这种从 A 改为 B 又 改回 A 的情况，使用AtomicStampedReference解决

CAS 与 synchronized 总结：

- synchronized 是从悲观的角度出发：总是假设最坏的情况，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会阻塞（共享资源每次只给一个线程使用，其它线程阻塞，用完后再把资源转让给其它线程），因此 synchronized 也称之为悲观锁，ReentrantLock 也是一种悲观锁，性能较差
- CAS 是从乐观的角度出发：总是假设最好的情况，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据。**如果别人修改过，则获取现在最新的值，如果别人没修改过，直接修改共享数据的值**，CAS 这种机制也称之为乐观锁，综合性能较好

注意：**CAS 必须借助 volatile 才能读取到共享变量的最新值来实现【比较并交换】的效果**

CAS 底层实现是在一个循环中不断地尝试修改目标值，直到修改成功。如果竞争不激烈修改成功率很高，否则失败率很高，失败后这些重复的原子性操作会耗费性能（导致大量线程**空循环，自旋转**）



### Atomic 原子类

```java
// setup to use Unsafe.compareAndSwapInt for updates（更新操作时提供“比较并替换”的作用）
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;

static {
    try {
        valueOffset = unsafe.objectFieldOffset
            (AtomicInteger.class.getDeclaredField("value"));
    } catch (Exception ex) { throw new Error(ex); }
}

private volatile int value;
```

`AtomicInteger` 类主要利用 CAS (compare and swap) + volatile 和 native 方法来保证原子操作，从而避免 synchronized 的高开销，执行效率大为提升。

CAS 的原理是拿期望的值和原本的一个值作比较，如果相同则更新成新的值。UnSafe 类的 `objectFieldOffset()` 方法是一个本地方法，这个方法是用来拿到“原来的值”的内存地址。另外 value 是一个 volatile 变量，在内存中可见，因此 JVM 可以保证任何时刻任何线程总能拿到该变量的最新值



**基本类型**

常见原子类：AtomicInteger、AtomicBoolean、AtomicLong

常用API：

| 方法                                  | 作用                                                         |
| ------------------------------------- | ------------------------------------------------------------ |
| public final int get()                | 获取 AtomicInteger 的值                                      |
| public final int getAndIncrement()    | 以原子方式将当前值加 1，返回的是自增前的值                   |
| public final int incrementAndGet()    | 以原子方式将当前值加 1，返回的是自增后的值                   |
| public final int getAndSet(int value) | 以原子方式设置为 newValue 的值，返回旧值                     |
| public final int addAndGet(int data)  | 以原子方式将输入的数值与实例中的值相加并返回 实例：AtomicInteger 里的 value |



**原子引用**

原子引用：对 Object 进行原子操作，提供一种读和写都是原子性的对象引用变量

原子引用类：AtomicReference（存在ABA问题）、AtomicStampedReference（维护版本号解决ABA问题）、AtomicMarkableReference（维护是否修改过标记解决ABA问题）

AtomicReference 类：

- 构造方法：`AtomicReference<T> atomicReference = new AtomicReference<T>()`
- 常用 API：
  - `public final boolean compareAndSet(V expectedValue, V newValue)`：CAS 操作
  - `public final void set(V newValue)`：将值设置为 newValue
  - `public final V get()`：返回当前值



**原子数组**

原子数组类：AtomicIntegerArray、AtomicLongArray、AtomicReferenceArray

```java
/**
    参数1，提供数组、可以是线程不安全数组或线程安全数组
    参数2，获取数组长度的方法
    参数3，自增方法，回传 array, index
    参数4，打印数组的方法
*/
// supplier 提供者 无中生有 ()->结果
// function 函数 一个参数一个结果 (参数)->结果 , BiFunction (参数1,参数2)->结果
// consumer 消费者 一个参数没结果 (参数)->void
private static <T> void demo(
    Supplier<T> arraySupplier,
    Function<T, Integer> lengthFun,
    BiConsumer<T, Integer> putConsumer,
    Consumer<T> printConsumer ) {
    
    List<Thread> ts = new ArrayList<>();
    T array = arraySupplier.get();
    int length = lengthFun.apply(array);
    for (int i = 0; i < length; i++) {
        // 每个线程对数组作 10000 次操作
        ts.add(new Thread(() -> {
            for (int j = 0; j < 10000; j++) {
                putConsumer.accept(array, j%length);
            }
        }));
    }
    ts.forEach(t -> t.start()); // 启动所有线程
    
    ts.forEach(t -> {
        try {
            t.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }); // 等所有线程结束
    printConsumer.accept(array);
}

// 使用
demo(
    ()-> new AtomicIntegerArray(10),
    (array) -> array.length(),
    (array, index) -> array.getAndIncrement(index),
    array -> System.out.println(array)
);
```



**原子更新器**

原子更新器类：AtomicReferenceFieldUpdater、AtomicIntegerFieldUpdater、AtomicLongFieldUpdater

利用字段更新器，可以针对对象的某个域（Field）进行原子操作，只能配合 volatile 修饰的字段使用，否则会出现异常 `IllegalArgumentException: Must be volatile type`

常用 API：

- `static <U> AtomicIntegerFieldUpdater<U> newUpdater(Class<U> c, String fieldName)`：构造方法
- `abstract boolean compareAndSet(T obj, int expect, int update)`：CAS



**原子累加器**

原子累加器类：LongAdder、DoubleAdder、LongAccumulator、DoubleAccumulator

原子累加器在有竞争时，**设置多个累加单元**，Therad-0 累加 Cell[0]，而 Thread-1 累加Cell[1]... 最后将结果汇总。这样它们在累加时操作的不同的 Cell 变量，因此**减少了 CAS 重试失败，从而提高性能**。



### LongAdder

LongAdder 类有几个关键域:

```java
// 累加单元数组, 懒惰初始化
transient volatile Cell[] cells;

// 基础值, 如果没有竞争, 则用 cas 累加这个域
transient volatile long base;

// 在 cells 创建或扩容时, 置为 1, 表示加锁
transient volatile int cellsBusy;

```

Cell需要防止缓存行伪共享问题：一个缓存行容纳多个cell数据就叫做伪共享

```java
// 防止缓存行伪共享  
@sun.misc.Contended
static final class Cell {

    volatile long value;
    
    Cell(long x) { 
        value = x; 
    }

    // 最重要的方法, 用来 cas 方式进行累加, prev 表示旧值, next 表示新值
    final boolean cas(long prev, long next) {
        return UNSAFE.compareAndSwapLong(this, valueOffset, prev, next);
    }
// 省略不重要代码
}
```

> Cell 是数组形式，**在内存中是连续存储的**，64 位系统中，一个 Cell 为 24 字节（16 字节的对象头和 8 字节的 value），每一个 cache line 为 64 字节，因此缓存行可以存下 2 个的 Cell 对象，当 Core-0 要修改 Cell[0]、Core-1 要修改 Cell[1]，无论谁修改成功都会导致当前缓存行失效，从而导致对方的数据失效，需要重新去主存获取，影响效率
>
> @sun.misc.Contended：防止缓存行伪共享，在使用此注解的对象或字段的前后各增加 128 字节大小的 padding，使用 2 倍于大多数硬件缓存行让 CPU 将对象预读至缓存时**占用不同的缓存行**，这样就不会造成对方缓存行的失效



LongAdder 是 Java8 提供的类，跟 AtomicLong 有相同的效果，但对 CAS 机制进行了优化，尝试使用分段 CAS 以及自动分段迁移的方式来大幅度提升多线程高并发执行 CAS 操作的性能

优化核心思想：数据分离，将 AtomicLong 的**单点的更新压力分担到各个节点，空间换时间**，在低并发的时候直接更新，可以保障和 AtomicLong 的性能基本一致，而在高并发的时候通过分散减少竞争，提高了性能



### Unsafe

Unsafe 是 CAS 的核心类，Unsafe 对象提供了非常底层的，操作内存、线程的方法，Unsafe 对象不能直接调用，只能通过反射获得

Unsafe 类存在 sun.misc 包，其中所有方法都是 native 修饰的，都是直接调用**操作系统底层资源**执行相应的任务，基于该类可以直接操作特定的内存数据，其内部方法操作类似 C 的指针.

```java
import lombok.Data;
import sun.misc.Unsafe;

import java.lang.reflect.Field;

public class TestUnsafeCAS {

    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {

//        Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
//        theUnsafe.setAccessible(true);
//        Unsafe unsafe = (Unsafe) theUnsafe.get(null);
        Unsafe unsafe = UnsafeAccessor.getUnsafe();
        System.out.println(unsafe);

        // 1. 获取域的偏移地址
        long idOffset = unsafe.objectFieldOffset(Teacher.class.getDeclaredField("id"));
        long nameOffset = unsafe.objectFieldOffset(Teacher.class.getDeclaredField("name"));

        Teacher t = new Teacher();
        System.out.println(t);

        // 2. 执行 cas 操作
        unsafe.compareAndSwapInt(t, idOffset, 0, 1);
        unsafe.compareAndSwapObject(t, nameOffset, null, "张三");

        // 3. 验证
        System.out.println(t);
    }
}

@Data
class Teacher {
    volatile int id;
    volatile String name;
}
```



模拟实现原子整数：

```java
public static void main(String[] args) {
    MyAtomicInteger atomicInteger = new MyAtomicInteger(10);
    if (atomicInteger.compareAndSwap(20)) {
        System.out.println(atomicInteger.getValue());
    }
}

class MyAtomicInteger {
    private static final Unsafe UNSAFE;
    private static final long VALUE_OFFSET;
    private volatile int value;

    static {
        try {
            //Unsafe unsafe = Unsafe.getUnsafe()这样会报错，需要反射获取
            Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
            theUnsafe.setAccessible(true);
            UNSAFE = (Unsafe) theUnsafe.get(null);
            // 获取 value 属性的内存地址，value 属性指向该地址，直接设置该地址的值可以修改 value 的值
            VALUE_OFFSET = UNSAFE.objectFieldOffset(
                		   MyAtomicInteger.class.getDeclaredField("value"));
        } catch (NoSuchFieldException | IllegalAccessException e) {
            e.printStackTrace();
            throw new RuntimeException();
        }
    }

    public MyAtomicInteger(int value) {
        this.value = value;
    }
    public int getValue() {
        return value;
    }

    public boolean compareAndSwap(int update) {
        while (true) {
            int prev = this.value;
            int next = update;
            //							当前对象  内存偏移量    期望值 更新值
            if (UNSAFE.compareAndSwapInt(this, VALUE_OFFSET, prev, update)) {
                System.out.println("CAS成功");
                return true;
            }
        }
    }
}
```



## 不可变

### 不可变设计

**将一个类所有的属性都设置成 final 的，并且只允许存在只读方法，那么这个类基本上就具备不可变性了**。更严格的做法是**这个类本身也是 final 的**，也就是不允许继承。因为子类可以覆盖父类的方法，有可能改变不可变性.

Java SDK 里很多类都具备不可变性，例如经常用到的 String 和 Long、Integer、Double 等基础类型的包装类都具备不可变性，这些对象的线程安全性都是靠不可变性来保证的.

```java
public final class String implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];
    /** Cache the hash code for the string */
    private int hash; // Default to 0
    
    // ...
    
}
```

如果具备不可变性的类，需要提供类似修改的功能，具体该怎么操作呢？

**保护性拷贝 （defensive copy）**

通过创建副本对象来避免共享的手段称之为【保护性拷贝（defensive copy）】 

```java
public String substring(int beginIndex) {
    if (beginIndex < 0) {
        throw new StringIndexOutOfBoundsException(beginIndex);
    }
    int subLen = value.length - beginIndex;
    if (subLen < 0) {
        throw new StringIndexOutOfBoundsException(subLen);
    }
    return (beginIndex == 0) ? this : new String(value, beginIndex, subLen);
}

public String(char value[], int offset, int count) {
    if (offset < 0) {
        throw new StringIndexOutOfBoundsException(offset);
    }
    if (count <= 0) {
        if (count < 0) {
            throw new StringIndexOutOfBoundsException(count);
        }
        if (offset <= value.length) {
            this.value = "".value;
            return;
        }
    }
    if (offset > value.length - count) {
        throw new StringIndexOutOfBoundsException(offset + count);
    }
    this.value = Arrays.copyOfRange(value, offset, offset+count);
}
```



### 享元模式

享元模式（Flyweight pattern）： 用于减少创建对象的数量，通过重用数量有限的同一类对象，以减少内存占用和提高性能，这种类型的设计模式属于结构型模式，它提供了减少对象数量从而改善应用所需的对象结构的方式。体现：

**1. 包装类**

在JDK中 Boolean，Byte，Short，Integer，Long，Character 等包装类提供了 valueOf 方法，例如 Long 的valueOf 会缓存 -128~127 之间的 Long 对象，在这个范围之间会重用对象，大于这个范围，才会新建 Long 对象：

```java
public static Long valueOf(long l) {
    final int offset = 128;
    if (l >= -128 && l <= 127) { // will cache
        return LongCache.cache[(int)l + offset];
    }
    return new Long(l);
}
```

> **注意：** 
>
> Byte, Short, Long 缓存的范围都是 -128~127 
>
> Character 缓存的范围是 0~127 
>
> Integer的默认范围是 -128~127 
>
> - 最小值不能变 
>
> - 但最大值可以通过调整虚拟机参数 `-Djava.lang.Integer.IntegerCache.high` 来改变 
>
> Boolean 缓存了 TRUE 和 FALSE 

**2. String 串池** 

**3. BigDecimal BigInteger** 



### final

```java
public class TestFinal {
	final int a = 20;
}
```

字节码：

```java
0: aload_0
1: invokespecial #1 // Method java/lang/Object."<init>":()V
4: aload_0
5: bipush 20		// 将值直接放入栈中
7: putfield #2 		// Field a:I
<-- 写屏障
10: return
```



final 变量的赋值通过 putfield 指令来完成，在这条指令之后也会加入写屏障，保证在其它线程读到它的值时不会出现为 0 的情况

其他线程访问 final 修饰的变量:

- **复制一份放入栈中**直接访问，效率高
- 大于 short 最大值会将其复制到类的常量池，访问时从常量池获取



### 无状态

无状态：成员变量保存的数据也可以称为状态信息，无状态就是没有成员变量

Servlet 为了保证其线程安全，一般不为 Servlet 设置成员变量，这种没有任何成员变量的类是线程安全的



## 线程池

线程池：一个容纳多个线程的容器，容器中的线程可以重复使用，省去了频繁创建和销毁线程对象的操作

线程池作用：

1. 降低资源消耗，减少了创建和销毁线程的次数，每个工作线程都可以被重复利用，可执行多个任务
2. 提高响应速度，当任务到达时，如果有线程可以直接用，不会出现系统僵死
3. 提高线程的可管理性，如果无限制的创建线程，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控

线程池的核心思想：**线程复用**，同一个线程可以被重复使用，来处理多个任务

池化技术 (Pool) ：一种编程技巧，核心思想是资源复用，在请求量大时能优化应用性能，降低系统频繁建连的资源开销



### 自定义线程池

![](.\JUC\自定义线程池.png)

```java
/**
 * 任务拒绝策略
 *
 * @param <T>
 */
@FunctionalInterface
interface MyRejectPolicy<T> {
    void reject(MyBlockingQueue<T> queue, T task);
}

```

```java
import lombok.extern.slf4j.Slf4j;

import java.util.ArrayDeque;
import java.util.Deque;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 任务队列
 *
 * @param <T>
 */
@Slf4j(topic = "c.MyBlockingQueue")
class MyBlockingQueue<T> {

    // 1. 任务队列
    private Deque<T> queue = new ArrayDeque<>();

    // 2. 锁
    private ReentrantLock lock = new ReentrantLock();

    // 3. 生产者条件变量
    private Condition fullWaitSet = lock.newCondition();

    // 4. 消费者条件变量
    private Condition emptyWaitSet = lock.newCondition();

    // 5. 容量
    private int capcity;

    /**
     * 构造方法
     * @param capcity 线程池容量
     */
    public MyBlockingQueue(int capcity) {
        log.info("构造BlockingQueue");
        this.capcity = capcity;
    }


    /**
     * 获取队列头部一个元素, 阻塞至 获取到元素 或 超时时长
     *
     * @param timeout
     * @param unit
     * @return
     */
    public T poll(long timeout, TimeUnit unit) {
        lock.lock();
        try {
            // 将 timeout 统一转换为 纳秒
            long nanos = unit.toNanos(timeout);

            //如果队列为空,当前线程在 emptyWaitSet 上等待
            while (queue.isEmpty()) {
                try {
                    // 返回值是剩余时间
                    if (nanos <= 0) {
                        //等待超时后队列仍为空,返回null
                        return null;
                    }
                    nanos = emptyWaitSet.awaitNanos(nanos);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            //获取队列头部元素并将其从队列中移除
            T t = queue.removeFirst();
            //唤醒 等待在fullWaitSet上的任 意一个线程
            fullWaitSet.signal();

            return t;
        } finally {
            lock.unlock();
        }
    }


    /**
     * 获取队列头部一个元素, 阻塞至获取到任务为止
     * @return
     */
    public T take() {
        lock.lock();
        try {

            while (queue.isEmpty()) {
                try {
                    emptyWaitSet.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            T t = queue.removeFirst();
            fullWaitSet.signal();
            return t;
        } finally {
            lock.unlock();
        }
    }

    /**
     * 向队列尾部添加一个元素, 阻塞至 加入到队列 或 超时时长
     * @param task
     * @param timeout
     * @param timeUnit
     * @return
     */
    public boolean offer(T task, long timeout, TimeUnit timeUnit) {
        lock.lock();
        try {
            long nanos = timeUnit.toNanos(timeout);
            while (queue.size() == capcity) {
                try {
                    if(nanos <= 0) {
                        return false;
                    }
                    log.debug("等待加入任务队列 {} ...", task);
                    nanos = fullWaitSet.awaitNanos(nanos);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            log.debug("加入任务队列 {}", task);
            queue.addLast(task);
            //添加任务成功后,唤起 等待在emptyWaitSet上的 一个线程
            emptyWaitSet.signal();

            return true;
        } finally {
            lock.unlock();
        }
    }


    /**
     * 向队列尾部添加一个元素, 阻塞至 加入到队列为止
     * @param task
     */
    public void put(T task) {
        lock.lock();
        try {
            while (queue.size() == capcity) {
                try {
                    log.debug("等待加入任务队列 {} ...", task);
                    fullWaitSet.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            log.debug("加入任务队列 {}", task);
            queue.addLast(task);
            emptyWaitSet.signal();
        } finally {
            lock.unlock();
        }
    }


    /**
     * 返回队列当前size
     * @return 返回队列当前size
     */
    public int size() {
        lock.lock();
        try {
            return queue.size();
        } finally {
            lock.unlock();
        }
    }

    /**
     * 尝试将任务task放入队列, 如果队列已满,则执行拒绝策略rejectPolicy
     *
     * @param rejectPolicy
     * @param task
     */
    public void tryPut(MyRejectPolicy<T> rejectPolicy, T task) {
        lock.lock();
        try {
            // 判断队列是否满
            if(queue.size() == capcity) {
                log.info("队列已满,按照拒绝策略处理任务 {}",task);
                rejectPolicy.reject(this, task);
            } else {  // 有空闲
                log.debug("队列未满,任务 {} 加入到队列中 ", task);
                queue.addLast(task);
                emptyWaitSet.signal();
            }
        } finally {
            lock.unlock();
        }
    }

}
```

```java
import lombok.extern.slf4j.Slf4j;

import java.util.HashSet;
import java.util.concurrent.TimeUnit;

@Slf4j(topic = "c.MyThreadPool")
class MyThreadPool {

    // 任务队列
    private MyBlockingQueue<Runnable> taskQueue;
    
    //队列已满时的拒绝策略
    private MyRejectPolicy<Runnable> rejectPolicy;

    // 线程集合
    private HashSet<Worker> workers = new HashSet<>();

    // 核心线程数
    private int coreSize;

    // 获取任务时的超时时间
    private long timeout;
    private TimeUnit timeUnit;

    // 执行任务
    public void execute(Runnable task) {
        log.info("接收到任务需要执行: "+task);

        // 当任务数没有超过 coreSize 时，直接交给 worker 对象执行
        // 如果任务数超过 coreSize 时，加入任务队列暂存
        synchronized (workers) {
            if(workers.size() < coreSize) {
                Worker worker = new Worker(task,"worker--"+workers.size());
                workers.add(worker);
                log.info("coreSize未满,新增 worker  {} 来执行任务 {}", worker, task);
                worker.start();

            } else {
                log.info("coreSize已经满了!!!!!,需要先尝试将任务{} 放到等待队列中 ",task);
                taskQueue.tryPut(rejectPolicy, task);

            }
        }
    }

    /**
     * 构造函数
     *
     * @param coreSize 线程池最大核心线程数
     * @param timeout 和timeUnit一起指定超时时长
     * @param timeUnit 和timeout一起指定超时时长
     * @param queueCapcity 任务队列容量
     * @param rejectPolicy 任务队列满时针对添加操作的拒绝策略
     */
    public MyThreadPool(int coreSize, long timeout, TimeUnit timeUnit, int queueCapcity, MyRejectPolicy<Runnable> rejectPolicy) {
        log.info("构造ThreadPool");
        this.coreSize = coreSize;
        this.timeout = timeout;
        this.timeUnit = timeUnit;
        this.taskQueue = new MyBlockingQueue<>(queueCapcity);
        this.rejectPolicy = rejectPolicy;
    }

    /**
     * 线程池中的工作线程
     */
    class Worker extends Thread{
        /**
         * 执行任务主体
         */
        private Runnable task;

        public Worker(Runnable task,String workerName) {
            this.task = task;
            this.setName(workerName);
        }

        /**
         * 执行已有任务或从队列中获取一个任务执行.
         * 如果都执行完了,就结束线程
         */
        @Override
        public void run() {
            log.info("worker跑run了,让我看看有没有task来做");

            // 执行任务
            // 1) 当 task 不为空，执行任务
            // 2) 当 task 执行完毕，再接着从任务队列获取任务并执行
//            while(task != null || (task = taskQueue.take()) != null) {
            while(task != null || (task = taskQueue.poll(timeout, timeUnit)) != null) {
                try {
                    log.debug("获取到任务了,正在执行...{}", task);
                    task.run();
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    log.info("搞定一个任务 {},尝试获取新任务执行",task);
                    task = null;
                }
            }

            synchronized (workers) {
                log.debug("当前worker {} 因长时间没有获取到可执行任务 将被释放", this);
                workers.remove(this);
            }

        }

    }

}
```

```java
import lombok.extern.slf4j.Slf4j;
import java.util.concurrent.TimeUnit;

@Slf4j(topic = "c.TestCustomThreadPool")
public class TestMyThreadPool {

    public static void main(String[] args) {

        MyThreadPool threadPool = new MyThreadPool(1,
                3000,
                TimeUnit.MILLISECONDS,
                1,
                (queue, task)->{
            // 1. 死等
//            queue.put(task);
            // 2) 带超时等待
            queue.offer(task, 1500, TimeUnit.MILLISECONDS);
            // 3) 让调用者放弃任务执行
//            log.debug("放弃{}", task);
            // 4) 让调用者抛出异常
//            throw new RuntimeException("任务执行失败 " + task);
            // 5) 让调用者自己执行任务
//                log.info("当前拒绝策略: 让调用线程池的调用者自己执行任务,没有开新线程,直接调用的run()");
//                task.run();
        });

        int total =4;
        for (int i = 1; i <= total; i++) {
            int j = i;
            threadPool.execute(() -> {
                try {
                    log.debug("开始执行第 {}/{} 个任务 ", j,total);
                    Thread.sleep(1000L);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                log.debug("第 {}/{} 个任务 执行结束", j,total);
            });
        }
    }

}
```



### Executor框架

`Executor` 框架结构主要由三大部分组成：

**1、任务(`Runnable` /`Callable`)**

执行任务需要实现的 **`Runnable` 接口** 或 **`Callable`接口**。**`Runnable` 接口**或 **`Callable` 接口** 实现类都可以被 **`ThreadPoolExecutor`** 或 **`ScheduledThreadPoolExecutor`** 执行。

**`Runnable` 接口**不会返回结果或抛出检查异常，但是 **`Callable` 接口**可以

**2、任务的执行(`Executor`)**

如下图所示，包括任务执行机制的核心接口 **`Executor`** ，以及继承自 `Executor` 接口的 **`ExecutorService` 接口。`ThreadPoolExecutor`** 和 **`ScheduledThreadPoolExecutor`** 这两个关键类实现了 **`ExecutorService`** 接口。

![](.\JUC\executor-class-diagram.png)

**3、异步计算的结果(`Future`)**

**`Future`** 接口以及 `Future` 接口的实现类 **`FutureTask`** 类都可以代表异步计算的结果。

当我们把 **`Runnable`接口** 或 **`Callable` 接口** 的实现类提交给 **`ThreadPoolExecutor`** 或 **`ScheduledThreadPoolExecutor`** 执行。（调用 `submit()` 方法时会返回一个 **`FutureTask`** 对象）

**`Executor` 框架的使用示意图**

![](.\JUC\Executor框架的使用示意图-8GKgMC9g.png)

1. 主线程首先要创建实现 `Runnable` 或者 `Callable` 接口的任务对象。

2. 把创建完成的实现 `Runnable`/`Callable`接口的 对象直接交给 `ExecutorService` 执行: `ExecutorService.execute（Runnable command）`）或者也可以把 `Runnable` 对象或`Callable` 对象提交给 `ExecutorService` 执行（`ExecutorService.submit（Runnable task）`或 `ExecutorService.submit（Callable <T> task）`）。

3. 如果执行 `ExecutorService.submit（…）`，`ExecutorService` 将返回一个实现`Future`接口的对象（我们刚刚也提到过了执行 `execute()`方法和 `submit()`方法的区别，`submit()`会返回一个 `FutureTask` 对象）。由于 `FutureTask` 实现了 `Runnable`，我们也可以创建 `FutureTask`，然后直接交给 `ExecutorService` 执行。

4. 最后，主线程可以执行 `FutureTask.get()`方法来等待任务执行完成。主线程也可以执行 `FutureTask.cancel（boolean mayInterruptIfRunning）`来取消此任务的执行



### ThreadPoolExecutor

线程池实现类 `ThreadPoolExecutor` 是 `Executor` 框架最核心的类

#### 线程池状态

ThreadPoolExecutor 使用 **int 的高 3 位来表示线程池状态，低 29 位表示线程数量**

| 状态       | 高3位 | 接收新任务 | 处理阻塞任务队列 | 说明                                      |
| ---------- | ----- | ---------- | ---------------- | ----------------------------------------- |
| RUNNING    | 111   | Y          | Y                |                                           |
| SHUTDOWN   | 000   | N          | Y                | 不接收新任务，但处理阻塞队列剩余任务      |
| STOP       | 001   | N          | N                | 中断正在执行的任务，并抛弃阻塞队列任务    |
| TIDYING    | 010   | -          | -                | 任务全执行完毕，活动线程为 0 即将进入终结 |
| TERMINATED | 011   | -          | -                | 终止状态                                  |

从数字上比较，TERMINATED > TIDYING > STOP > SHUTDOWN > RUNNING .

因为第一位是符号位,RUNNING 是负数,所以最小.



这些信息存储在一个原子变量 ctl 中，目的是将线程池状态与线程个数合二为一，这样就可以用一次 cas 原子操作进行赋值

```java
// c 为旧值， ctlOf 返回结果为新值
ctl.compareAndSet(c, ctlOf(targetState, workerCountOf(c))));

// rs 为高 3 位代表线程池状态， wc 为低 29 位代表线程个数，ctl 是合并它们
private static int ctlOf(int rs, int wc) { return rs | wc; }
```



#### 参数

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                         RejectedExecutionHandler handler)
```

参数：

- corePoolSize 核心线程数目。 默认情况下，线程池中线程的数量如果 <= corePoolSize，那么即使这些线程处于空闲状态，那也不会被销毁。

- maximumPoolSize 最大线程数目 。当队列中存放的任务达到队列容量时，当前可以同时运行的数量变为最大线程数，创建线程并立即执行最新的任务，与核心线程数之间的差值又叫非核心线程数。

- keepAliveTime 空闲存活时间 - 当线程池中线程的数量大于corePoolSize，并且某个线程的空闲时间超过了keepAliveTime，那么这个线程就会被销毁。

- unit 时间单位 

- workQueue 工作队列 ，存放被提交但尚未被执行的任务

- threadFactory 线程工厂 - 可以为线程创建时起个好名字 

- handler 拒绝策略

  RejectedExecutionHandler 下有 4 个实现类：

  - AbortPolicy：让调用者抛出 RejectedExecutionException 异常，**默认策略**
  - CallerRunsPolicy：让调用者线程执行
  - DiscardPolicy：直接丢弃任务，不予任何处理也不抛出异常
  - DiscardOldestPolicy：放弃队列中最早的任务，把当前任务加入队列中尝试再次提交当前任务

  补充：其他框架拒绝策略

  - Dubbo：在抛出 RejectedExecutionException 异常前记录日志，并 dump 线程栈信息，方便定位问题
  - Netty：创建一个新线程来执行任务
  - ActiveMQ：带超时等待（60s）尝试放入队列
  - PinPoint：它使用了一个拒绝策略链，会逐一尝试策略链中每种拒绝策略

有没有办法既能保证任务不被丢弃且在服务器有余力时及时处理呢？

**任务持久化**的思路，这里所谓的任务持久化，包括但不限于:

1. 设计一张任务表间任务存储到 MySQL 数据库中。
2. `Redis`缓存任务。
3. 将任务提交到消息队列中。



#### 工作方式

```java
// 存放线程池的运行状态 (runState) 和线程池内有效线程的数量 (workerCount)
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

private static int workerCountOf(int c) {
    return c & CAPACITY;
}
//任务队列
private final BlockingQueue<Runnable> workQueue;

public void execute(Runnable command) {
    // 如果任务为null，则抛出异常。
    if (command == null)
        throw new NullPointerException();
    // ctl 中保存的线程池当前的一些状态信息
    int c = ctl.get();

    //  下面会涉及到 3 步 操作
    // 1.首先判断当前线程池中执行的任务数量是否小于 corePoolSize
    // 如果小于的话，通过addWorker(command, true)新建一个线程，并将任务(command)添加到该线程中；然后，启动该线程从而执行任务。
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 2.如果当前执行的任务数量大于等于 corePoolSize 的时候就会走到这里，表明创建新的线程失败。
    // 通过 isRunning 方法判断线程池状态，线程池处于 RUNNING 状态并且队列可以加入任务，该任务才会被加入进去
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 再次获取线程池状态，如果线程池状态不是 RUNNING 状态就需要从任务队列中移除任务，并尝试判断线程是否全部执行完毕。同时执行拒绝策略。
        if (!isRunning(recheck) && remove(command))
            reject(command);
        // 如果当前工作线程数量为0，新创建一个线程并执行。
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    //3. 通过addWorker(command, false)新建一个线程，并将任务(command)添加到该线程中；然后，启动该线程从而执行任务。
    // 传入 false 代表增加线程时判断当前线程数是否少于 maxPoolSize
    //如果addWorker(command, false)执行失败，则通过reject()执行相应的拒绝策略的内容。
    else if (!addWorker(command, false))
        reject(command);
}
```

1. 创建线程池，这时没有创建线程（**懒惰**），等待提交过来的任务请求，调用 execute 方法才会创建线程
2. 当调用 execute() 方法添加一个请求任务时，线程池会做如下判断：
   - 如果正在运行的线程数量小于 corePoolSize，那么马上创建线程运行这个任务
   - 如果正在运行的线程数量大于或等于 corePoolSize，那么将这个任务放入队列
   - 如果这时队列满了且正在运行的线程数量还小于 maximumPoolSize，那么会创建非核心线程**立刻运行这个任务**，对于阻塞队列中的任务不公平。
   - 如果队列满了且正在运行的线程数量大于或等于 maximumPoolSize，那么线程池会启执行**拒绝策略**
3. 当一个线程完成任务时，会从队列中取下一个任务来执行
4. 当一个线程空闲超过空闲存活时间（keepAliveTime）时，线程池会判断：如果当前运行的线程数大于 corePoolSize，那么这个线程就被停掉，所以线程池的所有任务完成后最终会收缩到 corePoolSize 大小

<img src=".\JUC\工作原理.png" style="zoom: 50%;" />

#### JDK Executors类中提供的典型线程池实现

##### **newFixedThreadPool**

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```

特点：

- 核心线程数 == 最大线程数（没有救急线程被创建），因此也无需空闲存活时间 
- 阻塞队列是无界的，可以放任意数量的任务 
- 可能出现OOM，因为队列是无界的，所以任务可能挤爆内存
- 适用于任务量已知，相对耗时的任务



##### **newCachedThreadPool**

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

特点 :

- 核心线程数是 0，最大线程数是 Integer.MAX_VALUE，救急线程的空闲生存时间是 60s，意味着 
  - 全部都是救急线程（60s 后可以回收）
  - 救急线程可以无限创建 

- 队列采用了 SynchronousQueue 同步队列实现，特点是它**没有容量**，没有线程来取是放不进去的，每来个任务就必须有线程接着（类似一手交钱、一手交货）

* 适合任务数比较密集，但每个任务执行时间较短的情况



##### **newSingleThreadExecutor**

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

特点： 

* 线程数固定为 1，任务数多于 1 时，会放入无界队列排队。

* 任务执行完毕，这唯一的线程也不会被释放。 

* 适合希望多个任务顺序执行场景。



和自己创建一个线程来工作的区别？

自己创建一个单线程串行执行任务，如果任务执行失败而终止那么没有任何补救措施，而线程池还会新建一个线程，保证池的正常工作 



和Executors.newFixedThreadPool(1)的区别？

Executors.newSingleThreadExecutor() 线程个数始终为1，不能修改 

Executors.newFixedThreadPool(1) 初始时为1，以后还可以修改，对外暴露的是 ThreadPoolExecutor 对象，可以强转后调用 setCorePoolSize 等方法进行修改。



##### newWorkStealingPool

```java
public static ExecutorService newWorkStealingPool() {
    return new ForkJoinPool
        (Runtime.getRuntime().availableProcessors(),
         ForkJoinPool.defaultForkJoinWorkerThreadFactory,
         null, true);
}
```

特点：

* 线程数会参照当前可用的处理核心数
* 每个线程都有自己的双端队列，当自己的任务处理完毕后，会去别的线程的任务队列尾部拿任务来执行，加快执行速率
* 工作窃取池不保证提交任务的执行顺序

JDK8引入，返回的就是ForkJoinPool，1.8用的并行流就是这个线程池。



##### newScheduledThreadPool

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE,
              DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
              new DelayedWorkQueue());
    }
```

创建一个定长线程池，支持定时及周期性任务执行。

#### 开发要求

阿里巴巴 Java 开发手册要求：

- **线程资源必须通过线程池提供，不允许在应用中自行显式创建线程**

  - 使用线程池的好处是减少在创建和销毁线程上所消耗的时间以及系统资源的开销，解决资源不足的问题
  - 如果不使用线程池，有可能造成系统创建大量同类线程而导致消耗完内存或者过度切换的问题

- 线程池**不允许使用 Executors 去创建，而是通过 ThreadPoolExecutor 的方式**，这样的处理方式更加明确线程池的运行规则，规避资源耗尽的风险

  Executors 返回的线程池对象弊端如下：

  - FixedThreadPool 和 SingleThreadExecutor：请求队列长度为 Integer.MAX_VALUE，可能会堆积大量的请求，从而导致 OOM（内存溢出）
  - CacheThreadPool 和 ScheduledThreadPool：允许创建线程数量为 Integer.MAX_VALUE，可能会创建大量的线程，导致 OOM

创建多大容量的线程池合适？

- 一般来说池中**总线程数是核心池线程数量两倍**，确保当核心池有线程停止时，核心池外有线程进入核心池
- 过小会导致程序不能充分地利用系统资源、容易导致饥饿
- 过大会导致更多的线程上下文切换，占用更多内存

核心线程数常用公式：

- **CPU 密集型任务   (N+1)：** 这种任务消耗的是 CPU 资源，可以将核心线程数设置为 **N (CPU 核心数) + 1**，比 CPU 核心数多出来的一个线程是为了防止线程发生缺页中断，或者其它原因导致的任务暂停而带来的影响。一旦任务暂停，CPU 某个核心就会处于空闲状态，而在这种情况下多出来的一个线程就可以充分利用 CPU 的空闲时间

- **I/O 密集型任务（2N+1）：** 这种系统 CPU 处于阻塞状态，用大部分的时间来处理 I/O 交互，而线程在处理 I/O 的时间段内不会占用 CPU 来处理，这时就可以将 CPU 交出给其它线程使用，因此在 I/O 密集型任务的应用中，我们可以多配置一些线程，具体的计算方法是 **2N+1 或 CPU 核数/ (1-阻塞系数)，阻塞系数在 0.8~0.9 之间**

- 经验公式如下 

  ```
  线程数 = 核数 * 期望 CPU 利用率 * 总时间(CPU计算时间+等待时间) / CPU 计算时间
  ```

#### 提交方法

```java
// 执行任务
void execute(Runnable command);

// 提交任务 task，用返回值 Future 获得任务执行结果
<T> Future<T> submit(Callable<T> task);
<T> Future<T> submit(Runnable task, T result);
Future<?> submit(Runnable task);

// 提交 tasks 中所有任务
<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
    throws InterruptedException;

// 提交 tasks 中所有任务，带超时时间
<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                              long timeout, TimeUnit unit)
    throws InterruptedException;

// 提交 tasks 中所有任务，哪个任务先成功执行完毕，返回此任务执行结果，其它任务取消
<T> T invokeAny(Collection<? extends Callable<T>> tasks)
    throws InterruptedException, ExecutionException;

// 提交 tasks 中所有任务，哪个任务先成功执行完毕，返回此任务执行结果，其它任务取消，带超时时间
<T> T invokeAny(Collection<? extends Callable<T>> tasks,
                long timeout, TimeUnit unit)
 throws InterruptedException, ExecutionException, TimeoutException;
```

Future 接口有 5 个方法，它们分别是取消任务的方法 cancel()、判断任务是否已取消的方法 isCancelled()、判断任务是否已结束的方法 isDone()以及2 个获得任务执行结果的 get() 和 get(timeout, unit)

`execute()`方法用于提交不需要返回值的任务，所以无法判断任务是否被线程池执行成功与否；

`submit()`方法用于提交需要返回值的任务。线程池会返回一个 `Future` 类型的对象，通过这个 `Future` 对象可以判断任务是否执行成功，并且可以通过 `Future` 的 `get()`方法来获取返回值，`get()`方法会阻塞当前线程直到任务完成，而使用 `get（long timeout，TimeUnit unit）`方法的话，如果在 `timeout` 时间内任务还没有执行完，就会抛出 `java.util.concurrent.TimeoutException`



#### 关闭方法

ExecutorService 类 API：

| 方法                                                  | 说明                                                         |
| ----------------------------------------------------- | ------------------------------------------------------------ |
| void shutdown()                                       | 线程池状态变为 SHUTDOWN，线程池不再接受新任务了，但是队列里的任务得执行完毕，不会阻塞调用线程的执行 |
| List shutdownNow()                                    | 线程池状态变为 STOP，用 interrupt 中断正在执行的任务，线程池会终止当前正在运行的任务，并停止处理排队的任务并返回正在等待执行的 List |
| boolean isShutdown()                                  | 调用 `shutdown()` 方法后返回为 true。                        |
| boolean isTerminated()                                | 当调用 `shutdown()` 方法后，并且所有提交的任务完成后返回为 true |
| boolean awaitTermination(long timeout, TimeUnit unit) | 调用 shutdown 后，由于调用线程不会等待所有任务运行结束，如果它想在线程池 TERMINATED 后做些事情，可以利用此方法等待 |



#### 任务调度

##### Timer

Timer 可以实现延时功能和周期性任务，Timer 的优点在于简单易用，但由于所有任务都是由同一个线程来调度，因此所有任务都是串行执行的，同一时间只能有一个任务在执行，前一个任务的延迟或异常都将会影响到之后的任务。

核心就是一个优先队列和封装的执行任务的线程。维持一个小根堆，最快需要执行的任务排在优先队列的第一个，然后有个TimerThread线程不断拿排在第一个的任务和当前时间对比，时间到了就执行，时间未到就调用 wait() 等待

##### Scheduled

任务调度线程池 ScheduledThreadPoolExecutor 继承 ThreadPoolExecutor：

构造方法：`Executors.newScheduledThreadPool(int corePoolSize)`

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE,
              DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
              new DelayedWorkQueue());
    }
```

ScheduledThreadPoolExecutor 和 Timer 对比：

- `Timer` 对系统时钟的变化敏感，`ScheduledThreadPoolExecutor`不是；
- `Timer` 只有一个执行线程，因此长时间运行的任务可以延迟其他任务。 `ScheduledThreadPoolExecutor` 可以配置任意数量的线程。 此外，如果你想（通过提供 `ThreadFactory`），你可以完全控制创建的线程;
- 在`TimerTask` 中抛出的运行时异常会杀死一个线程，从而导致 `Timer` 死机即计划任务将不再运行。`ScheduledThreadExecutor` 不仅捕获运行时异常，还允许您在需要时处理它们（通过重写 `afterExecute` 方法`ThreadPoolExecutor`）。抛出异常的任务将被取消，但其他任务将继续运行。



#### 正确处理异常

1. 主动捕获

2. 在执行Future的get()时会获取到异常栈信息

   ```java
   ExecutorService pool = Executors.newFixedThreadPool(1);
   
   Future<Boolean> f = pool.submit(() -> {
       log.debug("task1");
       int i = 1 / 0;
       return true;
   });
   log.debug("result:{}", f.get());
   ```




#### 线程池中线程异常后，销毁还是复用？

先说结论，需要分两种情况：

- **使用`execute()`提交任务**：当任务通过`execute()`提交到线程池并在执行过程中抛出异常时，如果这个异常没有在任务内被捕获，那么该异常会导致当前线程终止，并且异常会被打印到控制台或日志文件中。线程池会检测到这种线程终止，并创建一个新线程来替换它，从而保持配置的线程数不变。
- **使用`submit()`提交任务**：对于通过`submit()`提交的任务，如果在任务执行中发生异常，这个异常不会直接打印出来。相反，异常会被封装在由`submit()`返回的`Future`对象中。当调用`Future.get()`方法时，可以捕获到一个`ExecutionException`。在这种情况下，线程不会因为异常而终止，它会继续存在于线程池中，准备执行后续的任务。

简单来说：使用`execute()`时，未捕获异常导致线程终止，线程池创建新线程替代；使用`submit()`时，异常被封装在`Future`中，线程继续复用。

这种设计允许`submit()`提供更灵活的错误处理机制，因为它允许调用者决定如何处理异常，而`execute()`则适用于那些不需要关注执行结果的场景。

#### 线程池命名

1. **利用 guava 的 `ThreadFactoryBuilder`**

   ```java
   ThreadFactory threadFactory = new ThreadFactoryBuilder()
                           .setNameFormat(threadNamePrefix + "-%d")
                           .setDaemon(true).build();
   ExecutorService threadPool = new ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime, TimeUnit.MINUTES, workQueue, threadFactory);
   ```

2. **自己实现 `ThreadFactory`。**

   ```java
   import java.util.concurrent.ThreadFactory;
   import java.util.concurrent.atomic.AtomicInteger;
   
   /**
    * 线程工厂，它设置线程名称，有利于我们定位问题。
    */
   public final class NamingThreadFactory implements ThreadFactory {
   
       private final AtomicInteger threadNum = new AtomicInteger();
       private final String name;
   
       /**
        * 创建一个带名字的线程池生产工厂
        */
       public NamingThreadFactory(String name) {
           this.name = name;
       }
   
       @Override
       public Thread newThread(Runnable r) {
           Thread t = new Thread(r);
           t.setName(name + " [#" + threadNum.incrementAndGet() + "]");
           return t;
       }
   }
   ```

   

### ForkJoin

Fork/Join：线程池的实现，体现是**分治思想**，适用于能够进行任务拆分的 CPU 密集型运算，用于并行计算

将一个大任务拆分为算法上相同的小任务，然后将这些小任务并行执行

**Fork 对应的是分治任务模型里的任务分解，Join 对应的是结果合并**。Fork/Join 计算框架主要包含两部分，一部分是**分治任务的线程池 ForkJoinPool**，另一部分是**分治任务 ForkJoinTask**

ForkJoinTask 是一个抽象类，最核心的是 fork() 方法和 join() 方法，其中 fork() 方法会异步地执行一个子任务，而 join() 方法则会阻塞当前线程来等待子任务的执行结果。ForkJoinTask 有两个子类——**RecursiveAction 和 RecursiveTask**，通过名字你就应该能知道，它们都是用递归的方式来处理分治任务的。这两个子类都定义了抽象方法 compute()，不过区别是 RecursiveAction 定义的 compute() 没有返回值，而 RecursiveTask 定义的 compute() 方法是有返回值的。这两个子类也是抽象类，在使用的时候，需要你定义子类去扩展。

```java
static void main(String[] args){
  // 创建分治任务线程池  
  ForkJoinPool fjp = 
    new ForkJoinPool(4);
  // 创建分治任务
  Fibonacci fib = 
    new Fibonacci(30);   
  // 启动分治任务  
  Integer result = 
    fjp.invoke(fib);
  // 输出结果  
  System.out.println(result);
}
// 递归任务
static class Fibonacci extends 
    RecursiveTask<Integer>{
  final int n;
  Fibonacci(int n){this.n = n;}
  protected Integer compute(){
    if (n <= 1)
      return n;
    Fibonacci f1 = 
      new Fibonacci(n - 1);
    // 创建子任务  
    f1.fork();
    Fibonacci f2 = 
      new Fibonacci(n - 2);
    // 等待子任务结果，并合并结果  
    return f2.compute() + f1.join();
  }
}
```

ForkJoinPool 实现了**工作窃取算法**来提高 CPU 的利用率：

- 每个线程都维护了一个**双端队列**，用来存储需要执行的任务
- 工作窃取算法允许空闲的线程从其它线程的双端队列中窃取一个任务来执行
- 窃取的必须是最晚的任务，避免和队列所属线程发生竞争，但是队列中只有一个任务时还是会发生竞争



## JUC

### AQS

AQS：AbstractQueuedSynchronizer，**是阻塞式锁和相关的同步器工具的框架**，AQS 就是一个抽象类，主要用来构建锁和同步器。

AQS 用 state 状态属性来表示资源的状态（分**独占模式和共享模式**），子类需要定义如何维护这个状态，控制如何获取锁和释放锁

- 独占模式是只有一个线程能够访问资源，如 ReentrantLock
- 共享模式允许多个线程访问资源，如 Semaphore，ReentrantReadWriteLock 是组合式

AQS 核心思想：

- 如果被请求的共享资源 state 空闲，则将当前请求资源的线程设置为有效的工作线程，并将共享资源设置锁定状态
- 请求的共享资源被占用，AQS 用队列实现线程阻塞等待以及被唤醒时锁分配的机制，将暂时获取不到锁的线程加入到队列中
- 等待队列：使用了CLH 队列，并不支持优先级队列，**同步队列是双向链表**，类似于 Monitor 的 EntryList 
- 条件变量：条件变量来实现等待、唤醒机制，支持多个条件变量，类似于 Monitor 的 WaitSet ，**条件队列是单向链表**

![](.\JUC\AQS.png)

同步器的设计是基于**模板方法模式**，该模式是基于继承的，主要是为了在不改变模板结构的前提下在子类中重新定义模板中的内容以实现复用代码

- 使用者继承 `AbstractQueuedSynchronizer` 并重写指定的方法
- 将 AQS 组合在自定义同步组件的实现中，并调用其模板方法，这些模板方法会调用使用者重写的方法

AQS 使用了模板方法模式，自定义同步器时需要重写下面几个 AQS 提供的模板方法：

```java
isHeldExclusively()		//该线程是否正在独占资源。只有用到condition才需要去实现它
tryAcquire(int)			//独占方式。尝试获取资源，成功则返回true，失败则返回false
tryRelease(int)			//独占方式。尝试释放资源，成功则返回true，失败则返回false
tryAcquireShared(int)	//共享方式。尝试获取资源。负数表示失败；0表示成功但没有剩余可用资源；正数表示成功且有剩余资源
tryReleaseShared(int)	//共享方式。尝试释放资源，成功则返回true，失败则返回false
```

- 默认情况下，每个方法都抛出 `UnsupportedOperationException`
- 这些方法的实现必须是内部线程安全的
- AQS 类中的其他方法都是 final ，所以无法被其他类使用，只有这几个方法可以被其他类使用



**CLH锁**

CLH 锁是对自旋锁的一种改进，有效的解决了自旋锁竞争激烈时饥饿和性能较差问题。首先它将线程组织成一个队列，保证先请求的线程先获得锁，避免了饥饿问题。其次锁状态去中心化，让每个线程在不同的状态变量中自旋，这样当一个线程释放它的锁时，只能使其后续线程的高速缓存失效，缩小了影响范围，从而减少了 CPU 的开销。

CLH 锁数据结构很简单，类似一个链表队列，所有请求获取锁的线程会排列在链表队列中，自旋访问队列中前一个节点的状态。当一个节点释放锁时，只有它的后一个节点才可以得到锁。每一个 CLH 节点有两个属性：所代表的线程和标识是否持有锁的状态变量。



### ReentrantLock 原理

ReentrantLock 是基于 AQS 实现的可重入锁，支持公平和非公平两种方式。

内部实现依靠一个state 变量和两个等待队列：同步队列和等待队列。

利用 CAS 和 修改 state 来争抢锁，争抢不到就进入同步队列等待，同步队列是一个双向链表；每个条件变量对应着一个等待队列。

是否公平的区别是：线程获取锁时是加入到队列尾部还是直接利用 CAS 争抢锁。公平锁先检查 AQS 队列中是否有前驱节点，没有才去 CAS 竞争。



![](.\JUC\ReentrantLock.png)

先从构造器开始看，默认为**非公平锁**实现

```java
public ReentrantLock() {
   sync = new NonfairSync();
}
```

#### 加解锁流程

**没有竞争**：ExclusiveOwnerThread 属于 Thread-0，state 设置为 1

```java
// ReentrantLock.NonfairSync#lock
final void lock() {
    // 用 cas 尝试（仅尝试一次）将 state 从 0 改为 1, 如果成功表示【获得了独占锁】
    if (compareAndSetState(0, 1))
        // 设置当前线程为独占线程
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);//失败进入
}
```

**第一个竞争出现**：

```java
// AbstractQueuedSynchronizer#acquire
public final void acquire(int arg) {
    // tryAcquire 尝试获取锁失败时, 会调用 addWaiter 将当前线程封装成node入队，acquireQueued 阻塞当前线程，
    // acquireQueued 返回 true 表示挂起过程中线程被中断唤醒过，false 表示未被中断过
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        // 如果线程被中断了逻辑来到这，完成一次真正的打断效果
        selfInterrupt();
}
```

1. Thread-0 持有锁，Thread-1 执行，CAS 尝试将 state 由 0 改为 1，结果失败（第一次），进入 acquire 逻辑

2. 进入 tryAcquire 再次尝试获取锁逻辑，这时 state 已经是1，结果仍然失败（第二次）

3. 进入 addWaiter 逻辑，构造 Node 队列

   * 图中黄色三角表示该 Node 的 waitStatus 状态，其中 0 为默认**正常状态**

   * Node 的创建是懒惰的，其中第一个 Node 称为 **Dummy（哑元）或哨兵**，用来占位，并不关联线程

4. 当前线程进入 acquireQueued 逻辑 

   1. acquireQueued 会在一个死循环中不断尝试获得锁，失败后进入 park 阻塞 

   2. 如果自己是紧邻着 head（排第二位），那么再次 tryAcquire 尝试获取锁，state 仍为 1 则失败（第三次）

   3. 进入 shouldParkAfterFailedAcquire 逻辑，**将前驱 node 的 waitStatus 改为 -1**，返回 false；waitStatus 为 -1 的节点用来唤醒下一个节点
   4. shouldParkAfterFailedAcquire 执行完毕回到 acquireQueued ，再次 tryAcquire 尝试获取锁，这时 state 仍为 1 获取失败（第四次）
   5. 当再次进入 shouldParkAfterFailedAcquire 时，这时其前驱 node 的 waitStatus 已经是 -1 了，返回 true
   6. 进入 parkAndCheckInterrupt， Thread-1 park（灰色表示）

| <img src=".\JUC\addWaiter.png"  />   | ![](.\JUC\acquireQueued.png) |
| ------------------------------------ | ---------------------------- |
| ![](.\JUC\parkAndCheckInterrupt.png) | ![](.\JUC\多个失败.png)      |

```java
final boolean acquireQueued(final Node node, int arg) {
    // true 表示当前线程抢占锁失败，false 表示成功
    boolean failed = true;
    try {
        // 中断标记，表示当前线程是否被中断
        boolean interrupted = false;
        for (;;) {
            // 获得当前线程节点的前驱节点
            final Node p = node.predecessor();
            // 前驱节点是 head, FIFO 队列的特性表示轮到当前线程可以去获取锁
            if (p == head && tryAcquire(arg)) {
                // 获取成功, 设置当前线程自己的 node 为 head
                setHead(node);
                p.next = null; // help GC
                // 表示抢占锁成功
                failed = false;
                // 返回当前线程是否被中断
                return interrupted;
            }
            // 判断是否应当 park，返回 false 后需要新一轮的循环，返回 true 进入条件二阻塞线程
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                // 条件二返回结果是当前线程是否被打断，没有被打断返回 false 不进入这里的逻辑
                // 【就算被打断了，也会继续循环，并不会返回】
                interrupted = true;
        }
    } finally {
        // 【可打断模式下才会进入该逻辑】
        if (failed)
            cancelAcquire(node);
    }
}
```



**原OwnerThread释放锁时**：

Thread-0 释放锁，进入 release 流程

1. 进入 tryRelease，设置 exclusiveOwnerThread 为 null，state = 0

2. 当前队列不为 null，并且 head 的 waitStatus = -1，进入 unparkSuccessor
3. 找到队列中距离 head 最近的一个没取消的 Node，unpark 恢复其运行，本例中即为 Thread-1
4. 回到 Thread-1 的 acquireQueued 流程
5. 唤醒的线程会从 park 位置开始执行，如果加锁成功（没有竞争），会设置
   - exclusiveOwnerThread 为 Thread-1，state = 1
   - head 指向刚刚 Thread-1 所在的 Node，该 Node 会清空 Thread
   - 原本的 head 因为从链表断开，而可被垃圾回收

![](.\JUC\release.png)

但是，如果这时有其它线程来竞争**（非公平）**，例如这时 Thread-4 来了并抢占了锁

- Thread-4 被设置为 exclusiveOwnerThread，state = 1
- Thread-1 再次进入 acquireQueued 流程，获取锁失败，重新进入 park 阻塞

![](.\JUC\非公平锁.png)



#### 可重入原理

通过state计数的增加减少来实现可重入

```java
static final class NonfairSync extends Sync {
    // ...
    
    // Sync 继承过来的方法, 方便阅读, 放在此处
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 如果已经获得了锁, 线程还是当前线程, 表示发生了锁重入
        else if (current == getExclusiveOwnerThread()) {
            // state++
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
    
    // Sync 继承过来的方法, 方便阅读, 放在此处
    protected final boolean tryRelease(int releases) {
        // state-- 
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        // 支持锁重入, 只有 state 减为 0, 才释放成功
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }
}
```



#### 可打断原理

**不可打断模式**

在此模式下，即使它被打断，仍会驻留在 AQS 队列中，一直要等到获得锁后方能得知自己被打断了.

被打断时只是记录下被打断过,等获得锁以后才继续向下运行

```java
// Sync 继承自 AQS
static final class NonfairSync extends Sync {
    // ...
    
    private final boolean parkAndCheckInterrupt() {
        // 如果打断标记已经是 true, 则 park 会失效
        LockSupport.park(this);
        // interrupted 会清除打断标记
        return Thread.interrupted();
    }
    
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null;
                    failed = false;
                    // 还是需要获得锁后, 才能返回打断状态
                    return interrupted;
                }
                if (
                    shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt()
                ) {
                    // 如果是因为 interrupt 被唤醒, 返回打断状态为 true
                    interrupted = true;
                }
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
    
    public final void acquire(int arg) {
        if (
            !tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg)
        ) {
            // 如果打断状态为 true
            selfInterrupt();
        }
    }
    
    static void selfInterrupt() {
        // 重新产生一次中断
        Thread.currentThread().interrupt();
    }
}
```

**可打断模式**

被打断时直接抛异常 InterruptedException

```java
static final class NonfairSync extends Sync {
    public final void acquireInterruptibly(int arg) throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        // 如果没有获得到锁, 进入 ㈠
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }
    
    // ㈠ 可打断的获取锁流程
    private void doAcquireInterruptibly(int arg) throws InterruptedException {
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt()) {
                    // 在 park 过程中如果被 interrupt 会进入此
                    // 这时候抛出异常, 而不会再次进入 for (;;)
                    throw new InterruptedException();
                }
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
}

```



#### 公平原理

与非公平锁主要区别在于 tryAcquire 方法：先检查 AQS 队列中是否有前驱节点，没有才去 CAS 竞争

```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;
    final void lock() {
        acquire(1);
    }
    
    // AQS 继承过来的方法, 方便阅读, 放在此处
    public final void acquire(int arg) {
        if (
            !tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg)
        ) {
            selfInterrupt();
        }
    }
    // 与非公平锁主要区别在于 tryAcquire 方法的实现
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            // 先检查 AQS 队列中是否有前驱节点, 没有才去竞争
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
    
    // ㈠ AQS 继承过来的方法, 方便阅读, 放在此处
    public final boolean hasQueuedPredecessors() {
        Node t = tail;
        Node h = head;
        Node s;
        // h != t 时表示队列中有 Node
        return h != t &&
            (
            // (s = h.next) == null 表示队列中还有没有老二
            (s = h.next) == null ||
            // 或者队列中老二线程不是此线程
            s.thread != Thread.currentThread()
        );
    }
}
```



#### 条件变量原理

每个条件变量其实就对应着一个等待队列，其实现类是 ConditionObject 

**await 流程** ：

总体流程是将 await 线程包装成 node 节点放入 ConditionObject 的条件队列，如果被唤醒就将 node 转移到 AQS 的执行阻塞队列，等待获取锁，**每个 Condition 对象都包含一个等待队列**

1. 开始 Thread-0 持有锁，调用 await，线程进入 ConditionObject 的 addConditionWaiter 流程 ，直到被唤醒或打断
2. 创建新的 Node ，状态为 -2（Node.CONDITION），关联 Thread-0，加入等待队列 ConditionObject 尾部
3. 接下来 Thread-0 进入 AQS 的 fullyRelease 流程，释放同步器上的锁，fullyRelease 中会 unpark AQS 队列中的下一个节点竞争锁
4. park 阻塞 Thread-0

| ![](.\JUC\await.png) | ![](.\JUC\await1.png) |
| -------------------- | --------------------- |



**signal 流程** 

1. 假设 Thread-1 要来唤醒 Thread-0
2. 进入 ConditionObject 的 doSignal 流程，取得等待队列中第一个 Node，即 Thread-0 所在 Node
3. 执行 transferForSignal 流程，将该 Node 加入 AQS 队列尾部，将 Thread-0 的 waitStatus 改为 0，Thread-3 的waitStatus 改为 -1
4. Thread-1 释放锁，进入 unlock 流程

| ![](.\JUC\sihnal.png) | ![](.\JUC\isignal1.png) |
| --------------------- | ----------------------- |



### ReentrantReadWriteLock

独占锁：指该锁一次只能被一个线程所持有，对 ReentrantLock 和 Synchronized 而言都是独占锁

共享锁：指该锁可以被多个线程锁持有

ReentrantReadWriteLock 其**读锁是共享锁，写锁是独占锁**

​	

注意：

- 读-读能共存、读-写不能共存、写-写不能共存
- 读锁不支持条件变量，写锁支持
- **重入时升级不支持**：持有读锁的情况下去获取写锁会导致获取写锁永久等待，需要先释放读，再去获得写
- **重入时降级支持**：持有写锁的情况下去获取读锁，造成只有当前线程会持有读锁，因为写锁会互斥其他的锁

读锁不能升级为写锁：本线程在释放读锁之前，想要获取写锁是不一定能获取到的，因为其他线程可能持有读锁（读锁共享），可能导致阻塞较长的时间，所以java干脆直接不支持读锁升级为写锁。

写锁可以降级为读锁：本线程在释放写锁之前，获取读锁一定是可以立刻获取到的，不存在其他线程持有读锁或者写锁（读写锁互斥），所以java允许锁降级。



构造方法：

- `public ReentrantReadWriteLock()`：默认构造方法，非公平锁
- `public ReentrantReadWriteLock(boolean fair)`：true 为公平锁

常用API：

- `public ReentrantReadWriteLock.ReadLock readLock()`：返回读锁
- `public ReentrantReadWriteLock.WriteLock writeLock()`：返回写锁
- `public void lock()`：加锁
- `public void unlock()`：解锁
- `public boolean tryLock()`：尝试获取锁



#### 实现原理

读写锁用的是同一个 Sycn 同步器，因此等待队列、state 等也是同一个，原理与 ReentrantLock 加锁相比没有特殊之处，不同是**写锁状态占了 state 的低 16 位，而读锁使用的是 state 的高 16 位**



**t1 线程：w.lock（写锁），成功上锁 state = 0_1**

```java
// lock()  -> sync.acquire(1);
public void lock() {
    sync.acquire(1);
}
public final void acquire(int arg) {
    // 尝试获得写锁，获得写锁失败，将当前线程关联到一个 Node 对象上, 模式为独占模式 
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

```java
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    // 获得低 16 位, 代表写锁的 state 计数
    int w = exclusiveCount(c);
    // 说明有读锁或者写锁
    if (c != 0) {
        // c != 0 and w == 0 表示有读锁，【读锁不能升级】，直接返回 false
        // w != 0 说明有写锁，写锁的拥有者不是自己，获取失败
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        
        // 执行到这里只有一种情况：【写锁重入】，所以下面几行代码不存在并发
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // 写锁重入, 获得锁成功，没有并发，所以不使用 CAS
        setState(c + acquires);
        return true;
    }
    
    // c == 0，说明没有任何锁，判断写锁是否该阻塞，是 false 就尝试获取锁，失败返回 false
    if (writerShouldBlock() || !compareAndSetState(c, c + acquires))
        return false;
    // 获得锁成功，设置锁的持有线程为当前线程
    setExclusiveOwnerThread(current);
    return true;
}
// 非公平锁 writerShouldBlock 总是返回 false, 无需阻塞
final boolean writerShouldBlock() {
    return false; 
}
// 公平锁会检查 AQS 队列中是否有前驱节点, 没有(false)才去竞争
final boolean writerShouldBlock() {
    return hasQueuedPredecessors();
}
```

**t2线程： r.lock（读锁），进入 tryAcquireShared 流程**

```java
public void lock() {
    sync.acquireShared(1);
}
public final void acquireShared(int arg) {
    // tryAcquireShared 返回负数, 表示获取读锁失败
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

```java
// 尝试以共享模式获取
protected final int tryAcquireShared(int unused) {
    Thread current = Thread.currentThread();
    int c = getState();
    // exclusiveCount(c) 代表低 16 位, 写锁的 state，成立说明有线程持有写锁
    // 写锁的持有者不是当前线程，则获取读锁失败，【写锁允许降级】
    if (exclusiveCount(c) != 0 && getExclusiveOwnerThread() != current)
        return -1;
    
    // 高 16 位，代表读锁的 state，共享锁分配出去的总次数
    int r = sharedCount(c);
    // 读锁是否应该阻塞
    if (!readerShouldBlock() &&	r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {	// 尝试增加读锁计数
        // 加锁成功
        // 加锁之前读锁为 0，说明当前线程是第一个读锁线程
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        // 第一个读锁线程是自己就发生了读锁重入
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            // cachedHoldCounter 设置为当前线程的 holdCounter 对象，即最后一个获取读锁的线程
            HoldCounter rh = cachedHoldCounter;
            // 说明还没设置 rh
            if (rh == null || rh.tid != getThreadId(current))
                // 获取当前线程的锁重入的对象，赋值给 cachedHoldCounter
                cachedHoldCounter = rh = readHolds.get();
            // 还没重入
            else if (rh.count == 0)
                readHolds.set(rh);
            // 重入 + 1
            rh.count++;
        }
        // 读锁加锁成功
        return 1;
    }
    // 逻辑到这 应该阻塞，或者 cas 加锁失败
    // 会不断尝试 for (;;) 获取读锁, 执行过程中无阻塞
    return fullTryAcquireShared(current);
}
// 非公平锁 readerShouldBlock 偏向写锁一些，看 AQS 阻塞队列中第一个节点是否是写锁，是则阻塞，反之不阻塞
// 防止一直有读锁线程，导致写锁线程饥饿
// true 则该阻塞, false 则不阻塞
final boolean readerShouldBlock() {
    return apparentlyFirstQueuedIsExclusive();
}
final boolean readerShouldBlock() {
    return hasQueuedPredecessors();
}
```

获取读锁失败，进入 sync.doAcquireShared(1) 流程开始阻塞，首先也是调用 addWaiter 添加节点，不同之处在于节点被设置为 Node.SHARED 模式而非 Node.EXCLUSIVE 模式，注意此时 t2 仍处于活跃状态

```java
private void doAcquireShared(int arg) {
    // 将当前线程关联到一个 Node 对象上, 模式为共享模式
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            // 获取前驱节点
            final Node p = node.predecessor();
            // 如果前驱节点就头节点就去尝试获取锁
            if (p == head) {
                // 再一次尝试获取读锁
                int r = tryAcquireShared(arg);
                // r >= 0 表示获取成功
                if (r >= 0) {
                    //【这里会设置自己为头节点，唤醒相连的后序的共享节点】
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            // 是否在获取读锁失败时阻塞      					 park 当前线程
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

如果没有成功，在 doAcquireShared 内 for (;;) 循环一次，shouldParkAfterFailedAcquire 内把前驱节点的 waitStatus 改为 -1，再 for (;;) 循环一次尝试 tryAcquireShared，不成功在 parkAndCheckInterrupt() 处 park

![](.\JUC\读写锁.png)

**t1线程： w.unlock， 写锁解锁**

```java
public void unlock() {
    // 释放锁
    sync.release(1);
}
public final boolean release(int arg) {
    // 尝试释放锁
    if (tryRelease(arg)) {
        Node h = head;
        // 头节点不为空并且不是等待状态不是 0，唤醒后继的非取消节点
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
protected final boolean tryRelease(int releases) {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    // 因为可重入的原因, 写锁计数为 0, 才算释放成功
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        setExclusiveOwnerThread(null);
    setState(nextc);
    return free;
}
```

唤醒流程 sync.unparkSuccessor，这时 t2 在 doAcquireShared 的 parkAndCheckInterrupt() 处恢复运行，继续循环，执行 tryAcquireShared 成功则让读锁计数加一

接下来 t2 调用 setHeadAndPropagate(node, 1)，它原本所在节点被置为头节点；还会检查下一个节点是否是 shared，如果是则调用 doReleaseShared() 将 head 的状态从 -1 改为 0 并唤醒下一个节点，这时 t3 在 doAcquireShared 内 parkAndCheckInterrupt() 处恢复运行，**唤醒连续的所有的共享节点**

![](.\JUC\读写锁共享锁.png)

```java
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; 
    // 设置自己为 head 节点
    setHead(node);
    // propagate 表示有共享资源（例如共享读锁或信号量），为 0 就没有资源
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        // 获取下一个节点
        Node s = node.next;
        // 如果当前是最后一个节点，或者下一个节点是【等待共享读锁的节点】
        if (s == null || s.isShared())
            // 唤醒后继节点
            doReleaseShared();
    }
}
```

```java
private void doReleaseShared() {
    // 如果 head.waitStatus == Node.SIGNAL ==> 0 成功, 下一个节点 unpark
	// 如果 head.waitStatus == 0 ==> Node.PROPAGATE
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            // SIGNAL 唤醒后继
            if (ws == Node.SIGNAL) {
                // 因为读锁共享，如果其它线程也在释放读锁，那么需要将 waitStatus 先改为 0
            	// 防止 unparkSuccessor 被多次执行
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;  
                // 唤醒后继节点
                unparkSuccessor(h);
            }
            // 如果已经是 0 了，改为 -3，用来解决传播性
            else if (ws == 0 && !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                
        }
        // 条件不成立说明被唤醒的节点非常积极，直接将自己设置为了新的 head，
        // 此时唤醒它的节点（前驱）执行 h == head 不成立，所以不会跳出循环，会继续唤醒新的 head 节点的后继节点
        if (h == head)                   
            break;
    }
}
```



### StampedLock

StampedLock：读写锁，该类自 JDK 8 加入，是为了进一步优化 ReentrantReadWriteLock **读性能**

ReadWriteLock 支持两种模式：一种是读锁，一种是写锁。而 StampedLock 支持三种模式，分别是：**写锁**、**悲观读锁**和**乐观读**。

**乐观读这个操作是无锁的**，是允许一个线程获取写锁的。

特点：

- 在使用读锁、写锁时都必须配合**戳**使用
- StampedLock **不支持条件变量**
- StampedLock **不支持重入**

基本用法

- 加解读锁：

  ```java
  long stamp = lock.readLock();
  lock.unlockRead(stamp);			// 类似于 unpark，解指定的锁
  ```

- 加解写锁：

  ```java
  long stamp = lock.writeLock();
  lock.unlockWrite(stamp);
  ```

- 乐观读，StampedLock 支持 `tryOptimisticRead()` 方法，读取完毕后做一次**戳校验**，如果校验通过，表示这期间没有其他线程的写操作，数据可以安全使用，如果校验没通过，需要重新获取读锁，保证数据一致性

  ```java
  long stamp = lock.tryOptimisticRead();
  // 验戳
  if(!lock.validate(stamp)){
  	// 锁升级
  }
  ```

注意：

使用 StampedLock 一定不要调用中断操作，如果需要支持中断功能，一定使用可中断的悲观读锁 readLockInterruptibly() 和写锁 writeLockInterruptibly()，否则会导致 CPU 飙升。

```java
class Point {
  private int x, y;
  final StampedLock sl = 
    new StampedLock();
  // 计算到原点的距离  
  int distanceFromOrigin() {
    // 乐观读
    long stamp = 
      sl.tryOptimisticRead();
    // 读入局部变量，
    // 读的过程数据可能被修改
    int curX = x, curY = y;
    // 判断执行读操作期间，
    // 是否存在写操作，如果存在，
    // 则 sl.validate 返回 false
    if (!sl.validate(stamp)){
      // 升级为悲观读锁
      stamp = sl.readLock();
      try {
        curX = x;
        curY = y;
      } finally {
        // 释放悲观读锁
        sl.unlockRead(stamp);
      }
    }
    return Math.sqrt(
      curX * curX + curY * curY);
  }
}
```

在上面这个代码示例中，如果执行乐观读操作的期间，存在写操作，会把乐观读升级为悲观读锁。这个做法挺合理的，否则你就需要在一个循环里反复执行乐观读，直到执行乐观读操作的期间没有写操作（只有这样才能保证 x 和 y 的正确性和一致性），而循环读会浪费大量的 CPU。



### Semaphore

信号量，用来限制能同时访问共享资源的线程上限。

初始化state一个值，如果来了一个线程则state-1，如果state的值小于 0，则当前线程将被阻塞，否则当前线程可以继续执行。

当一个线程执行完毕后将state+1，并唤醒阻塞队列中的一个等待线程。

构造方法：

- `public Semaphore(int permits)`：permits 表示许可线程的数量（state）
- `public Semaphore(int permits, boolean fair)`：fair 表示公平性，如果设为 true，下次执行的线程会是等待最久的线程

常用API：

- `public void acquire()`：表示获取许可
- `public void release()`：表示释放许可，acquire() 和 release() 方法之间的代码为同步代码

```java
public static void main(String[] args) {
    // 1.创建Semaphore对象
    Semaphore semaphore = new Semaphore(3);

    // 2. 10个线程同时运行
    for (int i = 0; i < 10; i++) {
        new Thread(() -> {
            try {
                // 3. 获取许可
                semaphore.acquire();
                sout(Thread.currentThread().getName() + " running...");
                Thread.sleep(1000);
                sout(Thread.currentThread().getName() + " end...");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                // 4. 释放许可
                semaphore.release();
            }
        }).start();
    }
}
```



**实现原理**

Semaphore 的 permits（state）为 3，这时 5 个线程来获取资源

```java
Sync(int permits) {
    setState(permits); // 实际就是使用AQS的state
}
```

调用`semaphore.acquire()` ，线程尝试获取许可证，如果 `state >= 0` 的话，则表示可以获取成功。如果获取成功的话，使用 CAS 操作去修改 `state` 的值 `state=state-1`。如果 `state<0` 的话，则表示许可证数量不足。此时会创建一个 Node 节点加入阻塞队列，挂起当前线程。

调用`semaphore.release();` ，线程尝试释放许可证，并使用 CAS 操作去修改 `state` 的值 `state=state+1`。释放许可证成功之后，同时会唤醒同步队列中的一个线程。被唤醒的线程会重新尝试去修改 `state` 的值 `state=state-1` ，如果 `state>=0` 则获取令牌成功，否则重新进入阻塞队列，挂起线程。



### CountDownLatch

`CountDownLatch` 允许 `count` 个线程阻塞在一个地方，直至所有线程的任务都执行完毕。

`CountDownLatch` 是一次性的，计数器的值只能在构造方法中初始化一次，之后没有任何机制再次对其设置值，当 `CountDownLatch` 使用完毕后，它不能再次被使用。



构造器：

- `public CountDownLatch(int count)`：初始化唤醒需要的 down 几步

常用API：

- `public void await() `：让当前线程等待，必须 down 完初始化的数字才可以被唤醒，否则进入无限等待
- `public void countDown()`：计数器进行减 1（down 1）

应用：等待多线程准备完毕、同步等待多个 Rest 远程调用结束

```java
AtomicInteger num = new AtomicInteger(0);

ExecutorService service = Executors.newFixedThreadPool(10, (r) -> {
    return new Thread(r, "t" + num.getAndIncrement());
});

CountDownLatch latch = new CountDownLatch(10);

String[] all = new String[10];
Random r = new Random();
for (int j = 0; j < 10; j++) {
    int x = j;
    service.submit(() -> {
        for (int i = 0; i <= 100; i++) {
            try {
                Thread.sleep(r.nextInt(100));
            } catch (InterruptedException e) {
            }
            all[x] = Thread.currentThread().getName() + "(" + (i + "%") + ")";
            System.out.print("\r" + Arrays.toString(all));
        }
        latch.countDown();
    });
}

latch.await();
System.out.println("\n游戏开始...");
service.shutdown();

/*
中间输出
[t0(52%), t1(47%), t2(51%), t3(40%), t4(49%), t5(44%), t6(49%), t7(52%), t8(46%), t9(46%)] 
最后输出
[t0(100%), t1(100%), t2(100%), t3(100%), t4(100%), t5(100%), t6(100%), t7(100%), t8(100%), t9(100%)] 
游戏开始... 
*/
```



**实现原理**

`CountDownLatch` 是共享锁的一种实现,它默认构造 AQS 的 `state` 值为 `count`。

当线程使用 `countDown()` 方法时,其实使用了`tryReleaseShared`方法以 CAS 的操作来减少 `state`,直至 `state` 为 0 。

当调用 `await()` 方法的时候，如果 `state` 不为 0，那就证明任务还没有执行完毕，`await()` 方法就会一直阻塞。直到`count` 个线程调用了`countDown()`使 state 值被减为 0，或者调用`await()`的线程被中断，该线程才会从阻塞中被唤醒，`await()` 方法之后的语句得到执行。



### CyclicBarrier

CyclicBarrier：循环屏障，用来进行线程协作，等待线程满足某个计数，才能触发自己执行.

`CountDownLatch` 的实现是基于 AQS 的，而 `CycliBarrier` 是基于 `ReentrantLock`(`ReentrantLock` 也属于 AQS 同步器)和 `Condition` 的。

首先设置了屏障数量，当线程调用 await 的时候计数器会减一，如果计数器不等于0的时候，线程会调用 condition.await 进行阻塞等待。如果计数器值等于0，调用 condition.signalAll 唤醒等待的线程，并且重置计数器，然后开启下一代。

常用方法：

- `public CyclicBarrier(int parties, Runnable barrierAction)`：用于在线程到达屏障 parties 时，执行 barrierAction
  - parties：代表多少个线程到达屏障开始触发线程任务
  - barrierAction：线程任务
- `public int await()`：线程调用 await 方法通知 CyclicBarrier 本线程已经到达屏障

与 CountDownLatch 的区别：

1. CountDownLatch 是一个线程阻塞等待其他线程到达一个节点之后才能继续执行，这个过程其他线程不会阻塞；CyclicBarrier是各个线程阻塞等待所有线程都达到一个节点后，所有线程继续执行。

2. CyclicBarrier 是可以重用的

应用：

可以实现多线程中，某个任务在等待其他线程执行完毕以后触发

```java
public static void main(String[] args) {
    ExecutorService service = Executors.newFixedThreadPool(2);
    CyclicBarrier barrier = new CyclicBarrier(2, () -> {
        System.out.println("task1 task2 finish...");
    });

    for (int i = 0; i < 3; i++) { // 循环重用
        service.submit(() -> {
            System.out.println("task1 begin...");
            try {
                Thread.sleep(1000);
                barrier.await();    // 2 - 1 = 1
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
        });

        service.submit(() -> {
            System.out.println("task2 begin...");
            try {
                Thread.sleep(2000);
                barrier.await();    // 1 - 1 = 0
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
        });
    }
    service.shutdown();
}
```



### CompletableFuture

更简洁的编写异步代码。如果任务之间有聚合关系，无论是 AND 聚合还是 OR 聚合，都可以通过 CompletableFuture 来解决

#### CompletableFuture 对象

```java
// 使用默认线程池 
// Runnable 接口的 run() 方法没有返回值，而 Supplier 接口的 get() 方法是有返回值的。
static CompletableFuture<Void> runAsync(Runnable runnable)
static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier)
// 可以指定线程池参数
static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor)
static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor)  
```

默认情况下 CompletableFuture 会使用公共的 ForkJoinPool 线程池，这个线程池默认创建的线程数是 CPU 的核数。

创建完 CompletableFuture 对象之后，会自动地异步执行 runnable.run() 方法或者 supplier.get() 方法。

#### CompletionStage 接口

CompletableFuture 类还实现了 CompletionStage 接口。任务是有时序关系的，比如有**串行关系、并行关系、汇聚关系**等，CompletionStage 接口可以清晰地描述任务之间的时序关系。

**串行关系**

主要是 thenApply、thenAccept、thenRun 和 thenCompose 这四个系列的接口。

```java
CompletionStage<R> thenApply(fn);
CompletionStage<R> thenApplyAsync(fn);
CompletionStage<Void> thenAccept(consumer);
CompletionStage<Void> thenAcceptAsync(consumer);
CompletionStage<Void> thenRun(action);
CompletionStage<Void> thenRunAsync(action);
CompletionStage<R> thenCompose(fn);
CompletionStage<R> thenComposeAsync(fn);
```

thenApply 系列函数里参数 fn 的类型是接口 Function<T, R>，这个接口里与 CompletionStage 相关的方法是 `R apply(T t)`，这个方法既能接收参数也支持返回值，所以 thenApply 系列方法返回的是`CompletionStage<R>`。

而 thenAccept 系列方法里参数 consumer 的类型是接口`Consumer<T>`，这个接口里与 CompletionStage 相关的方法是 `void accept(T t)`，这个方法虽然支持参数，但却不支持回值，所以 thenAccept 系列方法返回的是`CompletionStage<Void>`。

thenRun 系列方法里 action 的参数是 Runnable，所以 action 既不能接收参数也不支持返回值，所以 thenRun 系列方法返回的也是`CompletionStage<Void>`。

thenCompose 系列方法，这个系列的方法会新创建出一个子流程，最终结果和 thenApply 系列是相同的。

```java
CompletableFuture<String> f0 = 
  CompletableFuture.supplyAsync(
    () -> "Hello World")      //①
  .thenApply(s -> s + " QQ")  //②
  .thenApply(String::toUpperCase);//③
 
System.out.println(f0.join());
// 输出结果 任务①②③却是串行执行的，②依赖①的执行结果，③依赖②的执行结果。
HELLO WORLD QQ
```

**AND 汇聚关系**

主要是 thenCombine、thenAcceptBoth 和 runAfterBoth 系列的接口，这些接口的区别也是源自 fn、consumer、action 这三个核心参数不同

```java
CompletionStage<R> thenCombine(other, fn);
CompletionStage<R> thenCombineAsync(other, fn);
CompletionStage<Void> thenAcceptBoth(other, consumer);
CompletionStage<Void> thenAcceptBothAsync(other, consumer);
CompletionStage<Void> runAfterBoth(other, action);
CompletionStage<Void> runAfterBothAsync(other, action);
```

**OR 汇聚关系**

主要是 applyToEither、acceptEither 和 runAfterEither 系列的接口，这些接口的区别也是源自 fn、consumer、action 这三个核心参数不同。

```java
CompletionStage applyToEither(other, fn);
CompletionStage applyToEitherAsync(other, fn);
CompletionStage acceptEither(other, consumer);
CompletionStage acceptEitherAsync(other, consumer);
CompletionStage runAfterEither(other, action);
CompletionStage runAfterEitherAsync(other, action);
```

**异常处理**

```java
CompletionStage exceptionally(fn);
CompletionStage<R> whenComplete(consumer);
CompletionStage<R> whenCompleteAsync(consumer);
CompletionStage<R> handle(fn);
CompletionStage<R> handleAsync(fn);
```

exceptionally() 的使用非常类似于 try{}catch{}中的 catch{}，但是由于支持链式编程方式，所以相对更简单。

whenComplete() 和 handle() 系列方法就类似于 try{}finally{}中的 finally{}，无论是否发生异常都会执行 whenComplete() 中的回调函数 consumer 和 handle() 中的回调函数 fn。whenComplete() 和 handle() 的区别在于 whenComplete() 不支持返回结果，而 handle() 是支持返回结果的。

```java
CompletableFuture<Integer> 
  f0 = CompletableFuture
    .supplyAsync(()->7/0))
    .thenApply(r->r*10)
    .exceptionally(e->0);
System.out.println(f0.join());
```



#### 使用建议

1. 使用自定义线程池：

```java
private ThreadPoolExecutor executor = new ThreadPoolExecutor(10, 10,
        0L, TimeUnit.MILLISECONDS,
        new LinkedBlockingQueue<Runnable>());

CompletableFuture.runAsync(() -> {
     //...
}, executor);
```

2. `CompletableFuture`的`get()`方法是阻塞的，尽量避免使用。如果必须要使用的话，需要添加超时时间，否则可能会导致主线程一直等待，无法执行其他任务。
3. 正确进行异常处理
4. 合理组合多个任务

[asyncTool: 解决任意的多线程并行、串行、阻塞、依赖、回调的并行框架，可以任意组合各线程的执行顺序，带全链路执行结果回调。多线程编排一站式解决方案。来自于京东主App后台。 (gitee.com)](https://gitee.com/jd-platform-opensource/asyncTool)

### CompletionService

CompletionService 的实现原理也是内部维护了一个阻塞队列，当任务执行结束就把任务的执行结果的 Future 对象加入到阻塞队列中.

CompletionService的实现目标是**任务先完成可优先获取到，即结果按照完成先后顺序排序。**

CompletionService 接口的实现类是 ExecutorCompletionService，这个实现类的构造方法有两个，分别是：

1. `ExecutorCompletionService(Executor executor)`；
2. `ExecutorCompletionService(Executor executor, BlockingQueue<Future<V>> completionQueue)`。

这两个构造方法都需要传入一个线程池，如果不指定 completionQueue，那么默认会使用无界的 LinkedBlockingQueue

```java
public interface CompletionService<V> {
    // 提交
    Future<V> submit(Callable<V> task);
    Future<V> submit(Runnable task, V result);
    // 获取
    Future<V> take() throws InterruptedException;
    Future<V> poll();
    Future<V> poll(long timeout, TimeUnit unit) throws InterruptedException;
}
```



### 线程安全集合类

1. 遗留的线程安全集合如 Hashtable， Vector，Stack：每个方法都用 synchronized 保证线程安全

2. 使用 Collections 装饰的线程安全集合类：接收一个线程不安全集合，使用装饰器模式添加一个mutex 成员变量，本质还是方法上添加 synchronized 
3. java.util.concurrent.* 

java.util.concurrent.* 下的线程安全集合类，可以发现它们有规律，里面包含三类关键词： `Blocking、CopyOnWrite、Concurrent `

- Blocking 大部分实现基于锁，并提供用来阻塞的方法 
- CopyOnWrite 之类容器修改开销相对较重 
- Concurrent 类型的容器 
  - 内部很多操作使用 cas 优化，一般可以提供较高吞吐量 
  - 弱一致性 
    - 遍历时弱一致性，例如，当利用迭代器遍历时，如果容器发生修改，迭代器仍然可以继续进行遍历，这时内容是旧的 
    - 求大小弱一致性，size 操作未必是 100% 准确 
    - 读取弱一致性 



## ThreadLocal

实现每一个线程都有自己专属的本地变量副本来避免共享，从而避免了线程安全问题。

### **ThreadLocal 的工作原理**

ThreadLocal 的目标是让不同的线程有不同的变量 V，那最直接的方法就是创建一个 Map，它的 Key 是线程，Value 是每个线程拥有的变量 V，ThreadLocal 内部持有这样的一个 Map 就可以了。

<img src=".\JUC\ThreadLocal.png" style="zoom:67%;" />

`Thread`类有一个类型为`ThreadLocal.ThreadLocalMap`的实例变量`threadLocals`，也就是说每个线程有一个自己的`ThreadLocalMap`。

`ThreadLocalMap`有自己的独立实现，可以简单地将它的`key`视作`ThreadLocal`，`value`为代码中放入的值（实际上`key`并不是`ThreadLocal`本身，而是它的一个**弱引用**）。

每个线程在往`ThreadLocal`里放值的时候，都会往自己的`ThreadLocalMap`里存，读也是以`ThreadLocal`作为引用，在自己的`map`里找对应的`key`，从而实现了**线程隔离**。

我们还要注意`Entry`， 它的`key`是`ThreadLocal<?> k` ，继承自`WeakReference`， 也就是我们常说的弱引用类型。



### **内存泄露**

`ThreadLocalMap` 中使用的 key 为 `ThreadLocal` 的弱引用，而 value 是强引用。所以，如果 `ThreadLocal` 没有被外部强引用的情况下，在垃圾回收的时候，Entry.key 会被清理掉，而 Entry.value 不会被清理掉。

这样一来，`ThreadLocalMap` 中就会出现 key 为 null 的 Entry。假如我们不做任何措施的话，value 永远无法被 GC 回收，这个时候就可能会产生内存泄露。`ThreadLocalMap` 实现中已经考虑了这种情况，在调用 `set()`、`get()`、`remove()` 方法的时候，会清理掉 key 为 null 的记录。使用完 `ThreadLocal`方法后最好手动调用`remove()`方法

```java
ExecutorService es;
ThreadLocal tl;
es.execute(()->{
  //ThreadLocal 增加变量
  tl.set(obj);
  try {
    // 省略业务逻辑代码
  }finally {
    // 手动清理 ThreadLocal 
    tl.remove();
  }
});
```



### ThreadLocalMap Hash算法

既然是`Map`结构，那么`ThreadLocalMap`当然也要实现自己的`hash`算法来解决散列表数组冲突问题。

```java
int i = key.threadLocalHashCode & (len-1);
```

`ThreadLocalMap`中`hash`算法很简单，这里`i`就是当前 key 在散列表中对应的数组下标位置。

这里最关键的就是`threadLocalHashCode`值的计算，`ThreadLocal`中有一个属性为`HASH_INCREMENT = 0x61c88647`

```java
public class ThreadLocal<T> {
    private final int threadLocalHashCode = nextHashCode();

    private static AtomicInteger nextHashCode = new AtomicInteger();

    private static final int HASH_INCREMENT = 0x61c88647;

    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }

    static class ThreadLocalMap {
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);

            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
    }
}
```

每当创建一个`ThreadLocal`对象，这个`ThreadLocal.nextHashCode` 这个值就会增长 `0x61c88647` 。

这个值很特殊，它是**斐波那契数** 也叫 **黄金分割数**。`hash`增量为 这个数字，带来的好处就是 `hash` **分布非常均匀**。



### ThreadLocalMap Hash 冲突

`HashMap`中解决冲突的方法是在数组上构造一个**链表**结构，冲突的数据挂载到链表上，如果链表长度超过一定数量则会转化成红黑树。

而 `ThreadLocalMap` 中并没有链表结构，所以这里不能使用拉链法了，使用的是**线性探测法**。



### ThreadLocalMap.set()

**第一种情况：** 通过`hash`计算后的槽位对应的`Entry`数据为空：直接将数据放到槽位即可

**第二种情况：** 槽位数据不为空，`key`值与当前`ThreadLocal`通过`hash`计算获取的`key`值一致：直接更新该槽位的数据。

**第三种情况：** 槽位数据不为空，往后遍历过程中，在找到`Entry`为`null`的槽位之前，没有遇到`key`过期的`Entry`：	遍历散列数组，线性往后查找，如果找到`Entry`为`null`的槽位，则将数据放入该槽位中，或者往后遍历过程中，遇到了**key 值相等**的数据，直接更新即可。

**第四种情况：** 槽位数据不为空，往后遍历过程中，在找到`Entry`为`null`的槽位之前，遇到`key`过期的`Entry`：

![](.\JUC\set.png)

散列数组下标为 7 位置对应的`Entry`数据`key`为`null`，表明此数据`key`值已经被垃圾回收掉了，此时就会执行`replaceStaleEntry()`方法，进行探测式数据清理工作。

数据清理工作：

1）以当前`staleSlot`开始 向前迭代查找，找其他过期的数据，然后更新过期数据起始扫描下标`slotToExpunge`。`for`循环迭代，直到碰到`Entry`为`null`结束。`slotToExpunge`是为了更新探测清理过期数据的起始下标`slotToExpunge`的值

![](.\JUC\set12.png)

2）然后从当前节点`staleSlot`向后查找`key`值相等的`Entry`元素，找到后更新`Entry`的值并交换`staleSlot`元素的位置(`staleSlot`位置为过期元素)，更新`Entry`数据。如果没有找到相同 key 值的 Entry 数据：创建新的`Entry`，替换`table[stableSlot]`位置

![](.\JUC\set3.png)

3）然后开始进行过期`Entry`的清理工作：`expungeStaleEntry()`探测式清理和`cleanSomeSlots()`启发式清理工作

![](.\JUC\set4.png)



### 清理工作

`ThreadLocalMap`的两种过期`key`数据清理方式：**探测式清理**和**启发式清理**。

探测式清理，也就是`expungeStaleEntry`方法，遍历散列数组，向后探测清理过期数据，将过期数据的`Entry`设置为`null`，沿途中碰到未过期的数据则将此数据`rehash`后重新在`table`数组中定位，如果定位的位置已经有了数据，则会将未过期的数据放到最靠近此位置的`Entry=null`的桶中，使`rehash`后的`Entry`数据距离正确的桶的位置更近一些。

`set`和`get`到都会触发**探测式清理**操作。



启发式清理：执行logn次探测式清理



### 扩容机制

在`ThreadLocalMap.set()`方法的最后，如果执行完启发式清理工作后，未清理到任何数据，且当前散列数组中`Entry`的数量已经达到了列表的扩容阈值`(len*2/3)`，就开始执行`rehash()`逻辑：

`rehash()`首先是会进行探测式清理工作，从`table`的起始位置往后清理。清理完成之后，`table`中可能有一些`key`为`null`的`Entry`数据被清理掉，所以此时通过判断`size >= threshold - threshold / 4` 也就是`size >= threshold * 3/4` 来决定是否扩容。

扩容后的`tab`的大小为`oldLen * 2`，然后遍历老的散列表，重新计算`hash`位置，然后放到新的`tab`数组中，如果出现`hash`冲突则往后寻找最近的`entry`为`null`的槽位，遍历完成之后，`oldTab`中所有的`entry`数据都已经放入到新的`tab`中了。



### ThreadLocalMap.get()

**第一种情况：** 通过查找`key`值计算出散列表中`slot`位置，然后该`slot`位置中的`Entry.key`和查找的`key`一致，则直接返回

**第二种情况：** `slot`位置中的`Entry.key`和要查找的`key`不一致：往后遍历查找，如果时遇到`key=null`，触发一次探测式数据回收操作，执行`expungeStaleEntry()`方法，直到Entry为null或找到匹配值。



