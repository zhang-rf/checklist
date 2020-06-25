OpenJDK版本：jdk11-1ddf9a99e4ad

# 参考

红黑树：[https://en.wikipedia.org/wiki/Red–black_tree](https://en.wikipedia.org/wiki/Red–black_tree)

# 原理
基于红黑树的Map实现，提供了顺序访问元素的功能，检索和增删元素的时间复杂度为log(n)。

# 红黑树

## 属性
1. 每个节点是红色或黑色
2. 根是黑色的。有时会忽略此规则。由于根始终可以从红色更改为黑色，但反之亦然，因此该规则对分析的影响很小。
3. 所有叶子（无）为黑色
4. 如果节点为红色，则其两个子节点均为黑色
5. 从给定节点到其任何后代NIL节点的每条路径都经过相同数量的黑色节点

## 操作

### 插入
插入后，如果不满足红黑树的属性，则需要进行以下调整

#### 设置颜色
1. 新插入的节点（N）是红色
2. 如果N是root，则设置为黑色
3. 如果parent和uncle都是红色
   1. 将parent和uncle设置为黑色
   2. 将grandparent设置为红色

#### 旋转
1. 如果N和parent不在同一侧，向N的方向旋转parent
2. 将parent设置为黑色
3. 将grandparent设置为红色
4. 向N的方向旋转grandparent

# 数据结构
```java
static final class Entry<K,V> implements Map.Entry<K,V> {
    K key;
    V value;
    Entry<K,V> left;
    Entry<K,V> right;
    Entry<K,V> parent;
    boolean color = BLACK;
}
```

# 核心方法

## getEntry(Object)
```java
final Entry<K,V> getEntry(Object key) {
    // Offload comparator-based version for sake of performance
    if (comparator != null)
        // 使用comparator查找
        return getEntryUsingComparator(key);
    if (key == null)
        throw new NullPointerException();
    @SuppressWarnings("unchecked")
        Comparable<? super K> k = (Comparable<? super K>) key;
    // 二叉树查找
    Entry<K,V> p = root;
    while (p != null) {
        int cmp = k.compareTo(p.key);
        if (cmp < 0)
            p = p.left;
        else if (cmp > 0)
            p = p.right;
        else
            return p;
    }
    return null;
}
```

## put(K, V)
```java
public V put(K key, V value) {
    Entry<K,V> t = root;
    // 第一个节点（根节点）
    if (t == null) {
        compare(key, key); // type (and possibly null) check
        root = new Entry<>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }
    int cmp;
    Entry<K,V> parent;
    // split comparator and comparable paths
    Comparator<? super K> cpr = comparator;
    // 使用comparator或者Comparable key
    if (cpr != null) {
        do {
            parent = t;
            cmp = cpr.compare(key, t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    else {
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
        // 从根节点开始遍历，找到要插入的位置，与二叉树没有区别
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    Entry<K,V> e = new Entry<>(key, value, parent);
    if (cmp < 0)
        parent.left = e;
    else
        parent.right = e;
    // 再平衡
    fixAfterInsertion(e);
    size++;
    modCount++;
    return null;
}
```

## fixAfterInsertion(Entry<K,V>)
```java
/** From CLR */
private void fixAfterInsertion(Entry<K,V> x) {
    // 新节点（X）是红色
    x.color = RED;
    // parent是红色
    while (x != null && x != root && x.parent.color == RED) {
        // 如果parent是左节点
        if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
            // uncle
            Entry<K,V> y = rightOf(parentOf(parentOf(x)));
            // 如果uncle是红色
            if (colorOf(y) == RED) {
                // 设置parent为黑色
                setColor(parentOf(x), BLACK);
                // 设置uncle为黑色
                setColor(y, BLACK);
                // 设置grandparent为红色
                setColor(parentOf(parentOf(x)), RED);
                // 向上检查（grandparent）
                x = parentOf(parentOf(x));
            } else {
                // 如果X是右节点
                if (x == rightOf(parentOf(x))) {
                    // 左旋转parent
                    x = parentOf(x);
                    rotateLeft(x);
                }
                // 设置parent为黑色
                setColor(parentOf(x), BLACK);
                // 设置grandparent为红色
                setColor(parentOf(parentOf(x)), RED);
                // 右旋转grandparent
                rotateRight(parentOf(parentOf(x)));
            }
        // 如果parent是右节点，同上
        } else {
            Entry<K,V> y = leftOf(parentOf(parentOf(x)));
            if (colorOf(y) == RED) {
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                x = parentOf(parentOf(x));
            } else {
                if (x == leftOf(parentOf(x))) {
                    x = parentOf(x);
                    rotateRight(x);
                }
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateLeft(parentOf(parentOf(x)));
            }
        }
    }
    // 根节点保持黑色
    root.color = BLACK;
}
```

未完待续...
