# 参考
Java语言规范JLS-17：[https://docs.oracle.com/javase/specs/jls/se11/html/jls-17.html](https://docs.oracle.com/javase/specs/jls/se11/html/jls-17.html)

JSR-133开发者手册：[http://gee.cs.oswego.edu/dl/jmm/cookbook.html](http://gee.cs.oswego.edu/dl/jmm/cookbook.html)

Hotspot：jdk11-1ddf9a99e4ad

# happens-before 语义
Java语言规范JLS-17.4.5中，定义了关于内存操作（如读取和写入共享变量）之间的关系。一个线程写入的结果，仅在满足 happens-before 关系时才保证对另一个线程的读取可见。

# x86 CPU结构
存储器
1. 寄存器
2. load buffer、store buffer，合称排序缓冲 Memoryordering Buffers(MOB)
3. 本地缓存L1、L2，共享缓存L3
4. 内存

CPU首先将内存中的数据加载到寄存器中，将该寄存器用于算术运算并由机器指令进行操作。之后，通常通过相同的指令将操作后的数据存储回内存。为了弥补内存与寄存器之间巨大的时延差距，在两者之间引入了缓存，本地缓存L1、L2是核内独占的，这就意味着不同核心之间存在着数据一致性问题，而不同缓存之间需要保证一致性，主要通过MESI协议来完成。为了降低维持一致性带来的性能损耗，又引入了 load buffer 和 store buffer，将数据读写异步化。

# 指令乱序

## 编译器
编译器（此处指的是本地代码编译器，例如JIT）在不改变单线程语义的前提下，为了提高程序的运行速度，可以对指令进行重排序。

## CPU
load buffer 和 store buffer的引入，会导致 load/store 指令未完成时即执行下一条指令，也会隐式的造成指令重排序

## 内存屏障（memory barrier）与栅栏（fence）

### 内存屏障
```C++
__asm__ volatile ("" : : : "memory");
```
这是一条空的汇编指令，意味着这个指令其实是一个标识符，目的是告诉编译器不可以重排序这条指令前后代码的顺序，即形成了一个屏障，同时告诉编译器这条指令后的内存数据发生的修改，需要丢弃寄存器中保存的值重新读取，即内存屏障名称的由来。

### 栅栏
```C++
__asm__ volatile ("lock; addl $0,0(%%rsp)" : : : "cc", "memory");
```
X86架构中，使用lock前缀指令，可以保证指令前后的load/store操作不被重排序。值得一提的是，CAS的操作指令（cmpxchg）具有同样的效果。

## JVM内存屏障的定义
JSR-133中，定义了4种内存屏障
1. LoadLoad：禁止读取操作与读取操作之间的重排序
2. StoreStore：禁止读写入操作与写入操作之间的重排序
3. LoadStore：禁止读取操作与写入操作之间的重排序
4. StoreLoad：禁止写入操作与读取操作之间的重排序

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
可以看到，除了 StoreLoad，其他的都是编译器内存屏障，这是因为X86架构属于强一致性内存模型架构，这三种操作之间保证顺序性，只有 StoreLoad 需要栅栏。

# happens-before 操作
* 同步代码块的退出或Lock的解锁 happens-before 后续的进入同步代码块或Lock加锁
* 对volatile字段的写入 happens-before 后续对volatile字段的读取
* Thread.start() happens-before 启动线程中的后续操作
* Thread.join() 返回前的操作 happens-before Thread.join()返回后

同时，经过前面的研究可知，happens-before 具有传递性，即 happens-before 操作之前的操作同样对于后续可见

> 基于这些基本的 happens-before 动作，`java.util.concurrent`包下的各种类提供了更高层次的 happens-before 动作，详见`java.base\share\classes\java\util\concurrent\package-info.java`

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
            System.out.println("thread1 exit");
        }).start();
    }

    static void thread2() {
        new Thread(() -> {
            done = true;
        }).start();
    }
}
```
Q：为什么 thread1 无法退出？

A：因为 `done` 变量已经被读取到了寄存器中，无法感知内存中的修改，更为激进的编译器可能会优化成`while (true)`， `done` 变量添加 `volatile` 关键字后，即在load操作前插入了内存屏障，保证从内存中读取，也阻止了编译器的优化。

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
Q：为什么 thread1 在循环中添加了 sleep()，无需 `volatile` 关键字也能感知到变更？

A：因为 sleep() 实现中插入了栅栏（fence），猜测与前一行代码 `slp->reset()` 的写入有关。
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

```java
public class Main {

    static boolean done;
    volatile static boolean justABool = true;

    public static void main(String[] args) throws InterruptedException {
        thread1();
        Thread.sleep(100);
        thread2();
    }

    static void thread1() {
        new Thread(() -> {
            while (justABool && !done) ;
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
Q：为什么增加了 `volatile` 变量的读取即可保证 `done` 变量的可见性？

A：因为 `volatile` 变量的读取插入了内存屏障，刷新了寄存器中的值。所以 `volatile` 关键字影响的不仅仅是声明的变量本身。
