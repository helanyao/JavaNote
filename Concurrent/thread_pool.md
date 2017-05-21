# <center>Thread Pool</center>

<br></br>



&#12288;&#12288;可以把并发执行的任务传递给一个线程池，来替代为每个并发执行的任务都启动新线程。只要池里有空闲的线程，任务就会分配给一个线程执行。在线程池内部，任务被插入阻塞队列（Blocking Queue），线程池里的线程会去取这个队列里的任务。



## 1. 简单的线程池实现
``` java
public class ThreadPool {
  private BlockingQueue taskQueue = null;
  private List<PoolThread> threads = new ArrayList<PoolThread>();
  private boolean isStopped = false;

  public ThreadPool(int noOfThreads, int maxNoOfTasks) {
    taskQueue = new BlockingQueue(maxNoOfTasks);

    for (int i=0; i<noOfThreads; i++) {
      threads.add(new PoolThread(taskQueue));
    }
    for (PoolThread thread : threads) {
      thread.start();
    }
  }

  public void synchronized execute(Runnable task) {
    if(this.isStopped) 
      throw new IllegalStateException("ThreadPool is stopped");

    this.taskQueue.enqueue(task);
  }

  public synchronized boolean stop() {
    this.isStopped = true;
    for (PoolThread thread : threads) {
      thread.stop();
    }
  }
}
```

&#12288;&#12288;线程池实现由两部分组成。类ThreadPool是线程池公开接口，类PoolThread实现执行任务的子线程。

```java
public class PoolThread extends Thread {
  private BlockingQueue<Runnable> taskQueue = null;
  private boolean       isStopped = false;

  public PoolThread(BlockingQueue<Runnable> queue) {
    taskQueue = queue;
  }

  public void run() {
    while (!isStopped()) {
      try {
        Runnable runnable =taskQueue.take();
        runnable.run();
      } catch(Exception e) {
        // 写日志或者报告异常,但保持线程池运行
      }
    }
  }

  public synchronized void toStop() {
    isStopped = true;
    this.interrupt(); // 打断池中线程dequeue()调用.
  }

  public synchronized boolean isStopped() {
    return isStopped;
  }
}
```

&#12288;&#12288;为了执行一个任务，方法`ThreadPool.execute(Runnable r)`用Runnable的实现作为调用参数。在内部，Runnable对象被放入阻塞队列。空闲的PoolThread线程会把Runnable对象从队列中取出并执行。可以在`PoolThread.run()`方法里看到这些代码。执行完后，PoolThread进入循环且尝试从队列中再取出任务，直到线程终止。

&#12288;&#12288;`ThreadPool.stop()`方法可以停止ThreadPool。在内部，调用stop先标记`isStopped`成员变量（为`true`）。然后，线程池的每个子线程都调用`PoolThread.stop()`方法停止运行。如果线程池`execute()`在`stop()`后调用，会抛出IllegalStateException异常。

&#12288;&#12288;子线程会在完成当前任务后停止。`PoolThread.stop()`方法中调用了`this.interrupt()`，确保阻塞在`taskQueue.dequeue()`的`wait()`调用的线程能跳出`wait()`调用，并抛出InterruptedException异常离开`dequeue()`方法。这个异常在`PoolThread.run()`方法中被截获、报告，然后再检查`isStopped`变量。由于`isStopped`值是`true,` 因此`PoolThread.run()`方法退出，子线程终止。

<br></br>



## 2. ForkJoinPool
* JDK1.7引入；
* 大任务分割成小任务；
* 工作窃取(work-stealing)算法。

&#12288;&#12288;为减少窃取任务线程和被窃取任务线程之间的竞争，会使用双端队列，被窃取任务线程永远从双端队列的头部拿任务执行，而窃取任务的线程永远从双端队列的尾部拿任务执行。

&#12288;&#12288;应用场景包括做报表导出时，大量数据的导出处理；做BI时，大量的数据迁移清洗作业等。

<br>


