---
title: Java 内存模型
author: zhangzangqian
date: 2018-11-03 21:00:00 +0800
categories: [技术]
tags: [Java, 并发]
math: true
mermaid: true
---

## 概述

衡量一个服务性能高低好坏，每秒事务处理数（Transactions Per Seconds，TPS）是最重要的指标之一，它代表着一秒内服务端平均能响应的请求总数，而 TPS 值与程序的并发能力又有非常密切的关系。对于计算量相同的任务，程序线程并发协调的越有条不紊，效率自然就会越高；反之，线程之间频繁阻塞甚至死锁，将会大大降低程序的并发能力。

Java 语言和虚拟机提供了许多工具，把并发编程的门槛降低了不少。并且各种中间件服务器、各类框架都努力得替程序员处理尽可能多的线程并发细节，使得程序员在编码时能更关注业务逻辑，而不是花费大部分时间去关注此业务会同时被多少人调用、如何协调硬件资源。无论语言、中间件和框架如何现金，开发人员都不能期望他们能独立完成所有并发处理的事情，了解并发的内幕也是称为一个高级程序员不可缺少的课程。

## 硬件的效率与一致性

绝大数的运算任务不可能只靠处理机“计算”就能完成，处理器至少要与内存交互，如读取运算数据、存储运算结果等，这个 I/O 操作是很难消除的（无法仅靠寄存器来完成所有运算任务）。由于计算机的存储设备与处理器的运算速度有几个数量级的差距，所以现代计算机系统都不得不加入一层读写速度尽可能接近处理器运算速度的高速缓存（Cache）来作为与内存与处理器之间的缓冲：将运算需要使用到的数据复制到缓存中，让计算能快速进行，当运算结束后再从缓存同步回内从之中，这样处理器就无需等待缓慢的内存读写了。

基于高速缓存的存储交互很好的解决了处理器与内存的速度矛盾，但是也为计算机系统带来更高的复杂的，因为它引入了一个新的问题：缓存一致性（Cache Coherence）。在多处理器系统中，每个处理器都有自己的告诉缓存，而它们又共享同一主内存（Main Memory）。当多个处理器的运算任务都涉及同一块主内存区域时，将可能导致各自的缓存数据不一致，为了解决一致性问题，需要各个处理器访问缓存时都遵循一些协议，在读写时需要根据协议来进行操作，这类协议有 MSI、MESI（Illinois Protocol）、MOS、Synapse、Firefly 及 Drago Protocol 等。“内存模型”一词可以理解为在特定的操作协议下，对特定的内存或高速缓存进行读写访问的过程抽象。不同架构的物理机器可以拥有不一样的内存模型，而 Java 虚拟机也有自己的内存模型，并且这里介绍的内存访问操作与硬件的缓存访问操作具有很高的可比性。

![处理器、高速缓存、主内存间的交互关系](/assets/img/interaction.jpeg)

另外使得处理器内部的运算单元尽可能被充分利用，处理器可能会对输入代码进行乱序执行（Out-Of-Order Execution）优化，处理器会在计算之后将乱序执行的结果重组，保证该结果与顺序执行的结果是一致的，但并不保证程序中各个语句计算的先后顺序与输入代码的顺序一致，因此，如果存在一个计算任务以来另外一个计算任务的中间结果，那么气顺序并不能靠代码的先后顺序来保证。与处理器的乱序执行优化类似，Java 虚拟机的即时编译器中也有类似的执行重排序（Instruction Reorder）优化。

## Java 内存模型

**Java 虚拟机规范中试图定义一种 Java 内存模型（Java Memory Model，JMM）来屏蔽掉各种硬件和操作系统的内存访问差异，以实现让 Java 程序在各种平台下都能达到一致的内存访问效果。**

### 主内存和工作内存

