OpenJDK版本：jdk11-1ddf9a99e4ad

# 原理
通过链表实现的队列存储待获取资源的线程，外加一个`int`字段存储资源的数量，是各类锁及同步类的基类。

# 核心字段

## exclusiveOwnerThread
独占模式下，锁的持有者的线程
```java
private transient Thread exclusiveOwnerThread;
```

## state
同步状态（子类实现），一般用来存储资源数量或资源的占用次数
```java
private volatile int state;
```

## head/tail
等待队列的头节点、尾节点
```java
private transient volatile Node head;
private transient volatile Node tail;
```

# 数据结构
```java
static final class Node {
    /** Marker to indicate a node is waiting in shared mode */
    // 共享模式
    static final Node SHARED = new Node();
    /** Marker to indicate a node is waiting in exclusive mode */
    // 独占模式
    static final Node EXCLUSIVE = null;

    /** waitStatus value to indicate thread has cancelled. */
    // 等待队列中的线程因为一些原因如中断等，已经取消排队
    static final int CANCELLED =  1;
    /** waitStatus value to indicate successor's thread needs unparking. */
    // 释放当前节点时需要唤醒后继节点
    static final int SIGNAL    = -1;
    /** waitStatus value to indicate thread is waiting on condition. */
    // 当前节点在等待ConditionObject
    static final int CONDITION = -2;
    /**
     * waitStatus value to indicate the next acquireShared should
     * unconditionally propagate.
     */
    // 共享模式时使用
    static final int PROPAGATE = -3;

    /**
     * Status field, taking on only the values:
     *   SIGNAL:     The successor of this node is (or will soon be)
     *               blocked (via park), so the current node must
     *               unpark its successor when it releases or
     *               cancels. To avoid races, acquire methods must
     *               first indicate they need a signal,
     *               then retry the atomic acquire, and then,
     *               on failure, block.
     *   CANCELLED:  This node is cancelled due to timeout or interrupt.
     *               Nodes never leave this state. In particular,
     *               a thread with cancelled node never again blocks.
     *   CONDITION:  This node is currently on a condition queue.
     *               It will not be used as a sync queue node
     *               until transferred, at which time the status
     *               will be set to 0. (Use of this value here has
     *               nothing to do with the other uses of the
     *               field, but simplifies mechanics.)
     *   PROPAGATE:  A releaseShared should be propagated to other
     *               nodes. This is set (for head node only) in
     *               doReleaseShared to ensure propagation
     *               continues, even if other operations have
     *               since intervened.
     *   0:          None of the above
     *
     * The values are arranged numerically to simplify use.
     * Non-negative values mean that a node doesn't need to
     * signal. So, most code doesn't need to check for particular
     * values, just for sign.
     *
     * The field is initialized to 0 for normal sync nodes, and
     * CONDITION for condition nodes.  It is modified using CAS
     * (or when possible, unconditional volatile writes).
     */
    volatile int waitStatus;

    /**
     * Link to predecessor node that current node/thread relies on
     * for checking waitStatus. Assigned during enqueuing, and nulled
     * out (for sake of GC) only upon dequeuing.  Also, upon
     * cancellation of a predecessor, we short-circuit while
     * finding a non-cancelled one, which will always exist
     * because the head node is never cancelled: A node becomes
     * head only as a result of successful acquire. A
     * cancelled thread never succeeds in acquiring, and a thread only
     * cancels itself, not any other node.
     */
    volatile Node prev;

    /**
     * Link to the successor node that the current node/thread
     * unparks upon release. Assigned during enqueuing, adjusted
     * when bypassing cancelled predecessors, and nulled out (for
     * sake of GC) when dequeued.  The enq operation does not
     * assign next field of a predecessor until after attachment,
     * so seeing a null next field does not necessarily mean that
     * node is at end of queue. However, if a next field appears
     * to be null, we can scan prev's from the tail to
     * double-check.  The next field of cancelled nodes is set to
     * point to the node itself instead of null, to make life
     * easier for isOnSyncQueue.
     */
    volatile Node next;

    /**
     * The thread that enqueued this node.  Initialized on
     * construction and nulled out after use.
     */
    volatile Thread thread;

    /**
     * Link to next node waiting on condition, or the special
     * value SHARED.  Because condition queues are accessed only
     * when holding in exclusive mode, we just need a simple
     * linked queue to hold nodes while they are waiting on
     * conditions. They are then transferred to the queue to
     * re-acquire. And because conditions can only be exclusive,
     * we save a field by using special value to indicate shared
     * mode.
     */
    // Condition队列指针
    Node nextWaiter;
}
```

# 核心方法

