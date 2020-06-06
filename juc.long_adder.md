OpenJDK版本：jdk11-1ddf9a99e4ad

# 原理
在AtomicLong的基础上，CAS操作失败时，将值的更新分散到Cells中；获取真实值时，再将Cells中的值累加起来。

# 核心字段
继承自Striped64。

## cells
```java
transient volatile Cell[] cells;
```
更新冲突时，变化的临时存放处。

## base
```java
transient volatile long base;
```
无更新冲突时，值的存放处，相当于AtomicLong的value。

## cellsBusy
```java
transient volatile int cellsBusy;
```
更新cells时的锁。

# 核心方法

## add(long)
```java
public void add(long x) {
    Cell[] cs; long b, v; int m; Cell c;
    // 如果cells已经初始化，或者base的CAS操作失败
    if ((cs = cells) != null || !casBase(b = base, b + x)) {
        // 未出现cell的竞争
        boolean uncontended = true;
        /**
         * 1. cells未初始化
         * 2. 随机获取的cell为空
         * 3. 对随机获取的cell进行CAS操作失败
         */
        if (cs == null || (m = cs.length - 1) < 0 ||
            (c = cs[getProbe() & m]) == null ||
            !(uncontended = c.cas(v = c.value, v + x)))
            longAccumulate(x, null, uncontended);
    }
}
```

## longAccumulate(long, LongBinaryOperator, boolean)
```java
final void longAccumulate(long x, LongBinaryOperator fn,
                          boolean wasUncontended) {
    // 当前线程的probe作为cell的key
    int h;
    if ((h = getProbe()) == 0) {
        ThreadLocalRandom.current(); // force initialization
        h = getProbe();
        wasUncontended = true;
    }
    // cellsBusy锁碰撞
    boolean collide = false;                // True if last slot nonempty
    done: for (;;) {
        Cell[] cs; Cell c; int n; long v;
        // cells已初始化
        if ((cs = cells) != null && (n = cs.length) > 0) {
            // 对应key下的cell为空
            if ((c = cs[(n - 1) & h]) == null) {
                // cellsBusy锁未被占用
                if (cellsBusy == 0) {       // Try to attach new Cell
                    Cell r = new Cell(x);   // Optimistically create
                    // 尝试CAS方式获取cellsBusy锁
                    if (cellsBusy == 0 && casCellsBusy()) {
                        try {               // Recheck under lock
                            Cell[] rs; int m, j;
                            // 获取cellsBusy锁后二次检查，成功后初始化对应cell
                            if ((rs = cells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                break done;
                            }
                        } finally {
                            // 释放cellsBusy锁
                            cellsBusy = 0;
                        }
                        continue;           // Slot is now non-empty
                    }
                }
                collide = false;
            }
            // 上一轮出现竞争时，重置后下轮重试
            else if (!wasUncontended)       // CAS already known to fail
                wasUncontended = true;      // Continue after rehash
            // 尝试CAS操作更新cell值
            else if (c.cas(v = c.value,
                           (fn == null) ? v + x : fn.applyAsLong(v, x)))
                break;
            // 如果cells已经扩容到核心数大小，或者cells被其他线程扩容时，下轮重试
            else if (n >= NCPU || cells != cs)
                collide = false;            // At max size or stale
            // collide == false时，短路下方扩容代码，下轮重试
            else if (!collide)
                collide = true;
            // 获取cellsBusy锁
            else if (cellsBusy == 0 && casCellsBusy()) {
                try {
                    // 获取cellsBusy锁后二次检查，成功后进行cells扩容
                    if (cells == cs)        // Expand table unless stale
                        cells = Arrays.copyOf(cs, n << 1);
                } finally {
                    // 释放cellsBusy锁
                    cellsBusy = 0;
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            // 更新当前线程的probe，作为下一个cell的key
            h = advanceProbe(h);
        }
        // cells未初始化，且cellsBusy锁未被占用
        else if (cellsBusy == 0 && cells == cs && casCellsBusy()) {
            try {                           // Initialize table
                // 获取cellsBusy锁后二次检查，成功后进行cells初始化
                if (cells == cs) {
                    Cell[] rs = new Cell[2];
                    rs[h & 1] = new Cell(x);
                    cells = rs;
                    break done;
                }
            } finally {
                // 释放cellsBusy锁
                cellsBusy = 0;
            }
        }
        // cells初始化冲突时，降级为base的CAS操作
        else if (casBase(v = base,
                         (fn == null) ? v + x : fn.applyAsLong(v, x)))
            break done;
    }
}
```