### 2.1 原理
&#12288;&#12288;ForkJoinPool由ForkJoinTask数组和ForkJoinWorkerThread数组组成，ForkJoinTask数组存放提交给ForkJoinPool的任务，而ForkJoinWorkerThread数组负责执行任务。调用ForkJoinTask的`fork()`方法时，会调用ForkJoinWorkerThread的`pushTask()`方法异步执行任务，然后返回结果：

```java
public final ForkJoinTask fork() {         
    ((ForkJoinWorkerThread) Thread.currentThread()).pushTask(this);         
    return this; 
} 
```

&#12288;&#12288;`pushTask()`把当前任务存放在ForkJoinTask数组queue里。然后再调用ForkJoinPool的`signalWork()`方法唤醒或创建一个工作线程来执行任务：

```java
final void pushTask(ForkJoinTask t) {
    ForkJoinTask[] q; int s, m;
    if ((q = queue) != null) {    // ignore if queue removed
        long u = (((s = queueTop) & (m = q.length - 1)) << ASHIFT) + ABASE;
        UNSAFE.putOrderedObject(q, u, t);
        queueTop = s + 1;         // or use putOrderedInt
        if ((s -= queueBase) <= 2)
            pool.signalWork();
        else if (s == m)
            growQueue();
    }
}
```

&#12288;&#12288;`join()`方法的主要作用是阻塞当前线程并等待获取结果：

```java
public final V join() {
    if (doJoin() != NORMAL) 
        return reportResult();
    else
        return getRawResult();
}

private V reportResult() {
    int s;
    Throwable ex;
    if ((s = status) == CANCELLED)
        throw new CancellationException();
    if (s == EXCEPTIONAL && (ex = getThrowableExcetpion()) != null)
        UNSAFE.throwException(ex);

    return getRawResult();
}
```

&#12288;&#12288;首先，调用`doJoin()`得到当前任务的状态来判断返回什么结果，任务状态有四种：已完成（NORMAL），被取消（CANCELLED），信号（SIGNAL）和出现异常（EXCEPTIONAL）。

* 如果任务状态是已完成，则直接返回任务结果。
* 如果任务状态是被取消，则直接抛出CancellationException。
* 如果任务状态是抛出异常，则直接抛出对应的异常。

&#12288;&#12288;`doJoin()`方法的实现代码：

```java
private int doJoin() {
    Thread t;
    ForkJoinThread w;
    int s;
    boolean completed;
    if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) {
        if ((s = status) < 0)
            return s;
        if ((w = (ForkJoinWorkerThread)t).unpushTask(this)) {
            try {
                completed = exec();
            } catch (Throwable rex) {
                return setExceptionalCompletion(rex);
            }
            if (completed)
                return setCompletion(NORMAL);
        }
        return w.joinTask(this);
    } else {
        return externalAwaitDone();
    }
}
```

&#12288;&#12288;在`doJoin()`里，首先通过查看任务的状态，看任务是否已执行完，如果执行完则返回任务状态，如果没有则从任务数组里取出任务并执行。如果任务顺利执行完成了，则设置任务状态为`NORMAL`，如果出现异常，则纪录异常并将任务状态设置为`EXCEPTIONAL`。

<br>


### 2.2 ForkJoinPool VS ExecutorService
Fork-join allows you to easily execute divide and conquer jobs, which have to be implemented manualy if you want to execute it in ExecutorService. In practice ExecutorService is usualy used to process many independent requests (aka transaction) concurrently, and fork-join when you want to accelerate one coherent job.

<br>


### 2.3 RecursiveAction Example（没有返回值）
&#12288;&#12288;任务类`PrintTask`统一用`compute()`方法负责计算子任务或者讲任务分解成子任务；对每次的分解均调用类似`PrintTask.fork()`方法；最后勿忘`ForkJoinPool.shutdown()`.