**Java 内存模型的主要目标是定义程序中各个变量的访问规则，即在虚拟机中将变量存储到内存和从内存中取出变量这样的底层细节。**这里的变量与 Java 编程中所说的变量有所区别，它包括了实例字段、静态字段和构成数组对象的元素，但不包括局部变量表与方法参数，因为后者都是线程私有的，不会被共享，自然不会存在竞争问题。

Java 内存模型规定了所有的变量都存储在主内存（Main Memory）中（此处的主内存与介绍物理硬件时的主内存名字一样，两者也可以互相类比，但此处的尽是虚拟机内存的一部分）。每条线程都有自己的工作内存（Working Memory，可与前面讲的处理器告诉缓存类比），线程的工作内存保存了被该线程使用到的变量的主内存副本拷贝，线程对变量的所有操作（读取、赋值等）都必须在工作内存中进行，而不能直接读写主内存中的变量，不同的香橙之间也无法访问对方内存中的变量，线程间变量值得传递均需要通过主内存来完成，线程、主内存、工作内存三者的交互关系如下图所示。

![线程、主内存、工作内存三者的交互关系](/assets/img/xzgjhgx.jpeg)

**这里所讲的主内存、工作内存与 Java 内存区域中的 Java 堆、栈、方法区等并不是同一层次的内存划分，这两者基本上是没有关系的，如果两者一定要勉强对应起来，那从变量、主内存、工作内存的定义来看，主内存主要对应于 Java 堆中的对象实例数据部分，而工作内存则对应于虚拟机栈中的部分区域。从更低层次上说，主内存就直接对应于物理硬件的内存，而为了获取更好的运行速度，虚拟机（甚至是硬件系统本身的优化措施）可能会让工作内存优先存储与寄存器和高速缓存中，因为程序运行时主要访问读写的是工作内存。**

### 内存间交互操作

关于主内存与工作内存之间具体的交互协议，即一个变量如何从主内存拷贝到工作内存、如何从工作内存同步回主内存之类的实现细节，Java 内存模型中定义了以下 8 种操作来完成，虚拟机是现实必须保证每一种操作都是原子的、不可再分的（对于 double 和 long 类型的变量来说，load、store、read 和 write 操作在某些平台上允许例外）

- lock（锁定）：作用于主内存的变量，它把一个变量标识为一条线程独占的状态。
- unlock（解锁）：作用于主内存的变量，它把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定。
- read（读取）：作用于主内存的变量，它把一个变量的值从主内存传输到线程的工作内存中，以便随后的 load 动作使用。
- load（载入）：作用于工作内存的变量，它把 read 操作从主内存得到的变量值放入工作内存的变量副本中。
- use（使用）：作用于工作内存的变量，它把工作内存中一个变量的值传递给执行引擎，每当虚拟机遇到一个需要使用到的变量的值的字节码指令时将会执行这个操作。
- assign（赋值）：作用于工作内存的变量，它把一个执行引擎接收到的值赋给工作内存的变量，每当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作。
- store（存储）：作用于工作内存的变量，它把工作内存中一个变量的值传送到主内存中，以便随后的 write 操作使用。
- write（写入）：作用于主内存的变量，它把 store 操作从主内存中得到的变量的值放入主内存的变量中。

如果要把一个变量从主内存赋值到工作内存，那就要顺序地执行 read 和 load 操作，如果要把变量从工作内存同步回主内存，就要顺序地执行 store 和 write 操作。**Java 内存模型要求上述两个操作必须按顺序执行，而没有保证是连续执行。**也就是说，read 和 load 之间、store 和 write 之间是可插入其他指令的，如对主内存中的变量 a、b 进行访问时，一种可能出现顺序是 read a、read b、load b、load a。除此之外，Java 内存模型还规定了在执行上述 8 种基本操作时必须满足如下规则：

