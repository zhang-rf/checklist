OpenJDK版本：jdk11-1ddf9a99e4ad

基础知识：HashMap、TreeMap、LongAdder

# 原理
充分利用CAS操作和并行计算的优势，所有操作都可以并发执行，只有在发生哈希碰撞和一些中间操作时才会有锁开销。

# 核心字段

## TREEIFY_THRESHOLD
哈希节点树化所需的链表长度阈值
```java
static final int TREEIFY_THRESHOLD = 8;
```

## UNTREEIFY_THRESHOLD
扩容时已树化哈希节点回归链表的阈值
```java
static final int UNTREEIFY_THRESHOLD = 6;
```

## MIN_TREEIFY_CAPACITY
哈希节点树化前，哈希表容量最小值
```java
static final int MIN_TREEIFY_CAPACITY = 64;
```

## MIN_TRANSFER_STRIDE
扩容时单次迁移的最小哈希节点数
```java
private static final int MIN_TRANSFER_STRIDE = 16;
```

## MOVED、TREEBIN、RESERVED
三种预定义哈希值：已迁移节点（扩容时）、已树化节点、computeIfAbsent占位节点
```java
static final int MOVED     = -1; // hash for forwarding nodes
static final int TREEBIN   = -2; // hash for roots of trees
static final int RESERVED  = -3; // hash for transient reservations
```

## table
哈希数组
```java
transient volatile Node<K,V>[] table;
```

## nextTable
扩容后的哈希数组（中间变量）
```java
private transient volatile Node<K,V>[] nextTable;
```

## baseCount/cellsBusy/counterCells
哈希size，详见LongAdder
```java
private transient volatile long baseCount;
private transient volatile int cellsBusy;
private transient volatile CounterCell[] counterCells;
```

## sizeCtl
正数：
1. 初始化前的哈希容量
2. 下一次扩容时的哈希size

负数：
1. -1表示正在初始化哈希数组
2. 低16位存储正在扩容的线程数+1，高15位存储扩容版本号（resizeStamp）
```java
private transient volatile int sizeCtl;
```

## transferIndex
中间变量，下一次迁移时的数组起始索引（从后往前）
```java
private transient volatile int transferIndex;
```

# 核心方法

## spread(int)
将hashCode的高位分散到低位，以减少小容量哈希表的冲突概率
```java
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```

## tableSizeFor(int)
返回不小于所需容量的2的N次幂，便于哈希位运算
```java
private static final int tableSizeFor(int c) {
    int n = -1 >>> Integer.numberOfLeadingZeros(c - 1);
    return (n < 0) ? 1 : (n >= 64) ? 64 : n + 1;
}
```

## get(Object)
返回指定键映射的值，详见HashMap
```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

## putVal(K, V, boolean)
```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        // 分散hashCode
        int hash = spread(key.hashCode());
        // 链表长度
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh; K fk; V fv;
            if (tab == null || (n = tab.length) == 0)
                // 初始化哈希表，详见initTable()
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                // 对应位置无节点时，尝试CAS操作设置节点
                if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
                    break;                   // no lock when adding to empty bin
            }
            // 对应位置为ForwardingNode，表示正在进行扩容，则协助扩容
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else if (onlyIfAbsent // check first node without acquiring lock
                     && fh == hash
                     && ((fk = f.key) == key || (fk != null && key.equals(fk)))
                     && (fv = f.val) != null)
                return fv;
            else {
                V oldVal = null;
                // 哈希冲突时，阻止并发写入
                synchronized (f) {
                    // 加锁后二次检查
                    if (tabAt(tab, i) == f) {
                        // 节点hash值大于0代表链表节点
                        if (fh >= 0) {
                            binCount = 1;
                            // 链表操作，详见HashMap
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key, value);
                                    break;
                                }
                            }
                        }
                        // 红黑树操作，详见HashMap、TreeMap
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                        // computeIfAbsent递归调用，抛出异常
                        // 详见https://bugs.openjdk.java.net/browse/JDK-8062841
                        else if (f instanceof ReservationNode)
                            throw new IllegalStateException("Recursive update");
                    }
                }
                if (binCount != 0) {
                    // 链表转红黑树
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        // size++，并检查是否需要扩容，详见addCount()
        addCount(1L, binCount);
        return null;
    }
