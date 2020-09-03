# Java线程池

## 1. 线程池介绍

线程池是面试中我们常见的问题，也是实际开发中会经常用到的工具，所以我们要更好的了解线程池相关的内容，从而更好的进行开发，也对大家面试有一定的帮助。

### 线程池常见问题

1. 线程池的各个参数有什么作用？
2. 按照线程池的内部机制，提交新任务时，有哪些异常需要考虑？
3. 线程池的工作队列有哪几种？
4. 无界队列是否会导致内存飙升？
5. 常见线程池以及使用场景

### 什么是线程池

**线程池**（英语：thread pool）：一种线程使用模式。线程过多会带来调度开销，进而影响缓存局部性和整体性能。而线程池维护着多个线程，等待着监督管理者分配可并发执行的任务。这避免了在处理短时间任务时创建与销毁线程的代价。线程池不仅能够保证内核的充分利用，还能防止过分调度。可用线程数量应该取决于可用的并发处理器、处理器内核、内存、网络sockets等的数量。 例如，线程数一般取cpu数量+2比较合适，线程数过多会导致额外的线程切换开销。

> 资料源自百度百科

### 线程池的优势

1. **降低资源消耗**：重复利用已创建的线程降低线程创建和销毁造成的消耗
2. **提高响应速度**：任务到达时，任务可以不需要等到线程被创建就能立即执行
3. **提高线程的可管理性**：线程池可以对线程进行统一的分配，调优和监控

## 2. 线程池的使用

### 线程池的构造方法

线程池的真正实现类时**ThreadPoolExecutor**，他有四种构造方法（大致看下源码即可，后面有详细的参数分析）

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         Executors.defaultThreadFactory(), defaultHandler);
}
 
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         threadFactory, defaultHandler);
}
 
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          RejectedExecutionHandler handler) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         Executors.defaultThreadFactory(), handler);
}
 
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

可以看到，其需要如下几个参数：

- **corePoolSize**（必需）：核心线程数。默认情况下，核心线程会一直存活，但是当将**allowCoreThreadTimeout**设置为true时，核心线程也会超时回收。
- **maximumPoolSize**（必需）：线程池所能容纳的最大线程数。当活跃线程数达到该数值后，后续的新任务将会阻塞。
- **keepAliveTime**（必需）：线程闲置超时时长。如果超过该时长，非核心线程就会被回收。如果将**allowCoreThreadTimeout**设置为true时，核心线程也会超时回收。
- **unit**（必需）：指定keepAliveTime参数的时间单位。常用的有：**TimeUnit.MILLISECONDS**（毫秒）、**TimeUnit.SECONDS**（秒）、**TimeUnit.MINUTES**（分）。
- **workQueue**（必需）：任务队列。通过线程池的execute()方法提交的Runnable对象将存储在该参数中。其采用阻塞队列实现。
- **threadFactory**（可选）：线程工厂。用于指定为线程池创建新线程的方式。
- **handler**（可选）：拒绝策略。当达到最大线程数时需要执行的饱和策略。

在这里我们提到了“核心线程数”和“最大线程数”，这两个的区别是什么呢？什么是核心线程？除了核心线程，普通线程是做什么的呢？后面我们将会讲到。

### 线程池的使用流程

```java
// 创建线程池
ThreadPoolExecutor threadPool = new ThreadPoolExecutor(CORE_POOL_SIZE,
                                             MAXIMUM_POOL_SIZE,
                                             KEEP_ALIVE,
                                             TimeUnit.SECONDS,
                                             sPoolWorkQueue,
                                             sThreadFactory);
// 向线程池提交任务
threadPool.execute(new Runnable() {
    @Override
    public void run() {
        ... // 线程执行的任务
    }
});
// 关闭线程池
threadPool.shutdown(); // 设置线程池的状态为SHUTDOWN，然后中断所有没有正在执行任务的线程
threadPool.shutdownNow(); // 设置线程池的状态为 STOP，然后尝试停止所有的正在执行或暂停任务的线程，并返回等待执行任务的列表
```

## 3. 线程池的工作原理

上面我们提到了核心线程和非核心线程，其中核心线程就是用来处理我们任务的，但是核心线程用光了之后，我们只能把任务放到消息队列里，但是万一消息队列也用光了呢？

这个时候就需要创建非核心线程。但如果万一非核心线程+核心线程的数量大于了最大线程数**maximumPoolSize**，那么只能通过handler来处理拒绝策略了

下面我们来看一张流程图