- 不允许 read 和 load、store 和 write 操作之一单独出现，即不允许一个变量从主内存读取了但工作内存不接受，或者从工作内存发起回写了但主内存不接受的情况出现。
- 不允许一个线程丢弃它的最近的 assign 操作，即变量在工作内存中改变了之后必须把改变化同步回主内存。
- 不允许一个线程无原因地（没有发生过任何 assign 操作）把数据从线程的工作内存同步回主内存中。
- 一个新的变量只能在主内存中“诞生”，不允许在工作内存中直接使用一个未被初始化（load 或者 assign）的变量，换句话说，就是对一个变量实施 use、store 操作之前，必须先执行过了 assign 和  load 操作。
- 一个变量在同一时刻只允许一条线程对其进行 lock 操作，但 lock 操作可以被同一条线程重复执行多次，多次执行 lock 后，只要执行相同次数的 unlock 操作，变量才会被解锁。
- 如果对一个变量执行 lock 操作，那将会清空工作内存中次变量的值，在执行引擎使用这个变量前，需要重新执行 load 或 assign 操作初始化变量的值。
- 如果一个变量实现没有被 lock 操作锁定，那就不允许对它进行 unlock 操作，也不允许去 unlock 一个被其他线程锁定住的变量。
- 对一个变量执行 unlock 操作之前，必须先把此变量同步回主内存中（执行 store、write 操作）。

这 8 种内存访问操作以及上述规则限定，再加上稍后介绍的对 volatile 的一些特殊规定，就已经完全确定了 Java 程序中哪些内存访问操作在并发下是安全的。由于这种定义相当严谨担忧十分繁琐，实践起来很麻烦，所以在后面介绍这种定义的一个判断原则：现行发生原则（happens-before），用来确定一个访问在并发环境下是否安全。

### 对于 volatile 型变量对的特殊规则

当一个变量定义为 volatile 之后，它将具备两种特性：

- **保证此变量对所有线程的可见性，这里的“可见性”是指当一条线程修改了这个变量的值，新值对于其他线程来说是可以立即得知的。而普通变量不能做到这一点。**
- **进行指令重新排序优化，普通变量仅仅会保证在该方法的执行过程中所有以来赋值结果的地方都能获取到正确的结果，而不能保证变量赋值操作的顺序与程序代码中执行顺序一致。因此在一个线程的方法执行过程中无法感知到这点，这也就是 Java 内存模型中描述的所谓的“线程内表现为串行的语义”（Within-Thread As-If-Serial Semantics）。**

关于 volatile 变量的可见性，经常会被开发人员误解，认为一下描述成立：“volatile 变量对所有线程时立即可见的，对 volatile 变量在各个线程中是一直的，所以基于 volatile 变量的运算在并发下是安全的”。这句话的论据部分并没有错，但是其论据并不能得出“基于 volatile 变量的运算在并发下是安全的”这个结论。volatile 变量在各个线程的工作内存中不存在一致性问题（在各个线程的工作内存中， volatile 变量也可以存在不一致的情况，但由于每次使用之前都要先刷新，执行引擎看不到不一致的情况，因此可以认为不存在一致性问题），但是 Java 里面的运算并非原子操作，导致 volatile 变量的运算在并发环境下一样是不安全的，请看如下代码。

```java
/**
* volatile 变量自增运算测试
*/
public class VolatileTest {

    public static volatile int race = 0;

    public static void increase() {
        race++;
    }

    private static final int THREADS_COUNT = 20;

    public static void main(String[] args) {
        Thread[] threads = new Thread[THREADS_COUNT];
        for (int i = 0; i < THREADS_COUNT; i ++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 10000; j ++) {
                    increase();
                }
            });
            threads[i].start();
        }
        // 等待所有累加线程都结束
        while(Thread.activeCount() > 1) {
            Thread.yield();
        }
        System.out.println(race);
    }

}
```

这段代码如果能够正确并发的话，最后输出的结果应该是 200000。但运行完这段代码之后，并不会获得期望的结果，而且会发现每次运行程序，输出的结果都不一样，都是一个小于 200000 的数字，这是为什么呢?

