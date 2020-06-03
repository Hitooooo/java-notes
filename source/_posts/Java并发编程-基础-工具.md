---
title: Java并发编程-基础&工具
date: 2020-05-25 23:06:04
categories:
- High Concurrent
tags:
- JavaSE
- Concurrent
- Atomic
- Thread
---

Java并发基础知识、面试题以及Concurrent包下工具使用。
<!--more-->

## 基础知识

### 概念

1. 进程与线程的区别

    进程是操作系统进行资源分配的基本单位，而线程是操作系统进行调度的基本单位.

   * 进程单独占有一定的内存地址空间，所以进程间存在内存隔离，数据是分开的，**数据共享复杂**但是同步简单，各个进程之间互不干扰；而线程共享所属进程占有的内存地址空间和资源，**数据共享简单**，但是同步复杂。
   * 进程单独占有一定的内存地址空间，一个进程出现问题**不会影响其他进程**，不影响主程序的稳定性，可靠性高；**一个线程崩溃可能影响整个程序的稳定性**，可靠性较低。  
   * 进程单独占有一定的内存地址空间，进程的创建和销毁不仅需要保存寄存器和栈信息，**还需要资源的分配回收以及页调度，开销较大**；线程只需要保存寄存器和栈信息，开销较小。 

2. 上下文切换

   上下文切换（有时也称做进程切换或任务切换）是指 CPU 从一个进程（或线程）切换到另一个进程（或线程）。上下文是指**某一时间点 CPU 寄存器和程序计数器的内容。** 

### 入门类及接口

1.  Thread类与Runnable接口的比较 

   * 由于Java“单继承，多实现”的特性，Runnable接口使用起来比Thread更灵活。

   * Runnable接口出现更符合面向对象，将线程单独进行对象的封装。

   * Runnable接口出现，降低了线程对象和线程任务的耦合性。

   * 如果使用线程时不需要使用Thread类的诸多方法，显然使用Runnable接口更为轻量。

    我们通常优先使用“实现`Runnable`接口”这种方式来自定义线程类。 

2.  Callable、Future与FutureTask 

    `Callable`与`Runnable`类似，同样是只有一个抽象方法的函数式接口。不同的是，`Callable`提供的方法是**有返回值**的，而且支持**泛型**。  `Callable`一般是配合线程池工具`ExecutorService`来使用的,`ExecutorService`可以使用`submit`方法来让一个`Callable`接口执行。它会返回一个`Future`，我们后续的程序可以通过这个`Future`的`get`方法得到结果。 

   ```java
   public abstract interface Future<V> {
       public abstract boolean cancel(boolean paramBoolean);
       public abstract boolean isCancelled();
       public abstract boolean isDone();
       public abstract V get() throws InterruptedException, ExecutionException;
       public abstract V get(long paramLong, TimeUnit paramTimeUnit)
               throws InterruptedException, ExecutionException, TimeoutException;
   }
   ```

3.  Thread类的几个常用方法

   * currentThread()：静态方法，返回对当前正在执行的线程对象的引用；

   * start()：开始执行线程的方法，java虚拟机会调用线程内的run()方法；

   * yield()：yield在英语里有放弃的意思，同样，这里的yield()指的是当前线程愿意让出对当前处理器的占用。这里需要注意的是，就算当前线程调用了yield()方法，程序在调度的时候，也还有可能继续运行这个线程的；

   * sleep()：静态方法，使当前线程睡眠一段时间，不释放锁；

   * join()：使当前线程等待另一个线程执行完毕之后再继续执行，内部调用的是Object类的wait方法实现的；

   Object类中与线程有关的方法：

   *  **wait()**  ：让执行的当前线程休息一下，后面需要再唤醒。释放了该Object的monitor锁 

     - 线程拥有monitor锁才能用wait方法，执行后会放弃锁
     - 必须在synchronize修饰的方法或代码块中

   *  **notify(), notifyAll()** :  唤醒该Object上wait的线程， notify是随机唤醒一个，notifyAll唤醒所有。 

      必须在synchronize修饰的方法或代码块中，拥有该monitor，才能执行notify  **注意： 因为进行唤醒的方法notify执行完后，还持有monitor，所以被唤醒wait的线程不能立即获取锁，状态从waiting变成了blocked** 

4.  wait和notify为什么定义在Object中，sleep定义在Thread中？ 

    因为wait和notify涉及到锁的获取释放，锁是对象的。经常会有线程持有多个对象的锁来进行配合。 