![](https://img-blog.csdnimg.cn/20190809200646357.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9qaW1teXN1bi5ibG9nLmNzZG4ubmV0,size_16,color_FFFFFF,t_70)

## 4. 线程池的任务队列(workQueue)

任务队列是基于阻塞队列实现的，即采用生产者消费者模式。在Java中我们需要实现BlockingQueue接口。但是Java已经为我们提供了七种阻塞队列的实现：

1. **ArrayBlockingQueue**：一个由数组结构组成的有界阻塞队列（数组结构可配合指针实现一个环形队列）。
2. **LinkedBlockingQueue**： 一个由链表结构组成的有界阻塞队列，在未指明容量时，容量默认为**Integer.MAX_VALUE**。
3. **PriorityBlockingQueue**： 一个支持优先级排序的无界阻塞队列，对元素没有要求，可以实现**Comparable**接口也可以提供Comparator来对队列中的元素进行比较。跟时间没有任何关系，仅仅是按照优先级取任务。
4. **DelayQueue**：类似于**PriorityBlockingQueue**，是二叉堆实现的无界优先级阻塞队列。要求元素都实现**Delayed**接口，通过执行时延从队列中提取任务，时间没到任务取不出来。
5. **SynchronousQueue**： 一个不存储元素的阻塞队列，消费者线程调用take()方法的时候就会发生阻塞，直到有一个生产者线程生产了一个元素，消费者线程就可以拿到这个元素并返回；生产者线程调用put()方法的时候也会发生阻塞，直到有一个消费者线程消费了一个元素，生产者才会返回。
6. **LinkedBlockingDeque**： 使用双向队列实现的有界双端阻塞队列。双端意味着可以像普通队列一样FIFO（先进先出），也可以像栈一样FILO（先进后出）。
7. **LinkedTransferQueue**： 它是**ConcurrentLinkedQueue**、**LinkedBlockingQueue**和**SynchronousQueue**的结合体，但是把它用在ThreadPoolExecutor中，和**LinkedBlockingQueue**行为一致，但是是无界的阻塞队列。

## 5. 线程工厂(threadFactor)

在上面我们提到过线程池的参数中有一个为**threadFactory**（可选），他指定为线程池创建新线程的方式。

线程工厂指定创建线程的方式，需要实现**Threadfactory**接口，并实现**newThread(Runnable r)**方法。该参数可以不用指定，Executors框架已经为我们实现了一个默认的线程工厂：

```java

/**
 * The default thread factory.
 */
private static class DefaultThreadFactory implements ThreadFactory {
    private static final AtomicInteger poolNumber = new AtomicInteger(1);
    private final ThreadGroup group;
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;
 
    DefaultThreadFactory() {
        SecurityManager s = System.getSecurityManager();
        group = (s != null) ? s.getThreadGroup() :
                              Thread.currentThread().getThreadGroup();
        namePrefix = "pool-" +
                      poolNumber.getAndIncrement() +
                     "-thread-";
    }
 
    public Thread newThread(Runnable r) {
        Thread t = new Thread(group, r,
                              namePrefix + threadNumber.getAndIncrement(),
                              0);
        if (t.isDaemon())
            t.setDaemon(false);
        if (t.getPriority() != Thread.NORM_PRIORITY)
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}
```

## 6. 拒绝策略(handler)

在上面的流程图中，当我们的线程创建数达到最大时，如果还是不够用，那线程池就没办法了，他只能执行拒绝策略。

拒绝策略需要实现**RejectedExecutionHandler**接口，并实现**rejectedExecution(Runnable r, ThreadPoolExecutor executor)**方法

不过和线程工厂一样，Executors框架已经为我们实现了4种拒绝策略：

- **AbortPolicy**：丢弃任务并抛出**RejectedExecutionException**异常
- **DiscardPolicy**：丢弃任务，但是不抛出异常。可以配合这种模式进行自定义的处理方式。
- **DiscardOldestPolicy**：丢弃队列里最老的任务，将当前这个任务继续提交给线程池
- **CallerRunsPolicy**：交给线程池调用所在的线程进行处理

## 7. 四大功能线程池

就像前面的threadFactor和handler一样，Executors已经为我们封装好了4种常见的功能线程池，如下：

- 定长线程池（FixedThreadPool）
- 定时线程池（ScheduledThreadPool ）
- 可缓存线程池（CachedThreadPool）
- 单线程化线程池（SingleThreadExecutor）

### 定长线程池（FixedThreadPool）

创建方法的源码：

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>(),
                                  threadFactory);
}
```

从他的两个构造方法我们可以看到，nThreads是必要的参数，也就意味着我们一定要指定线程池中线程数的大小。

而且我们还能看到，在ThreadPoolExecutor的构造函数中，核心线程和总线程数都被指定为了nThreads，这也就意味着对于定长线程池FixedThreadPool而言，他只有核心线程

- **特点**：只有核心线程，线程数量固定，执行完立即回收，任务队列为链表结构的有界队列（**LinkedBlockingQueue**）。
- **应用场景**：控制线程最大并发数。

使用示例：

```java

// 1. 创建定长线程池对象 & 设置线程池线程数量固定为3
ExecutorService fixedThreadPool = Executors.newFixedThreadPool(3);
// 2. 创建好Runnable类线程对象 & 需执行的任务
Runnable task =new Runnable(){
  public void run() {
     System.out.println("执行任务啦");
  }
};
// 3. 向线程池提交任务
fixedThreadPool.execute(task);
```

### 定时线程池（ScheduledThreadPool ）

创建方法源码：

```java
private static final long DEFAULT_KEEPALIVE_MILLIS = 10L;
 
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE,
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
          new DelayedWorkQueue());
}
 
