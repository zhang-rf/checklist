OpenJDK版本：jdk11-1ddf9a99e4ad

# 核心字段

## ctl
高3位存储线程池状态，低29位存储工作线程数（workerCount）
```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```

## 线程池状态
RUNNING：接受新任务并处理队列中的任务

SHUTDOWN：不接受新任务，但处理队列中的任务

STOP：不接受新任务，放弃队列中的任务，并且中断工作线程

TIDYING：所有任务已终止，workerCount为零，但还未执行`terminated()`钩子

TERMINATED：`terminated()`钩子已执行
```java
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
```

## workQueue
工作队列
```java
private final BlockingQueue<Runnable> workQueue;
```

## mainLock
线程池中各种操作时的互斥锁
```java
private final ReentrantLock mainLock = new ReentrantLock();
```

## workers
工作线程集
```java
private final HashSet<Worker> workers = new HashSet<>();
```

## threadFactory
线程工厂，负责新建线程
```java
private volatile ThreadFactory threadFactory;
```

## handler
线程数达到上限且工作队列已满时的处理策略
```java
private volatile RejectedExecutionHandler handler;
```

## keepAliveTime
线程空闲时的KeepAlive时间
```java
private volatile long keepAliveTime;
```

## corePoolSize
核心线程数
```java
private volatile int corePoolSize;
```

## maximumPoolSize
最大线程数
```java
private volatile int maximumPoolSize;
```

# 核心方法

## execute(Runnable)
```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
    int c = ctl.get();
    // workerCount小于核心线程数，新建线程执行任务
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 尝试任务入队列
    if (isRunning(c) && workQueue.offer(command)) {
        // 成功后重新检查状态
        int recheck = ctl.get();
        // 线程池关闭，拒绝执行
        if (! isRunning(recheck) && remove(command))
            reject(command);
        // 没有核心线程，新建线程去执行队列中的任务
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 尝试新建非核心线程执行任务
    else if (!addWorker(command, false))
        reject(command);
}
```

## addWorker(Runnable, boolean)
```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (int c = ctl.get();;) {
        // Check if queue empty only if necessary.
        // 线程池关闭
        if (runStateAtLeast(c, SHUTDOWN)
            && (runStateAtLeast(c, STOP)
                || firstTask != null
                || workQueue.isEmpty()))
            return false;
        for (;;) {
            // 线程数达到上限
            if (workerCountOf(c)
                >= ((core ? corePoolSize : maximumPoolSize) & COUNT_MASK))
                return false;
            // CAS操作设置ctl
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateAtLeast(c, SHUTDOWN))
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        // 新建线程
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int c = ctl.get();
                // 加锁后二次检查状态
                if (isRunning(c) ||
                    (runStateLessThan(c, STOP) && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    // 历史最大线程数
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        // 新建线程失败，资源回收，并尝试关闭线程池
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

## runWorker(Worker)
```java
final void runWorker(Worker w) {
    // 工作线程
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    // 异常退出
    boolean completedAbruptly = true;
    try {
        // 存在待执行任务
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            // 线程池关闭
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                // 让任务感知到线程池关闭
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                try {
                    // 执行任务
                    task.run();
                    afterExecute(task, null);
                } catch (Throwable ex) {
                    afterExecute(task, ex);
                    throw ex;
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        // 处理工作线程退出
        processWorkerExit(w, completedAbruptly);
    }
}
```

## getTask()
```java
private Runnable getTask() {
    // 获取任务超时
    boolean timedOut = false; // Did the last poll() time out?
    for (;;) {
        int c = ctl.get();
        // Check if queue empty only if necessary.
        // 线程池关闭
        if (runStateAtLeast(c, SHUTDOWN)
            && (runStateAtLeast(c, STOP) || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }
        int wc = workerCountOf(c);
        // Are workers subject to culling?
        // 线程空闲时是否退出
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        // 线程空余
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }
        try {
            // 阻塞获取任务
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

## processWorkerExit()
```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    // 异常退出
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    // 计算完成任务数
    try {
        completedTaskCount += w.completedTasks;
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }
    // 尝试关闭线程池
    tryTerminate();
    int c = ctl.get();
    // 如果状态不是STOP
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        // 异常退出时重建线程
        addWorker(null, false);
    }
}
```

## shutdown()
```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        // 设置线程池状态
        advanceRunState(SHUTDOWN);
        // 中断空闲线程
        interruptIdleWorkers();
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    // 尝试关闭线程池
    tryTerminate();
}
```

## tryTerminate()
```java
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        // 已经TIDYING或还有正在执行的任务
        if (isRunning(c) ||
            runStateAtLeast(c, TIDYING) ||
            (runStateLessThan(c, STOP) && ! workQueue.isEmpty()))
            return;
        if (workerCountOf(c) != 0) { // Eligible to terminate
            interruptIdleWorkers(ONLY_ONE);
            return;
        }
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 设置线程池状态为TIDYING
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                    // 执行terminated()钩子
                    terminated();
                } finally {
                    // 设置线程池状态为TERMINATED
                    ctl.set(ctlOf(TERMINATED, 0));
                    // 唤醒正在执行awaitTermination()的线程
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
        // else retry on failed CAS
    }
}
```

## shutdownNow()
```java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        // 设置线程池状态为STOP
        advanceRunState(STOP);
        // 中断所有线程
        interruptWorkers();
        // 清空工作队列
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    // 尝试关闭线程池
    tryTerminate();
    // 返回工作队列中的任务
    return tasks;
}
```

# 线程池参数组合

## newFixedThreadPool
1. corePoolSize == maximumPoolSize
2. 无界队列

线程池的工作线程数小于corePoolSize时，新建工作线程
线程池的工作线程数等于corePoolSize时，将任务放入工作队列

## newSingleThreadExecutor
1. corePoolSize = maximumPoolSize = 1
2. 无界队列

同newFixedThreadPool

## newCachedThreadPool
1. corePoolSize = 0, maximumPoolSize = Integer.MAX_VALUE
3. keepAliveTime = 60s
2. SynchronousQueue

使用SynchronousQueue的零容量特性，没有空闲线程时新建工作线程

# 拒绝策略

## CallerRunsPolicy
调用线程执行被拒绝的任务

## AbortPolicy
抛出异常

## DiscardPolicy
丢弃提交的任务

## DiscardOldestPolicy
丢弃工作队列中的第一个任务，然后尝试再次提交任务

# ScheduledThreadPoolExecutor
由延迟队列实现的定时线程池，继承自ThreadPoolExecutor，本文不做详细介绍