```java
class PrintTask extends RecursiveAction {
    private static final int MAX = 20; // 每个小任务最多只打印20个数
    private int start, end;

    PrintTask(int s, int e) {
        start = s; end = e;
    }

    @Override
    protected void compute() {
        // 当end - task值小于MAX时，开始打印
        if ((end - start) < MAX) {
            for (int i = start; i < end; i++) {
                System.out.println(Thread.currentThread().getName() + i);
            }
        } else {
            // 将大任务分解成两个小任务
            int middle = (start + end) / 2;
            PrintTask left = new PrintTask(start, middle);
            PrintTask right = new PrintTask(middle, end);
            // 并发执行两个小任务
            left.fork();
            right.fork();
        }
    }
}

public class ForkJoinPoolTest1 {
    public static void main(String[] args) throws Exception {
        // 创建包含Runtime.getRuntime().availableProcessors()返回值作为个数的并行线程池
        ForkJoinPool fjp = new ForkJoinPool();
        // 提交可分解的PrintTask任务
        fjp.submit(new PrintTask(0, 1000));
        fjp.awaitTermination(2, TimeUnit.SECONDS); // 阻塞当前线程直到ForkJoinPool中任务都执行完毕
        fjp.shutdown();
    }
}
```

&#12288;&#12288;计算机共4核，可以看到有4个worker：

![Result](./Images/forkjoinpool_worker.png)

<br>


### 2.4 RecursiveTask Example（有返回值）

```java
class SumTask extends RecursiveTask<Integer> {
    private static final int MAX = 70; // 每个小任务最多只打印70个数
    private int arr[], start, end;

    SumTask(int[] a, int s, int e) {
        arr = a; start = s; end = e;
    }

    @Override
    protected Integer compute() {
        // 当end - task值小于MAX时，开始打印
        if ((end - start) < MAX) {
            for (int i = start; i < end; i++) {
                sum += arr[i];
            }

            return sum;
        } else {
            // 将大任务分解成两个小任务
            int middle = (start + end) / 2;
            SumTask left = new SumTask(arr, start, middle);
            SumTask right = new SumTask(arr, middle, end);
            // 并发执行两个小任务
            left.fork();
            right.fork();

            return left.join() + right.join();
        }
    }
}

public class ForkJoinPoolTest2 {
    public static void main(String[] args) throws Exception {
        int[] arr = new int[1000];
        Random ran = new Random();
        int total = 0;
        for (int i = 0; i < arr.length; i++) {
            int temp = ran.nextInt(100);
            total += (arr[i] = temp); // 计算初始化数据总和
        }
        // 创建包含Runtime.getRuntime().availableProcessors()返回值作为个数的并行线程池
        ForkJoinPool fjp = new ForkJoinPool();
        // 提交可分解的SumTask任务
        Future<Integer> future = fjp.submit(new SumTask(arr, 0, arr.length));
        System.out.println("Result = " + future.get());
        fjp.shutdown();
    }
}
```

<br></br>



## 3. ExecutorService线程池
### 3.1 建立多线程步骤

![Create ExecutorService](./Images/create_executorservice.png)


### 3.2 四种线程池

![Four Kinds of ExecutorService](./Images/kinds_of_executorservice.png)


### 3.3 工厂方法建立线程池
&#12288;&#12288;上面四种线程池都使用Executor的缺省线程工厂建立线程，也可单独定义自己的线程工厂。下面是缺省线程工厂代码:

```java
static class DefaultThreadFactory implements ThreadFactory {
    static final AtomicInteger poolNumber = new AtomicInteger(1);
    final ThreadGroup group;
    final AtomicInteger threadNumber = new AtomicInteger(1);
    final String namePrefix;

    DefaultThreadFactory() {
        SecurityManager s = System.getSecurityManager();
        group = (s != null) ? s.getThreadGroup() : Thread.currentThread().getThreadGroup();
        namePrefix = "pool-" + poolNumber.getAndIncrement() + "-thread- ";
    }

    public Thread newThread(Runnable r) {
        Thread t = new Thread(group, r, namePrefix + threadNumber.getAndIncrement(), 0);
        if (t.isDaemon())
            t.setDaemon(false);
        if (t.getPriority() != Thread.NORM_PRIORITY)
            t.setPriority(Thread.NORM_PRIORITY);

        return t;
    }
}
```