5.  wait和sleep方法的异同 

   同: 都可以使线程阻塞进入waiting或timedwaiting状态； 都可以响应中断，并抛出异常并清除中断状态

   异: wait需要在同步代码块中；wait可以不用参数永久等待，直到被中断或唤醒notify；wait会释放monitor锁，sleep不会释放锁；wait，notify是Object方法， sleep是Thread方法

###  线程状态转换

```java
// Thread.State 源码
public enum State {
    NEW,
    RUNNABLE,
    BLOCKED,
    WAITING,
    TIMED_WAITING,
    TERMINATED;
}
```

1. NEW

    处于NEW状态的线程此时尚未启动。这里的尚未启动指的是还没调用Thread实例的start()方法。 

   * **反复调用同一个线程的start()方法是否可行？**

   * **假如一个线程执行完毕（此时处于TERMINATED状态），再次调用这个线程的start()方法是否可行？**

   > 两个问题的答案都是不可行，在调用一次start()之后，threadStatus的值会改变（threadStatus !=0），此时再次调用start()方法会抛出IllegalThreadStateException异常。 

2. RUNNABLE

    表示当前线程正在运行中 

3. BLOCKED

    阻塞状态。处于BLOCKED状态的线程正**等待锁的释放**以进入同步区。 

4. WAITING

   等待状态。处于等待状态的线程变成RUNNABLE状态需要其他线程唤醒，再次参与锁竞争后才能执行 

   调用如下3个方法会使线程进入等待状态：

   - Object.wait()：使当前线程处于等待状态直到另一个线程唤醒它；
   - Thread.join()：等待线程执行完毕，底层调用的是Object实例的wait方法；
   - LockSupport.park()：除非获得调用许可，否则禁用当前线程进行线程调度。

5. TIMED_WAITING

   超时等待状态。线程等待一个具体的时间，时间到后会被自动唤醒。 

   调用如下方法会使线程进入超时等待状态：

   - Thread.sleep(long millis)：使当前线程睡眠指定时间；
   - Object.wait(long timeout)：线程休眠指定时间，等待期间可以通过notify()/notifyAll()唤醒；
   - Thread.join(long millis)：等待当前线程最多执行millis毫秒，如果millis为0，则会一直执行；
   - LockSupport.parkNanos(long nanos)： 除非获得调用许可，否则禁用当前线程进行线程调度指定时间；
   - LockSupport.parkUntil(long deadline)：同上，也是禁止线程进行调度指定时间；

6. TERMINATED

    终止状态。此时线程已执行完毕。 

### 线程间通讯

1.  锁与同步 

2.  等待/通知机制 

3. 信号量（eg： volatile 读写锁）

4. 管道

    管道是基于“管道流”的通信方式。JDK提供了`PipedWriter`、 `PipedReader`、 `PipedOutputStream`、 `PipedInputStream`。其中，前面两个是基于字符的，后面两个是基于字节流的。 

## 工具

#### 线程池

使用线程池主要有以下三个原因：

1. 创建/销毁线程需要消耗系统资源，线程池可以**复用已创建的线程**。
2. **控制并发的数量**。并发数量过多，可能会导致资源消耗过多，从而造成服务器崩溃。（主要原因）
3. **可以对线程做统一管理**。

##### 线程池原理

1. 构造方法

   ```java
   // 最长的七个参数的构造函数
   public ThreadPoolExecutor(int corePoolSize,
                             int maximumPoolSize,
                             long keepAliveTime,
                             TimeUnit unit,
                             BlockingQueue<Runnable> workQueue,
                             ThreadFactory threadFactory,
                             RejectedExecutionHandler handler)
   ```

   必须参数：

   *  **int corePoolSize**：该线程池中**核心线程数最大值** 
   *  **int maximumPoolSize**：该线程池中**线程总数最大值**  
   *  **long keepAliveTime**：**非核心线程闲置超时时长** 
   *  **TimeUnit unit**：keepAliveTime的单位 
   *  **BlockingQueue workQueue**：阻塞队列，维护着**等待执行的Runnable任务对象** 

   非必须参数：

   *  **ThreadFactory threadFactory:**  创建线程的工厂 ，用于批量创建线程，统一在创建线程时设置一些参数，如是否守护线程、线程的优先级等。如果不指定，会新建一个默认的线程工厂。 

   *  **RejectedExecutionHandler handler: ** 

     **拒绝处理策略**，线程数量大于最大线程数就会采用拒绝处理策略，四种拒绝处理的策略为 ：

     1. **ThreadPoolExecutor.AbortPolicy**：**默认拒绝处理策略**，丢弃任务并抛出RejectedExecutionException异常。
     2. **ThreadPoolExecutor.DiscardPolicy**：丢弃新来的任务，但是不抛出异常。
     3. **ThreadPoolExecutor.DiscardOldestPolicy**：丢弃队列头部（最旧的）的任务，然后重新尝试执行程序（如果再次失败，重复此过程）。
     4. **ThreadPoolExecutor.CallerRunsPolicy**：由调用线程处理该任务。