public static ScheduledExecutorService newScheduledThreadPool(
        int corePoolSize, ThreadFactory threadFactory) {
    return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
}
public ScheduledThreadPoolExecutor(int corePoolSize,
                                   ThreadFactory threadFactory) {
    super(corePoolSize, Integer.MAX_VALUE,
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
          new DelayedWorkQueue(), threadFactory);
}
```

从构造方法我们还是可以看到，我们一定要指定线程池的线程数，线程工厂可选。

这也就意味着其他的参数都是被设置好的，那我们就依次来看看

我们会发现 ：

- 最大线程数为**Integer.MAX_VALUE**，这意味着非核心线程数量是无上限的
- **keepAliveTime**被设置为了**DEFAULT_KEEPALIVE_MILLIS**，这是个常量，大小为10ms，也就意味着线程闲置10ms后回收，这么设计是为了防止由于非核心线程无上限，不及时回收会OOM，爆内存
- 任务队列被设置为**DelayedWorkQueue**，为延时阻塞队列，这是为了实现延时执行任务

**应用场景**：执行定时或周期性的任务。

使用示例：

```java
// 1. 创建 定时线程池对象 & 设置线程池线程数量固定为5
ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(5);
// 2. 创建好Runnable类线程对象 & 需执行的任务
Runnable task =new Runnable(){
  public void run() {
     System.out.println("执行任务啦");
  }
};
// 3. 向线程池提交任务
scheduledThreadPool.schedule(task, 1, TimeUnit.SECONDS); // 延迟1s后执行任务
scheduledThreadPool.scheduleAtFixedRate(task,10,1000,TimeUnit.MILLISECONDS);// 延迟10ms后、每隔1000ms执行任务
scheduledThreadPool.scheduleWithFixedDelay(task,10,1000,TimeUnit.MILLISECONDS);// 延迟10ms后、每隔1000ms执行任务
```

**注意**：**scheduleAtFixedRate**和**scheduleWithFixedDelay**的区别：

​					前者是上一个任务的**开始**时间 + 延迟时间 = 下一个任务的开始时间

​					后者是上一次任务的**结束**时间 + 延迟时间 = 下一次任务的开始时间

### 可缓存线程池（CachedThreadPool）

创建方法源码：

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>(),
                                  threadFactory);
}
```

从构造方法可以看出以下几点：

- 0核心线程
- 非核心线程无上限
- 执行完闲置60s回收
- 任务队列为不储存元素元素的阻塞队列SynchronousQueue

**应用场景**：执行大量、耗时少的任务。

这里有个问题，为什么采用不储存元素的阻塞队列呢？其实跟他的运行流程有关，来看个图

![](https://pic4.zhimg.com/v2-ebd1b37b1baaf31854af0de28340238f_b.jpg)

可以看到，因为没有核心线程，所以任务直接加到SynchronousQueue队列。而且由于非核心线程大小无上限，所以消息一定会被新创建/已有的线程拿走，所以不需要储存元素。

使用示例：

```java
// 1. 创建可缓存线程池对象
ExecutorService cachedThreadPool = Executors.newCachedThreadPool();
// 2. 创建好Runnable类线程对象 & 需执行的任务
Runnable task =new Runnable(){
  public void run() {
     System.out.println("执行任务啦");
  }
};
// 3. 向线程池提交任务
cachedThreadPool.execute(task);
```

### 单线程化线程池（SingleThreadExecutor）

创建方法源码：

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>(),
                                threadFactory));
}
```

从构造函数可以看出：

- 只有一个核心线程
- 无非核心线程
- 执行完立即回收
- 任务队列为链表结构的有界队列

**应用场景**：不适合并发但可能引起IO阻塞性及影响UI线程响应的操作，如数据库操作、文件操作等。（因为能保持串行化工作，所以数据库能保证ACID四大原则）

使用实例：

```java
// 1. 创建单线程化线程池
ExecutorService singleThreadExecutor = Executors.newSingleThreadExecutor();
// 2. 创建好Runnable类线程对象 & 需执行的任务
Runnable task =new Runnable(){
  public void run() {
     System.out.println("执行任务啦");
  }
};
// 3. 向线程池提交任务
singleThreadExecutor.execute(task);
```

### 对比

![](https://img-blog.csdnimg.cn/20190721095954211.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTM1NDExNDA=,size_16,color_FFFFFF,t_70)

## 8. 线程池可能存在的问题

Executors的4个功能线程池虽然方便，但现在已经不建议使用了，而是建议直接通过使用ThreadPoolExecutor的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。

其实Executors的4个功能线程有如下弊端：

- **FixedThreadPool**和**SingleThreadExecutor**：主要问题是堆积的请求处理队列均采用**LinkedBlockingQueue**，可能会耗费非常大的内存，甚至OOM。
- **CachedThreadPool**和**ScheduledThreadPool**：主要问题是线程数最大数是**Integer.MAX_VALUE**，可能会创建数量非常多的线程，甚至OOM。

 