## acquire(int)
```java
public final void acquire(int arg) {
    // 尝试获取资源（独占模式，子类实现）
    if (!tryAcquire(arg) &&
        // 获取资源失败入队列，等待资源分配
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        // 还原Thread.interrupted
        selfInterrupt();
}
```

## addWaiter(Node)
```java
private Node addWaiter(Node mode) {
    Node node = new Node(mode);
    for (;;) {
        Node oldTail = tail;
        // 队列已经初始化
        if (oldTail != null) {
            node.setPrevRelaxed(oldTail);
            // CAS操作更新尾节点
            if (compareAndSetTail(oldTail, node)) {
                oldTail.next = node;
                return node;
            }
        } else {
            // 初始化头节点
            initializeSyncQueue();
        }
    }
}
```

## acquireQueued(Node, int)
```java
final boolean acquireQueued(final Node node, int arg) {
    boolean interrupted = false;
    try {
        for (;;) {
            // 前驱
            final Node p = node.predecessor();
            // 前驱是head，尝试获取资源
            if (p == head && tryAcquire(arg)) {
                // 获取到资源，当前节点变为头节点
                setHead(node);
                p.next = null; // help GC
                return interrupted;
            }
            // 检查是否需要阻塞等待
            if (shouldParkAfterFailedAcquire(p, node))
                // 调用LockSupport.park()
                interrupted |= parkAndCheckInterrupt();
        }
    } catch (Throwable t) {
        // tryAcquire()异常情况下，取消当前节点
        cancelAcquire(node);
        if (interrupted)
            // 还原Thread.interrupted
            selfInterrupt();
        throw t;
    }
}
```

## shouldParkAfterFailedAcquire(Node, Node)
```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    // 前驱节点在释放资源时会唤醒下一节点
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    // 跳过CANCELLED前驱节点
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        // 确保前驱节点在释放资源时会唤醒下一节点
        pred.compareAndSetWaitStatus(ws, Node.SIGNAL);
    }
    return false;
}
```

## doAcquireNanos(int, long)
```java
private boolean doAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    // 超时时间点
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.EXCLUSIVE);
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                return true;
            }
            // 减去循环耗时
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L) {
                // 获取超时，取消当前节点
                cancelAcquire(node);
                return false;
            }
            // 超时时间大于SPIN_FOR_TIMEOUT_THRESHOLD时使用LockSupport.parkNanos()，否则自旋
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > SPIN_FOR_TIMEOUT_THRESHOLD)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        throw t;
    }
}
```

## cancelAcquire(Node)
```java
private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    if (node == null)
        return;
    node.thread = null;
    // Skip cancelled predecessors
    // 跳过CANCELLED前驱
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;
    // predNext is the apparent node to unsplice. CASes below will
    // fail if not, in which case, we lost race vs another cancel
    // or signal, so no further action is necessary, although with
    // a possibility that a cancelled node may transiently remain
    // reachable.
    Node predNext = pred.next;
    // Can use unconditional write instead of CAS here.
    // After this atomic step, other Nodes can skip past us.
    // Before, we are free of interference from other threads.
    node.waitStatus = Node.CANCELLED;
    // If we are the tail, remove ourselves.
    // 如果取消的节点是尾节点，直接将尾节点设置为未取消的前驱
    if (node == tail && compareAndSetTail(node, pred)) {
        pred.compareAndSetNext(predNext, null);
    } else {
        // If successor needs signal, try to set pred's next-link
        // so it will get one. Otherwise wake it up to propagate.
        int ws;
        // 前驱不是头节点，并且前驱状态设为SIGNAL
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && pred.compareAndSetWaitStatus(ws, Node.SIGNAL))) &&
            pred.thread != null) {
            // 将中间CANCELLED的节点移除
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                pred.compareAndSetNext(predNext, next);
        } else {
            // 前驱是头节点，唤醒当前节点线程
            unparkSuccessor(node);
        }
        node.next = node; // help GC
    }
}
```

## unparkSuccessor(Node)
```java
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            // 清除节点状态
            node.compareAndSetWaitStatus(ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        // 跳过CANCELLED节点
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node p = tail; p != node && p != null; p = p.prev)
                if (p.waitStatus <= 0)
                    s = p;
        }
        if (s != null)
            // 唤醒线程
            LockSupport.unpark(s.thread);
    }
```