2.  ThreadPoolExecutor的策略

   线程池本身有一个调度线程，这个线程就是用于管理布控整个线程池里的各种任务和事务，例如创建线程、销毁线程、任务队列管理、线程队列管理等等。

   故线程池也有自己的状态。`ThreadPoolExecutor`类中定义了一个`volatile int`变量**runState**来表示线程池的状态 ，分别为RUNNING、SHURDOWN、STOP、TIDYING 、TERMINATED。

3.  线程池主要的任务处理流程 

   ```java
   // JDK 1.8 
   public void execute(Runnable command) {
       if (command == null)
           throw new NullPointerException();   
       int c = ctl.get();
       // 1.当前线程数小于corePoolSize,则调用addWorker创建核心线程执行任务
       if (workerCountOf(c) < corePoolSize) {
          if (addWorker(command, true))
              return;
          c = ctl.get();
       }
       // 2.如果不小于corePoolSize，则将任务添加到workQueue队列。
       if (isRunning(c) && workQueue.offer(command)) {
           int recheck = ctl.get();
           // 2.1 如果isRunning返回false(状态检查)，则remove这个任务，然后执行拒绝策略。
           if (! isRunning(recheck) && remove(command))
               reject(command);
               // 2.2 线程池处于running状态，但是没有线程，则创建线程
           else if (workerCountOf(recheck) == 0)
               addWorker(null, false);
       }
       // 3.如果放入workQueue失败，则创建非核心线程执行任务，
       // 如果这时创建非核心线程失败(当前线程总数不小于maximumPoolSize时)，就会执行拒绝策略。
       else if (!addWorker(command, false))
            reject(command);
   }
   ```

   1. 线程总数量 < corePoolSize，无论线程是否空闲，都会新建一个核心线程执行任务（让核心线程数量快速达到corePoolSize，在核心线程数量 < corePoolSize时）。**注意，这一步需要获得全局锁。**

   2. 线程总数量 >= corePoolSize时，新来的线程任务会进入任务队列中等待，然后空闲的核心线程会依次去缓存队列中取任务来执行（体现了**线程复用**）。 

   3. 当缓存队列满了，说明这个时候任务已经多到爆棚，需要一些“临时工”来执行这些任务了。于是会创建非核心线程去执行这个任务。**注意，这一步需要获得全局锁。**

   4. 缓存队列满了， 且总线程数达到了maximumPoolSize，则会采取上面提到的拒绝策略进行处理。

4.  ThreadPoolExecutor如何做到线程复用的 

   ThreadPoolExecutor在创建线程时，会将线程封装成**工作线程worker**,并放入**工作线程组**中，然后这个worker反复从阻塞队列中拿任务去执行。 

    execute--> addWorker -->  runWorker -->  getTask 

   ```java
   // Worker.getTask方法源码
   private Runnable getTask() {
       boolean timedOut = false; // Did the last poll() time out?
   
       for (;;) {
           int c = ctl.get();
           int rs = runStateOf(c);
   
           // Check if queue empty only if necessary.
           if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
               decrementWorkerCount();
               return null;
           }
   
           int wc = workerCountOf(c);
   
           // Are workers subject to culling?
           // 1.allowCoreThreadTimeOut变量默认是false,核心线程即使空闲也不会被销毁
           // 如果为true,核心线程在keepAliveTime内仍空闲则会被销毁。 
           boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
           // 2.如果运行线程数超过了最大线程数，但是缓存队列已经空了，这时递减worker数量。 
   　　　　 // 如果有设置允许线程超时或者线程数量超过了核心线程数量，
           // 并且线程在规定时间内均未poll到任务且队列为空则递减worker数量
           if ((wc > maximumPoolSize || (timed && timedOut))
               && (wc > 1 || workQueue.isEmpty())) {
               if (compareAndDecrementWorkerCount(c))
                   return null;
               continue;
           }
   
           try {
               // 3.如果timed为true(想想哪些情况下timed为true),则会调用workQueue的poll方法获取任务.
               // 超时时间是keepAliveTime。如果超过keepAliveTime时长，
               // poll返回了null，上边提到的while循序就会退出，线程也就执行完了。
               // 如果timed为false（allowCoreThreadTimeOut为falsefalse
               // 且wc > corePoolSize为false），则会调用workQueue的take方法阻塞在当前。
               // 队列中有任务加入时，线程被唤醒，take方法返回任务，并执行。
               Runnable r = timed ?
                   workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                   workQueue.take();
               if (r != null)
                   return r;
               timedOut = true;
           } catch (InterruptedException retry) {
               timedOut = false;
           }
       }
   }
   ```

   核心线程的会一直卡在`workQueue.take`方法，被阻塞并挂起，不会占用CPU资源，直到拿到`Runnable` 然后返回（当然如果**allowCoreThreadTimeOut**设置为`true`,那么核心线程就会去调用`poll`方法，因为`poll`可能会返回`null`,所以这时候核心线程满足超时条件也会被销毁）。

   非核心线程会workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) ，如果超时还没有拿到，下一次循环判断**compareAndDecrementWorkerCount**就会返回`null`,Worker对象的`run()`方法循环体的判断为`null`,任务结束，然后线程被系统回收 。

