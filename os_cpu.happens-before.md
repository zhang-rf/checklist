# 参考
Java语言规范JLS-17：[https://docs.oracle.com/javase/specs/jls/se11/html/jls-17.html](https://docs.oracle.com/javase/specs/jls/se11/html/jls-17.html)

JSR-133开发者手册：[http://gee.cs.oswego.edu/dl/jmm/cookbook.html](http://gee.cs.oswego.edu/dl/jmm/cookbook.html)

OpenJDK版本：jdk11-1ddf9a99e4ad

x86存储结构：[https://mechanical-sympathy.blogspot.com/2013/02/cpu-cache-flushing-fallacy.html](https://mechanical-sympathy.blogspot.com/2013/02/cpu-cache-flushing-fallacy.html)

# Happens-before语义
Java语言规范（JLS-17.4.5）定义了内存操作（例如，读取和写入共享变量）之间的关系。仅当满足`happens-before`关系时，才能保证一个线程写入的结果对另一线程可见。

# x86存储结构
1. 寄存器
2. 排序缓冲（MOB），由load buffer和store buffer组成
3. 本地缓存L1、L2，共享缓存L3
4. 主存
5. ...

CPU使用数据时，需要将主存中的数据加载到寄存器中，该寄存器用于算术运算并由机器指令进行操作。之后，通常通过相同的指令将操作数据存储回主存。

为了弥补主存与寄存器之间的巨大时延差距，在两者之间引入了高速缓存，本地缓存L1、L2是核内独占的，这意味着不同核心之间存在数据一致性问题，核内缓存之间的一致性问题主要由MESI协议完成。

为了减少由于保持一致性而导致的性能损失，同时尽可能的保证“顺序”执行，又引入了排序缓冲以异步读取和写入缓存。

# 乱序
## 编译器
编译器（此处指本地代码编译器，例如JIT）在不改变单线程语义的前提下，可以对指令重新排序，以提高程序的执行速度。

## CPU
一种情况是，如果一个线程修改了共享变量（store）然后又读取了该变量（load），则看到的值可能是上次写入的值，而不是缓存中的最新值，这将隐式导致乱序。

## 内存屏障（Memory barrier）与栅栏（Fence）
### 内存屏障
```C++
__asm__ volatile ("" : : : "memory");
```
这是一个空的汇编指令，意味着该指令实际上是一个标识符，目的是告诉编译器该指令前后的代码顺序不能重排序，并且编译器在该指令之后需要丢弃寄存器中存储的值并重新读取，即内存屏障名称的由来。

### 栅栏
```C++
__asm__ volatile ("lock; addl $0,0(%%rsp)" : : : "cc", "memory");
```
在x86体系结构中，`lock`指令可用于防止指令前后的load/store操作被重排序。值得一提的是，CAS指令`cmpxchg`同样具有此效果。

## JVM内存屏障的定义
JSR-133定义了4种类型的内存屏障。

`LoadLoad`、`StoreStore`、`LoadStore`、`StoreLoad`

> hotspot\os_cpu\linux_x86\orderAccess_linux_x86.hpp
```C++
// A compiler barrier, forcing the C++ compiler to invalidate all memory assumptions
static inline void compiler_barrier() {
  __asm__ volatile ("" : : : "memory");
}

inline void OrderAccess::loadload()   { compiler_barrier(); }
inline void OrderAccess::storestore() { compiler_barrier(); }
inline void OrderAccess::loadstore()  { compiler_barrier(); }
inline void OrderAccess::storeload()  { fence();            }

inline void OrderAccess::acquire()    { compiler_barrier(); }
inline void OrderAccess::release()    { compiler_barrier(); }

inline void OrderAccess::fence() {
   // always use locked addl since mfence is sometimes expensive
#ifdef AMD64
  __asm__ volatile ("lock; addl $0,0(%%rsp)" : : : "cc", "memory");
#else
  __asm__ volatile ("lock; addl $0,0(%%esp)" : : : "cc", "memory");
#endif
  compiler_barrier();
}
```
可以看出，除了`StoreLoad`以外，其他的都是编译器内存障碍，这是因为x86存储结构是强一致的存储结构。出于前面所诉的情况，`StoreLoad`需要使用栅栏。

# Happens-before操作
* 监视器（monitor）上的解锁`happens-before`该监视器上的每个后续锁定。
* 对`volatile`字段的写入`happens-before`该字段的每次后续读取。
* 对`Thread.start()`的调用`happens-before`该线程中的任何操作。
* 线程中的所有操作都`happens-before`任意其他线程对该线程的`Thread.join()`调用。
* 所有对象的初始化`happens-before`后续的其他操作。

同时，经过先前的研究，我们可以得知`happens-before`具有传递性，即`happens-before`之前的操作对于后续操作也是可见的。

基于这些基本的`happens-before`操作，JUC下的各种类提供了更高级别的 `happens-before`操作。详见`java.base\share\classes\java\util\concurrent\package-info.java`

# FAQ
```java
public class Main {

    static boolean done;

    public static void main(String[] args) throws InterruptedException {
        thread1();
        Thread.sleep(1);
        thread2();
    }

    static void thread1() {
        new Thread(() -> {
            while (!done) ;
            System.out.println("Thread1 exit");
        }).start();
    }

    static void thread2() {
        new Thread(() -> {
            done = true;
        }).start();
    }
}
```
Q：为什么“thread1”不能退出？

A：由于已将“done”变量读入寄存器，因此无法感知到主存中的更改。更为激进的编译器可能将其优化为`while（true）;`。

为“done”变量添加`volatile`关键字之后，在load操作之前插入了内存屏障，以确保从主存中读取并阻止编译器的优化。

---

```java
public class Main {

    static boolean done;

    public static void main(String[] args) throws InterruptedException {
        thread1();
        Thread.sleep(1);
        thread2();
    }

    static void thread1() {
        new Thread(() -> {
            while (!done) {
                try {
                    Thread.sleep(1);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                }
            }
            System.out.println("Thread1 exit");
        }).start();
    }

    static void thread2() {
        new Thread(() -> {
            done = true;
        }).start();
    }
}
```

Q：在循环中添加`sleep()`语句后，为什么没有`volatile`关键字，“thread1”也能感知到更改？

A：因为`sleep()`的实现中插入了栅栏，猜测与前一行代码`slp->reset()`的写入有关。

> hotspot\os\posix\os_posix.cpp
```c++
int os::sleep(Thread* thread, jlong millis, bool interruptible) {
  assert(thread == Thread::current(),  "thread consistency check");

  ParkEvent * const slp = thread->_SleepEvent ;
  slp->reset() ;
  OrderAccess::fence() ;

  // ...
}
```

---

```java
public class Main {

    static boolean done;
    volatile static boolean anything = true;

    public static void main(String[] args) throws InterruptedException {
        thread1();
        Thread.sleep(100);
        thread2();
    }

    static void thread1() {
        new Thread(() -> {
            while (anything && !done) ;
            System.out.println("Thread1 exit");
        }).start();
    }

    static void thread2() {
        new Thread(() -> {
            done = true;
        }).start();
    }
}
```
Q：为什么没有“done”变量没有使用`volatile`关键字也能感知到更改？

A：因为`volatile`变量的内存屏障刷新了寄存器，所以`volatile`关键字不仅影响声明的变量本身，即`happens-before`的传递性。