```

## clear()
清空哈希表（并发执行）
```java
public void clear() {
    long delta = 0L; // negative number of deletions
    int i = 0;
    Node<K,V>[] tab = table;
    // 遍历删除
    while (tab != null && i < tab.length) {
        int fh;
        Node<K,V> f = tabAt(tab, i);
        // 空节点，检查下一节点
        if (f == null)
            ++i;
        // 正在扩容，则协助扩容
        else if ((fh = f.hash) == MOVED) {
            tab = helpTransfer(tab, f);
            i = 0; // restart
        }
        else {
            // 节点加锁
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> p = (fh >= 0 ? f :
                                   (f instanceof TreeBin) ?
                                   ((TreeBin<K,V>)f).first : null);
                    // 计算待删除节点元素数量
                    while (p != null) {
                        --delta;
                        p = p.next;
                    }
                    setTabAt(tab, i++, null);
                }
            }
        }
    }
    // 更新size
    if (delta != 0L)
        addCount(delta, -1);
}
```

## resizeStamp(int)
扩容邮戳，用作扩容版本号，解决并发扩容时CAS操作的ABA问题
```java
static final int resizeStamp(int n) {
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```

## initTable()
```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        // sizeCtl < 0，说明其他线程正在初始化哈希表，让出CPU时间等待下一次重试
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        // -1表示正在初始化
        else if (U.compareAndSetInt(this, SIZECTL, sc, -1)) {
            try {
                // 二次检查
                if ((tab = table) == null || tab.length == 0) {
                    // sizeCtl默认为0或初始化容量
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // 扩容size = table.length x 0.75
                    sc = n - (n >>> 2);
                }
            } finally {
                // 设置或还原sizeCtl
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

## addCount(long, int)
```java
private final void addCount(long x, int check) {
    // 计算size，详见LongAdder
    CounterCell[] cs; long b, s;
    if ((cs = counterCells) != null ||
        !U.compareAndSetLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell c; long v; int m;
        boolean uncontended = true;
        if (cs == null || (m = cs.length - 1) < 0 ||
            (c = cs[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSetLong(c, CELLVALUE, v = c.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    // 检查扩容
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        // size > sizeCtl则进行扩容
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            // 获取扩容版本号（resizeStamp）
            int rs = resizeStamp(n);
            // sc < 0，当前正在扩容
            if (sc < 0) {
                /*
                 * 检查是否能协助扩容
                 * 1. 扩容版本号相同
                 * 2. 扩容未结束
                 * 3. 不超过最大扩容线程数
                 * 4. 此处有BUG，详见https://bugs.openjdk.java.net/browse/JDK-8214427
                 */
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                // 协助扩容
                if (U.compareAndSetInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            // 初始扩容
            else if (U.compareAndSetInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```

## transfer(Node<K,V>[], Node<K,V>[])
```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // 单次迁移节点数
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    // 初始扩容
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        // 迁移从数组尾部开始
        transferIndex = n;
    }
    int nextn = nextTab.length;
    // 占位节点，让其他线程发现正在扩容
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    // 前进到下一节点（从后往前）
    boolean advance = true;
    // 迁移全部完成
    boolean finishing = false; // to ensure sweep before committing nextTab
    // i：当前节点位置，bound：本次迁移边界
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        // 查找下一节点位置
        while (advance) {
            int nextIndex, nextBound;
            // i在边界内，或迁移完成
            if (--i >= bound || finishing)
                advance = false;
            // 所有节点迁移完成
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            // 设置本次迁移范围
            else if (U.compareAndSetInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        // 本次迁移完成
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            // 迁移全部完成，更新table、sizeCtl
            if (finishing) {
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            // 扩容线程数-1
            if (U.compareAndSetInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                // 最后一个扩容线程完成收尾工作
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        // 空节点CAS操作替换成ForwardingNode
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // ForwardingNode节点，说明已被迁移
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            // 锁住节点，防止并发写入
            synchronized (f) {
                // 加锁后二次检查
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                        // 给链表分成两组迁移，所有元素只会在原始位置或原始位置+n
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    // 红黑树节点迁移，详见HashMap
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```
