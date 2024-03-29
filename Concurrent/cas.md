# <center>CAS</center>


<br></br>

* 非阻塞、乐观锁。 
* CAS通过调用JNI代码实现。
* Java无法直接访问底层操作系统，而是通过native访问。类Unsafe提供硬件级别原子操作。

```java
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;
private volatile int value;
 
static {
  try {
     valueOffset = unsafe.objectFieldOffset(AtomicInteger.class.getDeclaredField("value"));
  } catch (Exception ex) { throw new Error(ex); }
}
```

Java的CAS使用处理器提供的机器级别原子指令。同时，volatile变量读写和CAS可实现线程通信。把这些特性整合，就形成整个concurrent包基石。

如果分析concurrent包源码，会发现通用化实现模式：
* 首先，声明共享变量为volatile；
* 然后，使用CAS原子条件更新来实现线程间同步；
* 同时，配合以volatile读写和CAS具有的volatile读写内存语义实现线程通信。

AQS，非阻塞数据结构和原子变量类（java.util.concurrent.atomic包中的类），都是这种模式：

![concurrent包的实现示意图](./Images/overall.png)

<br></br>



## 原理
----
sun.misc.Unsafe类的`compareAndSwapInt()`方法借助C调用CPU底层指令实现：

``` java
public final native boolean compareAndSwapInt(Object o, long offset, int expected, int x)
```

这是个本地方法调用，在openjdk中依次调用的c++代码为：unsafe.cpp，atomic.cpp和atomicwindowsx86.inline.hpp。下面是对应于intel x86处理器的源代码的片段：

```
// Adding a lock prefix to an instruction on MP machine
// VC++ doesn't like the lock prefix to be on a single line
// so we can't insert a label after the lock prefix.
// By emitting a lock prefix, we can define a label after it.
#define LOCK_IF_MP(mp) __asm cmp mp, 0  \
                       __asm je L0      \
                       __asm _emit 0xF0 \
                       __asm L0:

inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
  // alternative for InterlockedCompareExchange
  int mp = os::is_MP();
  __asm {
    mov edx, dest
    mov ecx, exchange_value
    mov eax, compare_value
    LOCK_IF_MP(mp)
    cmpxchg dword ptr [edx], ecx
  }
}
```

程序会根据当前处理器类型来决定是否为`cmpxchg`指令添加lock前缀。如果在多处理器上运行，就为`cmpxchg`指令加上lock前缀（`lock cmpxchg`）。反之，就省略lock前缀（单处理器自身会维护单处理器内的顺序一致性，不需要lock前缀提供的内存屏障效果）。

intel的手册对lock前缀的说明如下：

1. 确保对内存的读-改-写操作原子执行。如果要访问的内存区域在lock前缀指令执行期间已在处理器内部缓存中被锁定（即包含该内存区域的缓存行当前处于独占或修改状态），并且该内存区域被完全包含在单个缓存行中，那么处理器将直接执行该指令。由于在指令执行期间该缓存行会一直被锁定，其它处理器无法读/写该指令要访问的内存区域，因此能保证指令执行的原子性。这个操作过程叫做缓存锁定（cache locking）。但当多处理器之间的竞争程度很高或者指令访问的内存地址未对齐时，仍然会锁住总线。
2. 禁止该指令与之前和之后的读和写指令重排序。
3. 把写缓冲区中的所有数据刷新到内存中。

第2点和第3点所具有的内存屏障效果，足以同时实现volatile读写的内存语义，编译器不能对CAS与CAS前面和后面的任意内存操作重排序。

> volatile读写内存语义:
> 1. 编译器不会对volatile读与volatile读后面任意内存操作重排序；
> 2. 编译器不会对volatile写与volatile写前面任意内存操作重排序。

CPU锁有3种：
1. 处理器自动保证基本内存操作的原子性

    首先处理器保证基本内存操作原子性。处理器保证从系统内存当中读取或写入一个字节是原子的，意思是当处理器读取一个字节时，其他处理器不能访问这个字节内存地址。

2. 使用总线锁保证原子性

    如果多处理器同时对共享变量进行读改写（比如`i++`），那么共享变量操作完之后共享变量的值会和期望的不一致，处理器使用总线锁解决这个问题。所谓总线锁是使用处理器提供的LOCK＃信号，当处理器在总线上输出此信号时，其他处理器请求被阻塞，那么该处理器可独占使用共享内存。

3. 使用缓存锁保证原子性

    同一时刻只需保证对某个内存地址操作原子即可，但总线锁把CPU和内存通信锁住，使得锁定期间，其他处理器不能操作其他内存地址数据，所以某些场合下使用缓存锁定代替总线锁定进行优化。

    频繁使用内存会缓存在处理器的L1，L2 和L3里，那么原子操作可在处理器内部缓存中进行，并不需要总线锁。所谓缓存锁定是如果缓存在处理器缓存行中内存区域在LOCK操作期间被锁定，当它执行锁操作回写内存时，处理器不在总线上声言LOCK＃信号，而是修改内部的内存地址，并允许它的缓存一致性机制来保证操作的原子性。因为缓存一致性会阻止同时修改被两个以上处理器缓存的内存区域数据，当其他处理器回写已被锁定的缓存行的数据时会起缓存行无效。

    但是有两种情况处理器不会使用缓存锁定。第一种情况是当操作的数据不能被缓存在处理器内部，或操作的数据跨多个缓存行，则处理器会调用总线锁定。第二种情况是有些处理器不支持缓存锁定。

<br></br>



## CAS缺点
----
### ABA问题
如果一个值原来是A，变成B，又变成A，那么用CAS检查时会发现它值没有变化，但实际上变化了。解决思路是用版本号。在变量前面追加上版本号，A－B－A变成1A-2B－3A。

A-B-A解决方法是不再仅替换指向一个预期修改对象的指针，而是指针结合一个计数器，然后使用单个CAS操作来替换指针+计数器。Java 1.5提供了AtomicStampedReference类，利用这个类可使用一个CAS操作（`compareAndSet()`）自动替换一个引用和一个标记（stamp）。

<br>


### 只保证一个共享变量的原子操作
当对一个共享变量执行操作时，可使用循环CAS方式来保证原子操作。但对多个共享变量操作，循环CAS无法保证操作原子性。这时可以用锁，或把多个共享变量合并成一个共享变量来操作。

比如有两个共享变量`i＝2`，`j=a`，合并一下`ij=2a`，然后用CAS来操作`ij`。从Java1.5开始提供了`AtomicReference`类来保证引用对象之间的原子性。

<br>


### 本地延迟
所有CPU会共享一条系统总线，靠此总线连接主存。每个核都有自己一级缓存。

Core1和Core2可能同时把主存中某个位置的值Load到自己L1 Cache中。当Core1在自己L1 Cache中修改这个位置值时，通过总线，使Core2中L1 Cache对应的值失效。Core2发现自己L1 Cache中值失效（Cache命中缺失），通过总线从内存中加载该地址最新值，通过总线的来回通信称为Cache一致性流量。

如果Cache一致性流量过大，总线将成为瓶颈。当Core1和Core2中值再次一致时，称为Cache一致性。锁设计终极目标是减少Cache一致性流量。CAS恰好会导致Cache一致性流量。如果很多线程共享同一个对象，当某个Core CAS成功时必然会引起总线风暴，即本地延迟。本质上偏向锁是为了消除CAS，降低Cache一致性流量。

<br></br>