## release(int)
```java
public final boolean release(int arg) {
    // 尝试释放资源（独占模式，子类实现）
    if (tryRelease(arg)) {
        Node h = head;
        // 释放后，如果状态为SIGNAL，调用LockSupport.unpark()唤醒下一线程
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

## doAcquireShared(int)
```java
private void doAcquireShared(int arg) {
    // 共享模式
    final Node node = addWaiter(Node.SHARED);
    boolean interrupted = false;
    try {
        for (;;) {
            final Node p = node.predecessor();
            // 前驱是头节点
            if (p == head) {
                // 尝试获取资源
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    // 成功后当前节点变更为头节点，并检查是否需要唤醒下一节点
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node))
                interrupted |= parkAndCheckInterrupt();
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        throw t;
    } finally {
        if (interrupted)
            selfInterrupt();
    }
}
```

## releaseShared(int)
```java
public final boolean releaseShared(int arg) {
    // 尝试释放资源（共享模式，子类实现）
    if (tryReleaseShared(arg)) {
        // 成功后唤醒后继节点
        doReleaseShared();
        return true;
    }
    return false;
}
```

## doReleaseShared()
```java
private void doReleaseShared() {
    /*
     * Ensure that a release propagates, even if there are other
     * in-progress acquires/releases.  This proceeds in the usual
     * way of trying to unparkSuccessor of head if it needs
     * signal. But if it does not, status is set to PROPAGATE to
     * ensure that upon release, propagation continues.
     * Additionally, we must loop in case a new node is added
     * while we are doing this. Also, unlike other uses of
     * unparkSuccessor, we need to know if CAS to reset status
     * fails, if so rechecking.
     */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!h.compareAndSetWaitStatus(Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                // 头节点SIGNAL状态，唤醒后继节点
                unparkSuccessor(h);
            }
            // 头节点状态设为PROPAGATE
            else if (ws == 0 &&
                     !h.compareAndSetWaitStatus(0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

## ConditionObject.await(int)
```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 等待节点入队
    Node node = addConditionWaiter();
    // 释放资源
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        // 阻塞线程
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // signal后恢复资源占用
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    // 恢复interrupt标志或异常
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

## isOnSyncQueue(Node)
```java
final boolean isOnSyncQueue(Node node) {
    // 节点状态为CONDITION
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    if (node.next != null) // If has successor, it must be on queue
        return true;
    /*
     * node.prev can be non-null, but not yet on queue because
     * the CAS to place it on queue can fail. So we have to
     * traverse from tail to make sure it actually made it.  It
     * will always be near the tail in calls to this method, and
     * unless the CAS failed (which is unlikely), it will be
     * there, so we hardly ever traverse much.
     */
    return findNodeFromTail(node);
}
```

## transferForSignal(Node)
```java
final boolean transferForSignal(Node node) {
    /*
     * If cannot change waitStatus, the node has been cancelled.
     */
    // 清除CONDITION状态
    if (!node.compareAndSetWaitStatus(Node.CONDITION, 0))
        return false;
    /*
     * Splice onto queue and try to set waitStatus of predecessor to
     * indicate that thread is (probably) waiting. If cancelled or
     * attempt to set waitStatus fails, wake up to resync (in which
     * case the waitStatus can be transiently and harmlessly wrong).
     */
    // 当前节点入队列（头节点）
    Node p = enq(node);
    int ws = p.waitStatus;
    // 设置SIGNAL状态或唤醒
    if (ws > 0 || !p.compareAndSetWaitStatus(ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

# ReentrantLock
ReentrantLock是AbstractQueuedSynchronizer的典型子类实现，分为公平锁和非公平锁两种模式。

## Sync
```java
abstract static class Sync extends AbstractQueuedSynchronizer {

    /**
     * Performs non-fair tryLock.  tryAcquire is implemented in
     * subclasses, but both need nonfair try for trylock method.
     */
    @ReservedStackAccess
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        // state == 0，锁空闲
        if (c == 0) {
            // CAS操作获取锁
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 锁重入
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            // 无需CAS操作，因为锁的持有者是当前线程
            setState(nextc);
            return true;
        }
        return false;
    }

    @ReservedStackAccess
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        // 非锁的持有者线程不能释放锁
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        // state归零，锁释放
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        // 先设置exclusiveOwnerThread后设置state
        setState(c);
        return free;
    }
}
```

## NonfairSync
```java
static final class NonfairSync extends Sync {
    protected final boolean tryAcquire(int acquires) {
        // CAS操作设置state抢占式获取锁
        return nonfairTryAcquire(acquires);
    }
}
```

## FairSync
```java
static final class FairSync extends Sync {
    /**
     * Fair version of tryAcquire.  Don't grant access unless
     * recursive call or no waiters or is first.
     */
    @ReservedStackAccess
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        // 锁空闲
        if (c == 0) {
            // 检查等待队列是否为空
            if (!hasQueuedPredecessors() &&
                // CAS操作获取锁
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 锁重入
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```