我们用 javap 反编译这段代码之后会得到如下代码清单，发现只有一行代码的 increase() 方法在 Class 文件中是有 4 条字节码指令构成的（return 指令不是由 race++ 产生的，这条指令可以不计算），从字节码层面上很容易就分析出并发失败的原因了：**当 getstatic 指令把 race 的值渠道操作栈顶是，volatile 关键字保证了 race 的值此时是正确的，但是在执行 iconst_1、iadd 这些指令时，其他线程可能已经把 race 的值加大了，而且在操作栈顶的值就变成了过期的数据，所以 putstatic 指令执行后就可能把较小的 race 值同步回主内存之中。**

```java
public static void increase();
    Code:
        Stack=2,Locals=0,Args_size=0
        0:  getstatic   #13; //Field race:I
        3:  iconst_1
        4:  iadd
        5:  putstatic   #13; //Field race:I
        8:  return
    LineNumberTable:
        line 14:0
        line 15:8
```

由于 volatile 变量只能保证可见性，在不符合以下两条规则的运算场景中，我们仍然要通过加锁（使用 synchornized 或者 java.util.concurrent 中的原子类）来保证原子性。

- 运算结果并不依赖变量的当前值，或者能够确保只有单一的线程修改变量的值。
- 变量不需要与其他的状态变量共同参与不变约束。

如下代码所示的场景就很适合使用 volatile 变量来控制并发，当shutdown() 方法被调用时，能保证所有线程中执行的 doWork() 方法都立即停下来。

```java
volatile boolean shutdownRequested;

public void shutdown() {
    shutdownRequested = true;
}

public void doWork() {
    while(!shutdownRequested) {
        // do stuff
    }
}
```

我们继续通过一个例子来看看为何指令重排序会干扰程序的并发执行，演示代码如下：

```java
Map configOptions;
char[] configText;
// 此变量必须定义为 volatile
volatile boolean initialized = false;

// 假设一下代码在线程 A 中执行
// 模拟读取配置信息，当读取完成后将 initialized 设置为 true 已通知其他线程配置可用
configOptions = new HashMap();
configText = readConfigFile(fileName);
processConfigOptions(configText, configOptions);
initialized = true;

// 假设以下代码在线程 B 中执行
// 等待 initialized 为 true，代表线程 A 已经把配置信息初始化完成
while(!initialized) {
    sleep();
}

// 使用线程 A 中初始化好的配置信息
doSomethingWithConfig();
```

以上代码描述的场景十分常见，只是我们在处理配置文件时一般不会发现并发而已。如果定义 initialized 变量没有使用 volatile 修饰，就可能由于指令重排序的优化，导致位于线程 A 中最后一段代码 “initialized = true” 被提前执行（这里虽然使用 Java 作为伪代码，但所指的重排序优化师机器级的操作优化，提前执行时是指这句话对应的汇编代码被提前执行），这样在线程 B 中使用配置信息的代码就可能出现错误，而 volatile 关键字则可以避免此类情况的发生。

**volatile 变量读操作的性能消耗与普通变量几乎没什么差别，但是写操作则可能会慢一些，因为它需要在本地代码中插入许多内存屏障指令来保证处理器不会发生乱序执行。不过即便如此，大多数场景下 volatile 的总开销仍然要比锁低，我们在 volatile 与锁之中选择的唯一依据仅仅是 voaltile 的语义是否满足使用场景的需求。**

### 原子性、可见性与有序性

Java 内存模型是围绕着并发过程中如何处理原子性、可见性和有序性这 3 个特征来建立的。

