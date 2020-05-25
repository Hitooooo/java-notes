---
title: Java并发编程(原理)
date: 2020-04-19 23:21:50
categories:
- High Concurrent
tags:
- JavaSE
- Concurrent
---

并发编程的目的是为了让程序运行得更快，但是，并不是启动更多的线程就能让程序最大限度地并发执行。因为存在上下文切换、死锁等问题

<!--more-->

## 并发挑战

1. **多线程不一定快**

    这是因为多线程需要上下文切换的支持，需要一定的资源存储线程切换时地现场信息。 如果数据量较小或计算简单，串行的耗时有可能是小于并发多线程的，那如何解决呢？

   * 无锁编程。多线程处理数据时，可以考虑将数据ID按Hash算法分段取模，不同线程处理不同的数据段。[海量数据处理常见面试题](https://mp.weixin.qq.com/s/rjGqxUvrEqJNlo09GrT1Dw)，失效从有道笔记中找
   * CAS算法。Java的Atomic包使用CAS算法来更新数据，而不需要加锁
   * 协程。在单线程里实现多任务的调度，并在单线程里面维持多个任务的切换。
   * 使用最少线程

2. **死锁处理**

   线程之间因为资源争夺问题，产生的相互等待现象。死锁产生需要四个必要条件

   1. 互斥条件：同一个资源，同一时间只能被一个进程使用
   2. 不可剥夺条件：已经拥有的资源，在使用完毕之前不会被其他线程抢夺
   3. 请求和保持条件：进程在拥有部分所需资源时，还可以继续像系统申请剩下所需的资源
   4. 循环等待条件：在死锁发生时，一定存在这样一个队列。p1等待p2持有的资源，p2等待p3持有的资源...等待p1持有的资源。

   **破坏掉上面的任意一个或多个条件，就可以预防死锁：**

   * 资源互斥是无法改变的特质，无法改变互斥条件
   * 破坏不可剥夺条件：进程只有在所有资源都可用的情况下再去运行，否则处于等待状态。哲学家只有左右都有筷子时，才会拿起筷子并就餐。
   * 破坏请求和保持：进程申请不到所需的资源时，释放自己占用的资源
   *  破坏循环等待：为资源编号，按序申请 。结合MySQL中事务之间产生的死锁及解决办法。[代码参考](https://www.jianshu.com/p/ea718226c56f)

## 基本概念

1. 进程与线程的区别
2. 线程的状态转换
3. 

## Java并发底层原理

### 前置知识点

####  Java内存结构 

{% asset_img JavaRuntimeAarea.png JavaRuntimeAarea%} 

![Java Runtime Aarea](.\Java并发编程\JavaRuntimeAarea.png)

**PS：黄色区域为所有线程共享，绿色区域为各个线程独有**

- 堆： Java堆（Java Heap）是Java虚拟机所管理的内存中***最大***的一块。Java堆是被所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例，***几乎所有的对象实例都在这里分配内存***。 
-  方法区：（Method Area）与Java堆一样，是各个线程共享的内存区域，**它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据**。 
-  运行时常量池： Class文件中除了有类的版本、字段、方法、接口等描述等信息外，还有一项信息是常量池（Constant Pool Table），用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后存放到常量池中。  运行时常量是相对于常量来说的，它具备一个重要特征是：动态性。当然，值相同的动态常量与我们通常说的常量只是来源不同，但是都是储存在池内同一块内存区域。Java语言并不要求常量一定只能在编译期产生，运行期间也可能产生新的常量，这些常量被放在运行时常量池中。这里所说的常量包括：基本类型包装类 
-  程序计数器：（Program Counter Register）是一块较小的内存空间，它的作用可以看做是当前线程所执行的字节码的行号指示器。 **可以从微机系统角度的IP指令指针理解，在JVM层面， 字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成。**
- JVM栈： 与程序计数器一样，Java虚拟机栈（Java Virtual Machine Stacks）也是线程私有的，**它的生命周期与线程相同**。**虚拟机栈描述的是Java方法执行的内存模型**：每个方法被执行的时候都会同时创建一个栈帧（Stack Frame）用于存储局部变量表、操作栈、动态链接、方法出口等信息。**每一个方法被调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中从入栈到出栈的过程**。  
- 本地方法栈： 本地方法栈（Native Method Stacks）与虚拟机栈所发挥的作用是非常相似的，其区别不过是虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，而**本地方法栈则是为虚拟机使用到的Native方法服务**。 

***由上图可知，堆内存是共享的，那么堆中内存会存在不可见问题呢？***

#### Java内存模型

>  堆中内存不可见：这是因为现代计算机为了高效，往往会在高速缓存区中缓存共享变量，因为cpu访问缓存区比访问内存要快得多。
>
> >  线程之间的共享变量存在主内存中，每个线程都有一个私有的本地内存，存储了该线程以读、写共享变量的副本。本地内存是Java内存模型的一个抽象概念，并不真实存在。它涵盖了缓存、写缓冲区、寄存器等。 

Java 线程之间的**通信由 Java 内存模型**（简称 JMM ）控制，从抽象的⻆度来说，JMM 定义了线程和主内存之间的抽象关系。 JMM 的抽象示意图如图所示：

{% asset_img JMM抽象示意图.jpg Java JMM抽象示意图%} 

 ![img](.\Java并发编程\JMM抽象示意图.jpg) 

- 主存存放线程需要操作的变量，但线程并不直接操作主存。
- 每个线程读取主存变量都是先拷贝一份到工作内存中，不同线程工作内存互不干扰。
- 线程修改了工作内存后，再写回主存中。

**所以，线程A无法直接访问线程B的工作内存，线程间通信必须经过主内存**

根据JMM的规定，**线程对共享变量的所有操作都必须在自己的本地内存中进行，不能直接从主内存中读取**。  所以线程B并不是直接去主内存中读取共享变量的值，而是先在本地内存B中找到这个共享变量，发现这个共享变量已经被更新了，然后本地内存B去主内存中读取这个共享变量的新值，并拷贝到本地内存B中，最后线程B再读取本地内存B中的新值。 

那么怎么知道这个共享变量的被其他线程更新了呢？这就是JMM的功劳了，也是JMM存在的必要性之一。**JMM通过控制主内存与每个线程的本地内存之间的交互，来提供内存可见性保证**。 

#### 重排序

>  计算机在执行程序时，为了提高性能，编译器和处理器常常会对指令做重排。 

 指令重排一般分为以下三种： 

*  编译器优化重排: 编译器在**不改变单线程程序语义**的前提下，可以重新安排语句的执行顺序。  
*  指令并行重排: 现代处理器采用了指令级并行技术来将多条指令重叠执行。如果**不存在数据依赖性**(即后一个执行的语句无需依赖前面执行的语句的结果)，处理器可以改变语句对应的机器指令的执行顺序。 
*  内存系统重排: 由于处理器使用缓存和读写缓存冲区，这使得加载(load)和存储(store)操作看上去可能是在乱序执行，因为三级缓存的存在，导致内存与缓存的数据同步存在时间差。 

 **指令重排可以保证串行语义一致，但是没有义务保证多线程间的语义也一致**。所以在多线程下，指令重排序可能会导致一些问题。 

##### JMM的保证

 **JMM的具体实现方针是：**

* **在不改变（正确同步的）程序执行结果的前提下，尽量为编译期和处理器的优化打开方便之门**。 
*  对于未同步的多线程程序，JMM只提供**最小安全性**：线程读取到的值，要么是之前某个线程写入的值，要么是默认值，不会无中生有。 

>  这里的同步包括了使用`volatile`、`final`、`synchronized`等关键字来实现**多线程下的同步**。 

 JMM没有保证未同步程序的执行结果与该程序在顺序一致性中执行结果一致。因为如果要保证执行结果一致，那么JMM需要禁止大量的优化，对程序的执行性能会产生很大的影响。 

#### happens-before

 一方面，程序员需要JMM提供一个强的内存模型来编写代码；另一方面，编译器和处理器希望JMM对它们的束缚越少越好，这样它们就可以最可能多的做优化来提高性能，希望的是一个弱的内存模型。 

 JMM考虑了这两种需求，并且找到了平衡点，对编译器和处理器来说，**只要不改变程序的执行结果（单线程程序和正确同步了的多线程程序），编译器和处理器怎么优化都行。** 

而对于程序员，JMM提供了**happens-before规则**（JSR-133规范），满足了程序员的需求——**简单易懂，并且提供了足够强的内存可见性保证。**换言之，程序员只要遵循happens-before规则，那他写的程序就能保证在JMM中具有强的内存可见性。 

 happens-before关系的定义如下： 

1.  如果一个操作happens-before另一个操作，那么第一个操作的执行结果将对第二个操作可见，而且第一个操作的执行顺序排在第二个操作之前。 
2.  两个操作之间存在happens-before关系，并不意味着Java平台的具体实现必须要按照happens-before关系指定的顺序来执行。如果重排序之后的执行结果，与按happens-before关系来执行的结果一致，那么JMM也允许这样的重排序。 

**如果操作A happens-before操作B，那么操作A在内存上所做的操作对操作B都是可见的，不管它们在不在一个线程。** 

在Java中，有以下天然的happens-before关系：

- 程序顺序规则：一个线程中的每一个操作，happens-before于该线程中的任意后续操作。
- 监视器锁规则：对一个锁的解锁，happens-before于随后对这个锁的加锁。
- volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读。
- 传递性：如果A happens-before B，且B happens-before C，那么A happens-before C。
- start规则：如果线程A执行操作ThreadB.start()启动线程B，那么A线程的ThreadB.start（）操作happens-before于线程B中的任意操作、
- join规则：如果线程A执行操作ThreadB.join（）并成功返回，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回。

### 1.volatile

volatile是一种轻量且在有限的条件下线程安全技术，它保证修饰的变量的可见性和有序性，**但非原子性**。相对于synchronize高效，而常常跟synchronize配合使用。 先来回忆下基本的概念：

内存可见性: MM有一个主内存，每个线程有自己私有的工作内存，工作内存中保存了一些变量在主内存的拷贝. 内存可见性，指的是线程之间的可见性，当一个线程修改了共享变量时，另一个线程可以读取到这个修改后的值.

重排序： 为优化程序性能，对原有的指令执行顺序进行优化重新排序。重排序可能发生在多个阶段，比如编译重排序、CPU重排序等

happens-before规则： 是一个给程序员使用的规则，只要程序员在写代码的时候遵循happens-before规则，JVM就能保证指令在多线程之间的顺序性符合程序员的预期 

#### volatile语义

* 保证变量的**内存可见性**： 当一个线程对`volatile`修饰的变量进行**写操作**时，JMM会立即把该线程对应的本地内存中的共享变量的值刷新到主内存；当一个线程对`volatile`修饰的变量进行**读操作**时，JMM会把立即该线程对应的本地内存置为无效，从主内存中读取共享变量的值。

* 禁止volatile变量与普通变量**重排序**：（JSR133提出，Java 5 开始才有这个“增强的volatile内存语义”，提供了一种比锁更轻量级的线程间通信机制。如单写多读模型

  >  编译器在**生成字节码时**，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。编译器选择了一个**比较保守的JMM内存屏障插入策略**，这样可以保证在任何处理器平台，任何程序中都能得到正确的volatile内存语义。这个策略是：
  >
  > - 在每个volatile写操作前插入一个StoreStore屏障；
  > - 在每个volatile写操作后插入一个StoreLoad屏障；
  > - 在每个volatile读操作后插入一个LoadLoad屏障；
  > - 在每个volatile读操作后再插入一个LoadStore屏障。

#### volatile与原子性

 volatile只对读或写具有原子性，本身并不对数据运算处理维持原子性，强调的是读写及时影响主存。一般的自增不是单个操作，而是读+写，volatile就无法保证自增操作的原子性。可以使用atomic包下的JDK实现数值运算的原子性。 

```java
public static class TestData {
    volatile int num = 0;
    //synchronized
    public void updateNum(){
        num++;  // idea也会提示非原子操作的警告
    }
    public static void main(String[] args) {
        final TestData testData = new TestData();
        for(int i = 1; i <= 10; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int j = 1; j <= 1000; j++) {
                        testData.updateNum();
                    }
                }
            }).start();
        }

        while (Thread.activeCount() > 2) {
            Thread.yield();
        }
        System.out.println("最终结果：" + testData.num);
    }
}
```

 你会发现运行结果并不是10000，而是小于期望值。可以多改几次j的最大值查看效果。 因为num++涉及到 read write sync，读到的值一开始时相同的，但写入的时候其他线程可能先写入了。 对于这种只能通过synchronizd关键字处理了。  只要多个线程涉及了读和写，[volatile is Not Always Enough](http://tutorials.jenkov.com/java-concurrency/volatile.html).一个写，多个读是没有问题的。 

####  volatile的用途

1.  DCL单例操作需要volatile和synchronize保证线程安全 

   ```java
   public class Singleton {
   
       private static Singleton instance; // 不使用volatile关键字
   
       // 双重锁检验
       public static Singleton getInstance() {
           if (instance == null) { // 第7行
               synchronized (Singleton.class) {
                   if (instance == null) {
                       instance = new Singleton(); // 第10行
                   }
               }
           }
           return instance;
       }
   }
   ```

    如果这里的变量声明不使用volatile关键字，是可能会发生错误的。它可能会被重排序： 

   ```
   instance = new Singleton(); // 第10行
   
   // 可以分解为以下三个步骤
   1 memory=allocate();// 分配内存 相当于c的malloc
   2 ctorInstanc(memory) //初始化对象
   3 s=memory //设置s指向刚分配的地址
   
   // 上述三个步骤可能会被重排序为 1-3-2，也就是：
   1 memory=allocate();// 分配内存 相当于c的malloc
   3 s=memory //设置s指向刚分配的地址
   2 ctorInstanc(memory) //初始化对象
   ```

   而一旦假设发生了这样的重排序，比如线程A在第10行执行了步骤1和步骤3，但是步骤2还没有执行完。这个时候线程A执行到了第7行，它会判定instance不为空，然后直接返回了一个未初始化完成的instance 

2.  status flag.A线程通知B线程值的改变。

### 2.synchronized

 Java中提供了两种实现同步的基础语义：`synchronized`方法和`synchronized`块 

```java
public class SyncTest {
    public void syncBlock(){
        synchronized (this){
            System.out.println("hello block");
        }
    }
    public synchronized void syncMethod(){
        System.out.println("hello method");
    }
}
```

```basic
{
  public void syncBlock();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter				 	  // monitorenter指令进入同步块
         4: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         7: ldc           #3                  // String hello block
         9: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        12: aload_1
        13: monitorexit						  // monitorexit指令退出同步块
        14: goto          22
        17: astore_2
        18: aload_1
        19: monitorexit						  // monitorexit指令退出同步块
        20: aload_2
        21: athrow
        22: return
      Exception table:
         from    to  target type
             4    14    17   any
            17    20    17   any
 

  public synchronized void syncMethod();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED      //添加了ACC_SYNCHRONIZED标记
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #5                  // String hello method
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
 
}
```

于`synchronized`代码块而言，`javac`在编译时，会生成对应的`monitorenter`和`monitorexit`指令分别对应`synchronized`同步块的进入和退出，有两个`monitorexit`指令的原因是：为了保证抛异常的情况下也能释放锁，所以`javac`为同步代码块添加了一个隐式的try-finally，在finally中会调用`monitorexit`命令释放锁。

而对于`synchronized`方法而言，`javac`为其生成了一个`ACC_SYNCHRONIZED`关键字，在JVM进行方法调用时，发现调用的方法被`ACC_SYNCHRONIZED`修饰，则会先尝试获得锁。 

 在JVM底层，对于这两种`synchronized`语义的实现大致相同。monitor与一个Java对象绑定，当执行命令时发现monitor时，会尝试获取对象所对应的monitor所有权，**即尝试获得对象的锁**。 

#### Java对象头

**synchronized用的锁是存在对象头里的。** 每个Java对象都有对象头。如果是非数组类型，则用2个字宽来存储对象头，如果是数组，则会用3个字宽来存储对象头。在32位处理器中，一个字宽是32位；在64位虚拟机中，一个字宽是64位。对象头的内容如下表： 

| 长度     | 内容                   | 说明                         |
| -------- | ---------------------- | ---------------------------- |
| 32/64bit | Mark Word              | 存储对象的hashCode或锁信息等 |
| 32/64bit | Class Metadata Address | 存储到对象类型数据的指针     |
| 32/64bit | Array length           | 数组的长度（如果是数组）     |

##### Mark Word

对象头里面的Mark Word的变化，体现了锁的状态

当对象状态为偏向锁时，`Mark Word`存储的是偏向的线程ID；

当状态为轻量级锁时，`Mark Word`存储的是指向线程栈中`Lock Record`的指针；

当状态为重量级锁时，`Mark Word`为指向堆中的monitor对象的指针。 

#### 锁的状态

1. 偏向锁

   Hotspot的作者经过以往的研究发现大多数情况下**锁不仅不存在多线程竞争，而且总是由同一线程多次获得**，于是引入了偏向锁。 偏向锁会偏向于第一个访问锁的线程，如果在接下来的运行过程中，该锁没有被其他的线程访问，则持有偏向锁的线程将永远不需要触发同步。也就是说，**偏向锁在资源无竞争情况下消除了同步语句，连CAS操作都不做了，提高了程序的运行性能。** 

2. 轻量级锁： 多个线程在不同时段获取同一把锁，即不存在锁竞争的情况，也就没有线程阻塞。针对这种情况，JVM采用轻量级锁来避免线程的阻塞与唤醒。 

   加锁： 如果一个线程获得锁的时候发现是轻量级锁，会把锁的Mark Word复制到自己的Displaced Mark Word里面。  然后线程尝试用CAS将锁的Mark Word替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失败，表示Mark Word已经被替换成了其他线程的锁记录，说明在与其它线程竞争锁，当前线程就尝试使用自旋（循环获取锁）来获取锁。  

   自旋是需要消耗CPU的，如果一直获取不到锁的话，那该线程就一直处在自旋状态，白白浪费CPU资源。  DK采用了更聪明的方式——**适应性自旋**，简单来说就是线程如果自旋成功了，则下次自旋的次数会更多，如果自旋失败了，则自旋的次数就会减少。  

   自旋也不是一直进行下去的，如果自旋到一定程度（和JVM、操作系统相关），依然没有获取到锁，称为自旋失败，那么这个线程会阻塞。同时这个锁就会**升级成重量级锁**。 

3. 重量级锁

   重量级锁依赖于操作系统的互斥量（mutex） 实现的，而操作系统中线程间状态的转换需要相对比较长的时间，所以重量级锁效率很低，但被阻塞的线程不会消耗CPU。 

   前面说到，每一个对象都可以当做一个锁，当多个线程同时请求某个对象锁时，对象锁会设置几种状态用来区分请求的线程 ：

   * Contention List：所有请求锁的线程将被首先放置到该竞争队列
   * Entry List：Contention List中那些有资格成为候选人的线程被移到Entry List
   * Wait Set：那些调用wait方法被阻塞的线程被放置到Wait Set
   * OnDeck：任何时刻最多只能有一个线程正在竞争锁，该线程称为OnDeck
   * Owner：获得锁的线程称为Owner
   * !Owner：释放锁的线程

   当一个线程尝试获得锁时，如果该锁已经被占用，则会将该线程封装成一个`ObjectWaiter`对象插入到**Contention List**的队列的队首，然后调用`park`函数挂起当前线程。

   当线程释放锁时，会从Contention List或EntryList中挑选一个线程唤醒，被选中的线程叫做`Heir presumptive`即假定继承人，假定继承人被唤醒后会尝试获得锁，但`synchronized`是非公平的，所以假定继承人不一定能获得锁。这是因为对于重量级锁，线程先自旋尝试获得锁，这样做的目的是为了减少执行操作系统同步操作带来的开销。如果自旋不成功再进入等待队列。这对那些已经在等待队列中的线程来说，稍微显得不公平，还有一个不公平的地方是自旋线程可能会抢占了Ready线程的锁。

   如果线程获得锁后调用`Object.wait`方法，则会将线程加入到WaitSet中，当被`Object.notify`唤醒后，会将线程从WaitSet移动到Contention List或EntryList中去。需要注意的是，当调用一个锁对象的`wait`或`notify`方法时，**如当前锁的状态是偏向锁或轻量级锁则会先膨胀成重量级锁**。

#### 锁的升级过程

每一个线程在准备获取共享资源时： 第一步，检查MarkWord里面是不是放的自己的ThreadId ,如果是，表示当前线程是处于 “偏向锁” 。

第二步，如果MarkWord不是自己的ThreadId，锁升级，这时候，用CAS来执行切换，新的线程根据MarkWord里面现有的ThreadId，通知之前线程暂停，之前线程将Markword的内容置为空。

第三步，两个线程都把锁对象的HashCode复制到自己新建的用于存储锁的记录空间，接着开始通过CAS操作， 把锁对象的MarKword的内容修改为自己新建的记录空间的地址的方式竞争MarkWord。

第四步，第三步中成功执行CAS的获得资源，失败的则进入自旋 。

第五步，自旋的线程在自旋过程中，成功获得资源(即之前获的资源的线程执行完成并释放了共享资源)，则整个状态依然处于 轻量级锁的状态，如果自旋失败 。

第六步，进入重量级锁的状态，这个时候，自旋的线程进行阻塞，等待之前线程执行完成并唤醒自己。

### 3.原子操作

原子本意不可进一步分割的粒子，而原子操作意为不可中断的一个或一系列操作。

在Java中使用锁和循环CAS实现原子操作

#### CAS

CAS的全称是：比较并交换（Compare And Swap）。在CAS中，有这样三个值：

- V：要更新的变量(var)
- E：预期值(expected)
- N：新值(new)

判断V是否等于E，如果等于，将V的值设置为N；如果不等，说明已经有其它线程更新了V，则当前线程放弃更新，什么都不做。 那有没有可能我在判断了V=E之后，正准备更新它的新值的时候，被其它线程更改了V的值呢？不会的。因为CAS是一种原子操作，它是一种系统原语，是一条CPU的原子指令，从CPU层面保证它的原子性

**当多个线程同时使用CAS操作一个变量时，只有一个会胜出，并成功更新，其余均会失败，但失败的线程并不会被挂起，仅是被告知失败，并且允许再次尝试，当然也允许失败的线程放弃操作.**

#### Java实现CAS原理-Unsafe类

 这里我们以`AtomicInteger`类的`getAndAdd(int delta)`方法为例，来看看Java是如何实现原子操作的。 

```java
// 真正依赖的实现
private static final jdk.internal.misc.Unsafe U = jdk.internal.misc.Unsafe.getUnsafe();
// 参数中offset，用于获取某个字段相对Java对象的“起始地址”的偏移量
private static final long VALUE = U.objectFieldOffset(AtomicInteger.class, "value");

public final int getAndAdd(int delta) {
    return U.getAndAddInt(this, VALUE, delta);
}
// Unsafe中被调用的实现
@HotSpotIntrinsicCandidate
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        // 获取原值，然后在原值基础上更新，更新失败更新原值后再次尝试更新
        // 一直更新失败一直尝试更新
        v = getIntVolatile(o, offset);
    } while (!weakCompareAndSetInt(o, offset, v, v + delta));
    return v;
}
// 循环条件
public final boolean weakCompareAndSetInt(Object o, long offset,
                                          int expected,
                                          int x) {
    return compareAndSetInt(o, offset, expected, x);
}
// 最终还是native方法
public final native boolean compareAndSetInt(Object o, long offset,
                                             int expected,
                                             int x);
```

 看到它是在不断尝试去用CAS更新。如果更新失败，就继续重试。那为什么要把获取“旧值”v的操作放到循环体内呢？其实这也很好理解。前面我们说了，CAS如果旧值V不等于预期值E，它就会更新失败。说明旧的值发生了变化。那我们当然需要返回的是被其他线程改变之后的旧值了，因此放在了do循环体内。 

#### CAS带来的三大问题

1. ABA问题

    所谓ABA问题，就是一个值原来是A，变成了B，又变回了A。这个时候使用CAS是检查不出变化的，但实际上却被更新了两次。 

   ABA问题的解决思路是在变量前面追加上**版本号或者时间戳**。从JDK 1.5开始，JDK的atomic包里提供了一个类`AtomicStampedReference`类来解决ABA问题。

   这个类的`compareAndSet`方法的作用是首先检查当前引用是否等于预期引用，并且检查当前标志是否等于预期标志，如果二者都相等，才使用CAS设置为新的值和标志。

2. 循环时间长开销大 

   CAS多与自旋结合。如果自旋CAS长时间不成功，会占用大量的CPU资源。

   解决思路是让JVM支持处理器提供的**pause指令**。

   pause指令能让自旋失败时cpu睡眠一小段时间再继续自旋，从而使得读操作的频率低很多,为解决内存顺序冲突而导致的CPU流水线重排的代价也会小很多。

3.  只能保证一个共享变量的原子操作 

   * 使用JDK 1.5开始就提供的`AtomicReference`类保证对象之间的原子性，把多个变量放到一个对象里面进行CAS操作； 
* 使用锁。锁内的临界区代码可以保证只有当前线程能操作。 

### 4. AQS(AbstractQueuedSynchronizer)

AQS是一个用来构建锁和同步器的框架，使用AQS能简单且高效地构造出应用广泛的同步器，比如我们提到的ReentrantLock，Semaphore，ReentrantReadWriteLock，SynchronousQueue，FutureTask等等皆是基于AQS的。  我们自己也能利用AQS非常轻松容易地构造出符合我们自己需求的同步器，只要子类实现它的几个`protected`方法即可。

#### 数据结构

 AQS内部使用了一个volatile的变量state来作为资源的标识。同时定义了几个获取和改版state的protected方法，子类可以覆盖这些方法来实现自己的逻辑： 

```java
getState()
setState()
compareAndSetState()
```

 这三种均是原子操作，其中compareAndSetState的实现依赖于Unsafe的compareAndSwapInt()方法。 

 AQS类本身实现的是一些排队和阻塞的机制，比如具体线程等待队列的维护（如获取资源失败入队/唤醒出队等）。它内部使用了一个先进先出（FIFO）的双端队列，并使用了两个指针head和tail用于标识队列的头部和尾部。其数据结构如图： 

{% asset_img AQS数据结构.png AQS数据结构%}

 ![img](.\Java并发编程\AQS数据结构.png) 

 AQS依赖这个双向队列来完成同步状态的管理。如果当前线程获取同步状态失败，AQS将会将当前线程以及等待状态信息构建成一个节点（Node）并将其加入到同步队列中，同时会阻塞当前线程。当同步状态释放时，会把首节点中的线程唤醒，使其再次获取同步状态。 在CHL中节点（Node）用来保存获取同步状态失败的线程（thread）、等待状态（waitStatus）、前驱节点（prev）和后继节点（next）。 AQS中内部维护的Node节点源码如下： 

```java
static final class Node {
    // 标记一个结点（对应的线程）在共享模式下等待
    static final Node SHARED = new Node();
    // 标记一个结点（对应的线程）在独占模式下等待
    static final Node EXCLUSIVE = null; 

    // waitStatus的值，表示该结点（对应的线程）已被取消
    static final int CANCELLED = 1; 
    // waitStatus的值，表示后继结点（对应的线程）需要被唤醒
    static final int SIGNAL = -1;
    // waitStatus的值，表示该结点（对应的线程）在等待某一条件
    static final int CONDITION = -2;
    /*waitStatus的值，表示有资源可用，新head结点需要继续唤醒后继结点（共享模式下，多线程并发释放资源，而head唤醒其后继结点后，需要把多出来的资源留给后面的结点；设置新的head结点时，会继续唤醒其后继结点）*/
    static final int PROPAGATE = -3;

    // 等待状态，取值范围，-3，-2，-1，0，1
    volatile int waitStatus;
    volatile Node prev; // 前驱结点
    volatile Node next; // 后继结点
    volatile Thread thread; // 结点对应的线程
    Node nextWaiter; // 等待队列里下一个等待条件的结点


    // 判断共享模式的方法
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    // 其它方法忽略，可以参考具体的源码
}

// AQS里面的addWaiter私有方法
private Node addWaiter(Node mode) {
    // 快速尝试添加尾节点，如果失败则调用enq(Node node)方法设置尾节点
    Node pred = tail;
    // 判断tail节点是否为空，不为空则添加节点到队列中
    if (pred != null) {
        node.prev = pred;
        // CAS设置尾节点
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}

/**
 * 插入节点到队列中.因为需要快速插入到队列尾部，也就解释了为什么是双向队列。
 * @param node the node to insert
 * @return node's predecessor
 */
private Node enq(final Node node) {
    // 死循环 知道将节点插入到队列中为止
    for (;;) {
        Node t = tail;
        // 如果队列为空，则首先添加一个空节点到队列中.使用了一个CAS方法（compareAndSetTail(pred, node)）,这里使用CAS的原因是防止在并发添加尾节点的时候出现线程不安全的问题（即有可能出现遗漏节点的情况）。
        if (t == null) {
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            // tail 不为空，则CAS设置尾节点
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

#### 源码分析

AQS的设计是基于**模板方法模式**的，它有一些方法必须要子类去实现的，它们主要有：

- isHeldExclusively()：该线程是否正在独占资源。只有用到condition才需要去实现它。
- tryAcquire(int)：独占方式。尝试获取资源，成功则返回true，失败则返回false。
- tryRelease(int)：独占方式。尝试释放资源，成功则返回true，失败则返回false。
- tryAcquireShared(int)：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
- tryReleaseShared(int)：共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。

##### 获取资源

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

{% asset_img acquire流程.jpg acquire流程%}

 ![img](.\Java并发编程\acquire流程.jpg) 

##### 释放资源

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

private void unparkSuccessor(Node node) {
    // 如果状态是负数，尝试把它设置为0
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
    // 得到头结点的后继结点head.next
    Node s = node.next;
    // 如果这个后继结点为空或者状态大于0
    // 通过前面的定义我们知道，大于0只有一种可能，就是这个结点已被取消
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 等待队列中所有还有用的结点，都向前移动
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    // 如果后继结点不为空，
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