#### 锁接口和类

我们先来看看`synchronized`有什么不足之处。

- 如果临界区是只读操作，其实可以多线程一起执行，但使用synchronized的话，**同一时间只能有一个线程执行**。
- synchronized无法知道线程有没有成功获取到锁
- 使用synchronized，如果临界区因为IO或者sleep方法等原因阻塞了，而当前线程又没有释放锁，就会导致**所有线程等待**。

而这些都是locks包下的锁可以解决的。

##### 锁的分类

1.  可重入锁和非可重入锁

    支持重新进入的锁，也就是说这个锁支持一个**线程对资源重复加锁**。 递归调用。

2.  公平锁与非公平锁

    这里的“公平”，其实通俗意义来说就是“先来后到”，也就是FIFO。如果对一个锁来说，先对锁获取请求的线程一定会先被满足，后对锁获取请求的线程后被满足，那这个锁就是公平的。反之，那就是不公平的。 

    一般情况下，**非公平锁能提升一定的效率。但是非公平锁可能会发生线程饥饿（有一些线程长时间得不到锁）的情况**。所以要根据实际的需求来选择非公平锁和公平锁。

3.  读写锁和排它锁

   我们前面讲到的synchronized用的锁和ReentrantLock，其实都是“排它锁”。也就是说，这些锁在同一时刻只允许一个线程进行访问。

   而读写锁可以再同一时刻允许多个读线程访问。Java提供了ReentrantReadWriteLock类作为读写锁的默认实现，内部维护了两个锁：一个读锁，一个写锁。通过分离读锁和写锁，使得在“读多写少”的环境下，大大地提高了性能。

#####  接口Condition/Lock/ReadWriteLock 

 Lock接口里面有一些获取锁和释放锁的方法声明，而ReadWriteLock里面只有两个方法，分别返回“读锁”和“写锁”： 

```java
public interface ReadWriteLock {
    Lock readLock();
    Lock writeLock();
}
```

 Lock接口中有一个方法是可以获得一个`Condition`: 

```java
Condition newCondition();
```

 之前我们提到了每个对象都可以用继承自`Object`的**wait/notify**方法来实现**等待/通知机制**。而Condition接口也提供了类似Object监视器的方法，通过与**Lock**配合来实现等待/通知模式。  Condition和Object的wait/notify基本相似。其中，Condition的await方法对应的是Object的wait方法，而Condition的**signal/signalAll**方法则对应Object的notify/notifyAll()。但Condition类似于Object的等待/通知机制的加强版。 

 如awaitNanos(long)，当前线程进入等待状态，直到被通知、中断或者超时。如果返回值小于等于0，可以认定就是超时了

#####  ReentrantLock 

ReentrantLock是一个非抽象类，它是Lock接口的JDK默认实现，实现了锁的基本功能。从名字上看，它是一个”可重入“锁，从源码上看，它内部有一个抽象类`Sync`，是继承了AQS，自己实现的一个同步器。同时，ReentrantLock内部有两个非抽象类`NonfairSync`和`FairSync`，它们都继承了Sync。从名字上看得出，分别是”非公平同步器“和”公平同步器“的意思。这意味着ReentrantLock可以支持”公平锁“和”非公平锁“。 

#### 并发集合容器