<br>


### 3.4 把Runnable实例加入pool
&#12288;&#12288;Executor的`execute()`将Runnable实例加入pool中,并进行一些pool size计算和优先级处理。`execute()`在Executor接口中定义,有多个实现类都定义了不同的`execute()`方法。如ThreadPoolExecutor类（cache,fiexed,single三种池子都是调用它）的`execute()`方法如下：

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    if (poolSize >= corePoolSize || !addIfUnderCorePoolSize(command)) {
        if (runState == RUNNING && workQueue.offer(command)) {
            if (runStat != RUNNING || poolSize == 0)
                ensureQueuedTaskHandled(command);
        } else if (!addIfUnderMaximumPoolSize(command)) {
            reject(command); // is shutdown or saturated
        }
    }
}
```

&#12288;&#12288;ThreadPoolExecutor有两个最重要的集合属性，分别是存储接收任务的任务队列和用来干活的作业集合：

```java
// task queue
private final BlockingQueue<Runnable> workQueue;
// worker thread set
private final HashSet<Worker> workers = new HashSet<Worker>();
```

&#12288;&#12288;其中阻塞队列workQueue存储待执行任务，在构造线程池时可选择满足BlockingQueue定义的SynchronousQueue、LinkedBlockingQueue或DelayedWorkQueue等不同阻塞队列来实现不同特征的线程池。

<br>


## 3.5 执行任务
&#12288;&#12288;Worker负责执行任务：`private final class Worker implements Runnable`。在`run()`中作业线程调用`getTask()`获取任务，然后调用`runTask(task)`执行任务:

``` java
public void run() {
    try {
        Runnable task = firstTask;
        // 循环从线程池的任务队列获取任务
        while (task != null || (task = getTask()) != null) {
            runTask(task); // 执行任务
            task = null;
        }
    } finally {
        workerDone(this);
    }
}
```

&#12288;&#12288;`getTask()`是ThreadPoolExecutor提供给内部类Worker的方法。作用是从任务队列中取任务:

```java
Runnable getTask() {
    for (;;) {
        r = workQueue.take(); // 从任务队列头部取任务
        return r;
    }
}
```

&#12288;&#12288;`runTask(Runnable task)`是工作线程Worker真正处理每个具体任务:

```java
private void runTask(Runnable task) {
    // 调用任务的run方法，即在Worker线程中执行Task内定义的内容
    task.run();
}
```

&#12288;&#12288;其中的`task.run()`实际是FutureTask的`run()`方法。FutureTask委托内部定义的Sync类，Sync继承自AQS，维护任务状态。

<br>


## 3.6 获取结果
&#12288;&#12288;获取执行结果只需调用FutureTask的`get()`方法即可。该方法是在Future接口中就定义的。

<br>


## 3.7 总结

![Overall of ExecutorService](./Images/executorservice_workflow.png)

1. 首先创建一个任务执行服务ExecutorService，使用工具类Executors工厂方法创建不同的线程池`hreadPoolExecutor；
2. 线程池是负责把任务封装成FutureTask对象，并根据输入定义好要获得结果的类型，就可以`submit()`任务了。
3. 线程池接收输入的task，根据需要创建工作线程，启动工作线程来执行task。
4. 工作线程在其`run()`方法中一直循环，从线程池领取可以执行的task，调用task的`run()`方法执行task的任务。
5. FutureTask的`run()`方法中调用内部类Sync的`innerRun()`方法执行具体任务，并把任务的执行结果返回给FutureTask的`result`变量。
6. 当提及任务的角色调用FutureTask的`get()`方法获取执行结果时，Sync的`innerGet()`方法被调用。根据任务的执行状态判断，任务执行完毕则返回执行结果；未执行完毕则等待。
















