1. 原子性（Atomicity）：由 Java 内存模型来直接保证的原子性操作报错 read、load、assign、use、store 和 write，我们大致可以认为基本数据类型的访问读写是具备原子性的。Java 内存模型还提供了 lock 和 unlock 操作来满足这种需求，尽管虚拟机未把 lock 和 unlock 操作直接提供给用户使用，但是操作却提供了更高层次的字节码指令 monitorenter 和 monitorexit 来隐式地使用这两个操作，这两个字节码指令反映到 Java 代码块中就是同步块—synchronized 关键字，因此在 synchronized 块之间的操作也具备原子性。

2. 可见性（Visibility）：可见性是指当一个线程修改了共享变量的值，其他线程能够立即得知这个修改。volatile 保证了多线程操作时变量的可见性。除了 volatile 职位，Java 还有两个关键字能实现可见性，即 synchronized 和 final。

3. 有序性（Ordering）：Java 内存模型的有序性在讲解 volatile 时也详细讨论过了，Java 程序中天然的有序性可以总结为一句话：如果在本线程内观察，所有的操作都是有序的；如果在一个线程中观察另一个线程：所有的操作都是无序的。前半句是指“线程内变现为串行的语义”（Within-Thread As-If-Serial Semantics），后半句是指“指令重排序”现象和“工作内存与主内存同步延迟”现象。

### 先行发生原则（happens-before）

**先行发生原则是判断数据是否竞争、线程是否安全的主要依据，依靠这个原则，我们可以通过几条规则一揽子解决并发环境下两个操作之间是否可能存在冲突的所有问题。**

**先行发生原则指的是 Java 内存模型中定义的两项操作之间的偏序关系，如果说操作 A 先行发生与操作 B，其实就是说发生操作 B 之前，操作 A 产生的影响可能被操作 B 观察到，“影响”包括修改了内存中共享变量的值、发送了消息、调用了方法等。**

下面是 Java 内存模型下一些“天然的”现行发生关系，这些先行发生关系无须任何同步器协助就已经存在，可以在编码中直接使用。**如果两个操作之间的关系不在此列，并且无法从以下规则推导出来，他们就没有顺序性保障，虚拟机可以对他们随意地进行重排序。**

- 程序次序规则（Program Order Rule）：在一个线程池内，按照程序代码执行，书写在前面的操作现行发生于书写在后面的操作。准确的说，应该是控制流顺序而不是程序代码顺序，因为要考虑分支、循环等结构。
- 管程锁定规则（Monitor Lock Rule）：一个 unlock 操作现行发生于后面对同一个锁的 lock 操作。这里必须强调的是同一个锁，而“后面”是指的时间上的先后顺序。
volatile 变量规则（Volatile Variable Rule）：对一个 volatile 变量的写操作先行发生与后面对这个变量的读操作，这里的“后面”指的是时间上的先后顺序。
- 线程启动规则（Thread Start Rule）：Thread 对象的 start() 方法现行发生于对此线程的每一个动作。
- 线程终止规则（Thread Termination Rule）：线程中的所有操作都先行发生于对此线程的终止监测，我们可以通过 Thread.join() 方法结束、Thread.isActive() 的返回值等手段监测线程已经终止执行。
- 线程终端规则（Thread Interruption Rule）：对县城 interrupt() 方法的调用优先发生于被中断线程的代码检测到中断事件的发生，可以通过 Thread.interrupted() 方法检测到是否有中断发生。
- 对象终结规则（Finalizer Rule）：一个对象的初始化完成（构造函数执行结束）先行发生于它的 finalize() 方法的开始。
- 传递性（Transitivity）：如果操作 A 先行发生于操作 B，操作 B 先行发生于操作 C，那就可以得出操作 A 先行发生于操作 C 的结论。

**一个操作“时间上的先发生”不代表这个操作会是“先行发生”，一个操作的“先行发生”也不一定是“时间上的先发生”！！！一个典型的例子就是多次提到的“指令重排序”。时间先后顺序与先行发生原则之间基本没有太大的关系，所以我们衡量并发安全问题的时候不要受到时间顺序的干扰，一切必须以先行发生原则为准。**