>  java.util包下提供了一些容器类，而Vector和HashTable是线程安全的容器类，但是这些容器实现同步的方式是通过对方法加锁(sychronized)方式实现的，这样读写均需要锁操作，导致性能低下。 

#####  ConcurrentMap接口 

```java
public interface ConcurrentMap<K, V> extends Map<K, V> {

    //插入元素
    V putIfAbsent(K key, V value);

    //移除元素
    boolean remove(Object key, Object value);

    //替换元素
    boolean replace(K key, V oldValue, V newValue);

    //替换元素
    V replace(K key, V value);
}
```

######  ConcurrentHashMap类

ConcurrentHashMap同HashMap一样也是基于散列表的map，但是它提供了一种与HashTable完全不同的加锁策略提供更高效的并发性和伸缩性。

ConcurrentHashMap提供了一种粒度更细的加锁机制来实现在多线程下更高的性能，这种机制叫分段锁(Lock Striping)。 就是**将数据分段，对每一段数据分配一把锁**。当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问。 有些方法需要跨段，比如size()、isEmpty()、containsValue()，它们可能需要锁定整个表而而不仅仅是某个段，这需要按顺序锁定所有段，操作完毕后，又按顺序释放所有段的锁。 

[详见博客](https://my.oschina.net/hosee/blog/675884)

###### JDK1.7实现

ConcurrentHashMap需要两个哈希操作来定位一个元素。第一个哈希查找Segement，第二个哈希查找元素所在的链表的头部。在这种结构下，哈希进程比普通的HashMap长，但是在那个时候的写操作，只有元素所在的段可以被锁定，并且它不会影响其他的段。理想情况下，ConcurrentHashMap可以同时支持写操作的段数， ConcurrentHashMap's concurrency is improved. 

{% asset_img concurrent-hashmap.png concurrent-hashmap%}

 ![ ](.\Java并发编程-基础-工具\concurrent-hashmap.png) 

* 设计思路

  1.  ConcurrentHashMap中的分段锁称为Segment，它即类似于HashMap的结构，即内部拥有一个Entry数组，数组中的每个元素又是一个链表 

  2.  Segment同时又是一个ReentrantLock（Segment继承了ReentrantLock） 

     ```java
     static final class Segment<K,V> extends ReentrantLock implements Serializable {
         //HashEntry array that actually stores the data
         transient volatile HashEntry<K,V>[] table;
     
         //The number of elements in segemnt
         transient int count;
     
         / / The number of operations that affect the size of the table
             transient int modCount;
     
         / / threshold, the elements in the segment exceed this value will expand
             transient int threshold;
     
         //Load factor
         final float loadFactor;
     }
     ```

  3.  HashEntry中的**value以及next都被volatile修饰**，这样在多线程读写过程中能够保持它们的可见性 

     ```java
     static final class HashEntry<K,V> {
         final int hash;
         final K key;
         volatile V value;
         volatile HashEntry<K,V> next;
     }
     ```

* 初始化

  每次put之前都需要检查key对应的Segment是否为null，如果是则调用ensureSegment()以确保对应的Segment被创建。ensureSegment可能在并发环境下被调用，但与想象中不同，ensureSegment并未使用锁来控制竞争，而是使用了Unsafe对象的getObjectVolatile()提供的原子读语义结合CAS来确保Segment创建的原子性。 

  ```java
  if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                  == null) { // recheck
      Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
      while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
             == null) {
          if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
              break;
      }
  }
  ```

* put/putIfAbsent/putAll

  在获得锁之后，Segment对链表进行遍历，如果某个HashEntry节点具有相同的key，则更新该HashEntry的value值，否则新建一个HashEntry节点，将它设置为链表的新head节点并将原头节点设为新head的下一个节点。新建过程中如果节点总数（含新建的HashEntry）超过threshold，则调用rehash()方法对Segment进行扩容，最后将新建HashEntry写入到数组中。

  put方法中，链接新节点的下一个节点（HashEntry.setNext()）以及将链表写入到数组中（setEntryAt()）都是通过Unsafe的putOrderedObject()方法来实现，这里并未使用具有原子写语义的putObjectVolatile()的原因是：JMM会保证获得锁到释放锁之间所有对象的状态更新都会在锁被释放之后更新到主存，从而保证这些变更对其他线程是可见的。

* get与containsKey

   get与containsKey两个方法几乎完全一致：他们都没有使用锁，而是通过Unsafe对象的getObjectVolatile()方法提供的原子读语义，来获得Segment以及对应的链表，然后对链表遍历判断是否存在key相同的节点以及获得该节点的value。但由于遍历过程中其他线程可能对链表结构做了调整，因此**get和containsKey返回的可能是过时的数据，这一点是ConcurrentHashMap在弱一致性上的体现**。如果要求强一致性，那么必须使用Collections.synchronizedMap()方法。 

* size、containsValue

  首先不加锁循环执行以下操作：循环所有的Segment（通过Unsafe的getObjectVolatile()以保证原子读语义），获得对应的值以及所有Segment的modcount之和。如果连续两次所有Segment的modcount和相等，则过程中没有发生其他线程修改ConcurrentHashMap的情况，返回获得的值。 

  当循环次数超过预定义的值时，这时需要对所有的Segment依次进行加锁，获取返回值后再依次解锁。值得注意的是，加锁过程中要强制创建所有的Segment，否则容易出现其他线程创建Segment并进行put，remove等操作。代码如下： 

  ```java
  for(int j =0; j < segments.length; ++j)
  	ensureSegment(j).lock();// force creation
  ```

 后，与HashMap不同的是，ConcurrentHashMap**并不允许key或者value为null**，按照Doug Lea的说法，这么设计的原因是在ConcurrentHashMap中，一旦value出现null，则代表HashEntry的key/value没有映射完成就被其他线程所见，需要特殊处理。 

###### JDK1.8实现

[ConcurrentHashMap在JDK8中进行了巨大改动，很需要通过源码来再次学习下Doug Lea的实现方法。](https://my.oschina.net/hosee/blog/675884)

它摒弃了Segment（锁段）的概念，而是启用了一种全新的方式实现,利用CAS算法。它沿用了与它同时期的HashMap版本的思想，底层依然由“数组”+链表+红黑树的方式思想([JDK7与JDK8中HashMap的实现](http://my.oschina.net/hosee/blog/618953))，但是为了做到并发，又增加了很多辅助的类，例如TreeBin，Traverser等对象内部类。

##### CopyOnWrite

>  当有多个调用者同时去请求一个资源数据的时候，有一个调用者出于某些原因需要对当前的数据源进行修改，这个时候系统将会复制一个当前数据源的副本给调用者修改。 

CopyOnWrite容器即**写时复制的容器**,当我们往一个容器中添加元素的时候，不直接往容器中添加，而是将当前容器进行copy，复制出来一个新的容器，然后向新容器中添加我们需要的元素，最后将原容器的引用指向新容器。 

**优点**： CopyOnWriteArrayList经常被用于“读多写少”的并发场景，是因为CopyOnWriteArrayList无需任何同步措施，大大增强了读的性能。在Java中遍历线程非安全的List(如：ArrayList和 LinkedList)的时候，若中途有别的线程对List容器进行修改，那么会抛出**ConcurrentModificationException**异常。CopyOnWriteArrayList由于其"读写分离"，遍历和修改操作分别作用在不同的List容器，所以在使用迭代器遍历的时候，则不会抛出异常。 

**缺点：** **内存占用大**，CopyOnWriteArrayList每次执行写操作都会将原容器进行拷贝了一份，数据量大的时候，内存会存在较大的压力，可能会引起频繁Full GC。**时效性** 写和读分别作用在不同新老容器上，在写操作执行过程中，读不会阻塞，但读取到的却是老容器的数据。 

```java
public boolean add(E e) {
    // ReentrantLock加锁，保证线程安全
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        // 拷贝原容器，长度为原容器长度加一
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        // 在新副本上执行添加操作
        newElements[len] = e;
        // 将原容器引用指向新副本
        setArray(newElements);
        return true;
    } finally {
        // 解锁
        lock.unlock();
    }
}
```

**案例：**

```java
import java.util.Collection;
import java.util.Map;
import java.util.Set;
// 我们来具体结合业务场景实现一个CopyOnWriteMap的并发容器并且来使用它。
public class CopyOnWriteMap<K, V> implements Map<K, V>, Cloneable {
    private volatile Map<K, V> internalMap;

    public CopyOnWriteMap() {
        internalMap = new HashMap<K, V>();
    }

    public V put(K key, V value) {
        synchronized (this) {
            Map<K, V> newMap = new HashMap<K, V>(internalMap);
            V val = newMap.put(key, value);
            internalMap = newMap;
            return val;
        }
    }

    public V get(Object key) {
        return internalMap.get(key);
    }

    public void putAll(Map<? extends K, ? extends V> newData) {
        synchronized (this) {
            Map<K, V> newMap = new HashMap<K, V>(internalMap);
            newMap.putAll(newData);
            internalMap = newMap;
        }
    }
}
```

 **场景：**假如我们有一个搜索的网站需要屏蔽一些“关键字”，“黑名单”每晚定时更新，每当用户搜索的时候，“黑名单”中的关键字不会出现在搜索结果当中，并且提示用户敏感字。 

```java
// 黑名单服务
public class BlackListServiceImpl {
    //　减少扩容开销。根据实际需要，初始化CopyOnWriteMap的大小，避免写时CopyOnWriteMap扩容的开销。
    private static CopyOnWriteMap<String, Boolean> blackListMap = 
        new CopyOnWriteMap<String, Boolean>(1000);

    public static boolean isBlackList(String id) {
        return blackListMap.get(id) == null ? false : true;
    }

    public static void addBlackList(String id) {
        blackListMap.put(id, Boolean.TRUE);
    }

    /**
     * 批量添加黑名单
     * (使用批量添加。因为每次添加，容器每次都会进行复制，所以减少添加次数，可以减少容器的复制次数。
     * 如使用上面代码里的addBlackList方法)
     * @param ids
     */
    public static void addBlackList(Map<String,Boolean> ids) {
        blackListMap.putAll(ids);
    }
}
```

##### 线程通信类

1.  CountDownLatch

   CountDownLatch这个类的作用也很贴合这个名字的意义，假设某个线程在执行任务之前，需要等待其它线程完成一些前置任务，必须等所有的前置任务都完成，才能开始执行本线程的任务。 

   ```java
   public class CountDownLatchDemo {
       // 定义前置任务线程
       static class PreTaskThread implements Runnable {
   
           private String task;
           private CountDownLatch countDownLatch;
   
           public PreTaskThread(String task, CountDownLatch countDownLatch) {
               this.task = task;
               this.countDownLatch = countDownLatch;
           }
   
           @Override
           public void run() {
               try {
                   Random random = new Random();
                   Thread.sleep(random.nextInt(1000));
                   System.out.println(task + " - 任务完成");
                   countDownLatch.countDown();
               } catch (InterruptedException e) {
                   e.printStackTrace();
               }
           }
       }
   
       public static void main(String[] args) {
           // 假设有三个模块需要加载
           CountDownLatch countDownLatch = new CountDownLatch(3);
   
           // 主任务
           new Thread(() -> {
               try {
                   System.out.println("等待数据加载...");
                   System.out.println(String.format("还有%d个前置任务", countDownLatch.getCount()));
                   countDownLatch.await();
                   System.out.println("数据加载完成，正式开始游戏！");
               } catch (InterruptedException e) {
                   e.printStackTrace();
               }
           }).start();
   
           // 前置任务
           new Thread(new PreTaskThread("加载地图数据", countDownLatch)).start();
           new Thread(new PreTaskThread("加载人物模型", countDownLatch)).start();
           new Thread(new PreTaskThread("加载背景音乐", countDownLatch)).start();
       }
   }
   ```

   > 等待数据加载... 
   >
   > 还有3个前置任务 
   >
   > 加载人物模型 - 任务完成
   >
   > 加载背景音乐 - 任务完成 
   >
   > 加载地图数据 - 任务完成 
   >
   > 数据加载完成，正式开始游戏！ 

   原理：内部类Sync最简单的实现AQS，初始化后每个线程结束后通过tryReleaseShared将标志位减一，

   ```java
   private static final class Sync extends AbstractQueuedSynchronizer {
       private static final long serialVersionUID = 4982264981922014374L;
   
       Sync(int count) {
           setState(count);
       }
   
       int getCount() {
           return getState();
       }
   
       protected int tryAcquireShared(int acquires) {
           return (getState() == 0) ? 1 : -1;
       }
   
       protected boolean tryReleaseShared(int releases) {
           // Decrement count; signal when transition to zero
           for (;;) {
               int c = getState();
               if (c == 0)
                   return false;
               int nextc = c-1;
               if (compareAndSetState(c, nextc))
                   return nextc == 0;
           }
       }
   }
   ```

   ```java
   // await将等待执行线程，加入到等待队列中。
   public void await() throws InterruptedException {
       sync.acquireSharedInterruptibly(1);
   }
   // 其他前置线程执行完毕后，releaseShared方法最终调用tryReleaseShared
   public void countDown() {
       sync.releaseShared(1);
   }
   ```

#####  Fork/Join

>  Fork/Join框架是一个实现了ExecutorService接口的多线程处理器，它专为那些可以通过递归分解成更细小的任务而设计，最大化的利用多核处理器来提高应用程序的性能。 

 ![img](https://gblobscdn.gitbook.com/assets%2F-L_5HvtIhTFW9TQlOF8e%2F-L_5TIKcBFHWPtY3OwUo%2F-L_5TJw4VyRYbSwAlVTD%2Ffork_join%E6%B5%81%E7%A8%8B%E5%9B%BE.png?alt=media)

> 工作窃取算法指的是在多线程执行不同任务队列的过程中，某个线程执行完自己队列的任务后从其他线程的任务队列里窃取任务来执行。 

值得注意的是，当一个线程窃取另一个线程的时候，为了减少两个任务线程之间的竞争，我们通常使用**双端队列**来存储任务。被窃取的任务线程都从双端队列的**头部**拿任务执行，而窃取其他任务的线程从双端队列的**尾部**执行任务。 

**Fork/Join的具体实现**

通常情况下，在创建任务的时候我们一般不直接继承ForkJoinTask，而是继承它的子类**RecursiveAction**和**RecursiveTask**。两个都是ForkJoinTask的子类，**RecursiveAction可以看做是无返回值的ForkJoinTask，RecursiveTask是有返回值的ForkJoinTask**。

**ForkJoinPool**是用于执行ForkJoinTask任务的执行（线程）池。ForkJoinPool管理着执行池中的线程和任务队列，此外，执行池是否还接受任务，显示线程的运行状态也是在这里处理。

 WorkQueue双端队列，ForkJoinTask存放在这里。当工作线程在处理自己的工作队列时，会从队列尾取任务来执行（LIFO）；如果是窃取其他队列的任务时，窃取的任务位于所属任务队列的队首（FIFO）。ForkJoinPool与传统线程池最显著的区别就是它维护了一个**工作队列数组**（volatile WorkQueue[] workQueues，ForkJoinPool中的**每个工作线程都维护着一个工作队列**）。

使用Fork/Join实现快排

```java
public class QuickSortForkJoin extends RecursiveAction {
    private static final long serialVersionUID = 1L;

    private int[] data;
    private int left;
    private int right;

    public QuickSortForkJoin(int[] data, int left, int right) {
        this.data = data;
        this.left = left;
        this.right = right;
    }

    @Override
    protected void compute() {
        if (left < right) {
            int pivot = partition(data, left, right);
            invokeAll(new QuickSortForkJoin(data, left, pivot - 1),
                    new QuickSortForkJoin(data, pivot + 1, right));
        }
    }

    private int partition(int[] data, int left, int right) {
        int pivotVal = data[left];
        int l = left;
        int r = right;
        while (l < r) {
            while (data[r] >= pivotVal && l < r) {
                r--;
            }
            while (data[l] <= pivotVal && l < r) {
                l++;
            }
            swap(data, l, r);
        }
        swap(data, left, l);
        return l;
    }

    private void swap(int[] data, int a, int b) {
        int temp = data[a];
        data[a] = data[b];
        data[b] = temp;
    }

    public static void main(String[] args) {
        int count = 10000;
        int[] data = new int[count];
        for (int i = count; i > 0; i--) {
            data[i - 1] = (int) (Math.random() * count);
        }
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        Future<Void> result = forkJoinPool.submit(new QuickSortForkJoin(data, 0, count - 1));
        try {
            result.get();
            System.out.println(Arrays.toString(data));
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```

##### Java 8 Stream并行计算

```java
public class StreamParallelDemo {
    public static void main(String[] args) {
        Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9)
                .parallel()
                .reduce((a, b) -> {
                    System.out.println(String.format("%s: %d + %d = %d",
                            Thread.currentThread().getName(), a, b, a + b));
                    return a + b;
                })
                .ifPresent(System.out::println);
    }
}
```

> ForkJoinPool.commonPool-worker-1: 3 + 4 = 7 ForkJoinPool.commonPool-worker-4: 8 + 9 = 17 ForkJoinPool.commonPool-worker-2: 5 + 6 = 11 ForkJoinPool.commonPool-worker-3: 1 + 2 = 3 ForkJoinPool.commonPool-worker-4: 7 + 17 = 24 ForkJoinPool.commonPool-worker-4: 11 + 24 = 35 ForkJoinPool.commonPool-worker-3: 3 + 7 = 10 ForkJoinPool.commonPool-worker-3: 10 + 35 = 45 
>
> 45 