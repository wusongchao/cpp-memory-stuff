# Sequential Consistency

## Cache Coherence

众所周知，当一个执行中的程序的数据被多个执行流并发访问的时候，就会涉及到**同步（Synchronization）**的问题。同步的目的是保证不同执行流对共享数据并发操作的一致性。早在单核时代，使用锁或者原子变量就很容易达成这一目的。甚至因为CPU的一些访存特性，对某些内存对齐数据的读或写也具有原子的特性。

比如，在《Intel® 64 and IA-32 Architectures Software Developer’s Manual》的第三卷System Programming Guide的Chapter 8 Multiple-Processor Management里，就给出了这样的说明：

![intel1](http://www.0xffffff.org/images/40/guaranteed_atomic_operations.png)

http://www.cs.cmu.edu/~410-f10/doc/Intel_Reordering_318147.pdf

也就是说，有些内存对齐的数据的访问在CPU层面就是原子进行的（注意这里说的只是单次的读或者写，类似普通变量i的i++操作不止一次内存访问）。此时，环形队列（Ring buffer）这种数据结构在某些架构的单核CPU上，只有一个Reader和一个Writer的情况下是不需要额外同步措施的。原因就是read_index和writer_index的写操作在满足对齐内存访问的情况下是原子的，不需要额外的同步措施。注意这里我加粗了单核CPU这个关键字，那么到了多核心处理器的今天，该操作就不是原子了吗？不，依旧是原子的，但是出现了其他的干扰因素迫使可能需要额外的同步措施才能保证原本无锁代码的正确运行。

首先是现代编译器的代码优化和编译器指令重排可能会影响到代码的执行顺序。编译期指令重排是通过调整代码中的指令顺序，在不改变代码语义的前提下，对变量访问进行优化。从而尽可能的减少对寄存器的读取和存储，并充分复用寄存器。但是编译器对数据的依赖关系判断只能在单执行流内，无法判断其他执行流对竞争数据的依赖关系。就拿无锁环形队列来说，如果Writer做的是先放置数据，再更新索引的行为。如果索引先于数据更新，Reader就有可能会因为判断索引已更新而读到脏数据。

那禁止编译器对该类变量的优化，解决了编译期的重排序就没事了吗？不，CPU还有**乱序执行（Out-of-Order Execution）**的特性。流水线（Pipeline）和乱序执行是现代CPU基本都具有的特性。机器指令在流水线中经历取指、译码、执行、访存、写回等操作。为了CPU的执行效率，流水线都是并行处理的，在不影响语义的情况下。**处理器次序（Process Ordering，机器指令在CPU实际执行时的顺序）**和**程序次序（Program Ordering，程序代码的逻辑执行顺序）**是允许不一致的，即满足**As-if-Serial**特性。显然，这里的不影响语义依旧只能是保证指令间的**显式因果关系**，无法保证**隐式因果关系**。即无法保证语义上不相关但是在程序逻辑上相关的操作序列按序执行。从此单核时代CPU的**Self-Consistent**特性在多核时代已不存在，多核CPU作为一个整体看，不再满足**Self-Consistent**特性。

简单总结一下，如果不做多余的防护措施，单核时代的无锁环形队列在多核CPU中，一个CPU核心上的Writer写入数据，更新index后。另一个CPU核心上的Reader依靠这个index来判断数据是否写入的方式不一定可靠。index有可能先于数据被写入，从而导致Reader读到脏数据。

所有的麻烦到这里就结束了吗？当然不，还有Cache的问题。前文提到的都是**顺序一致性（Sequential Consistency）**的问题，没有涉及**Cache一致性（Cache Coherence）**的问题。虽然说一般情况下程序员只需要关注顺序一致性即可，但是区分清楚这两个概念也能更好的解释**内存屏障（Memory Barrier）**。

开始提到Cache一致性协议之前，先介绍两个名词：

- Load/Read CPU读操作，是指将内存数据加载到寄存器的过程
- Store/Write CPU写操作，是指将寄存器数据写回主存的过程

现代处理器的缓存一般分为三级，由每一个核心独享的L1、L2 Cache，以及所有的核心共享L3 Cache组成：

![cache1](http://www.0xffffff.org/images/40/cpu_cache.png)

既然各个核心之间有独立的Cache存储器，那么这些存储器之间的数据同步就是个比较复杂的事情。缓存数据的一致性由缓存一致性协议保证。这里比较经典的当属[MESI协议](https://en.wikipedia.org/wiki/MESI_protocol)。Intel的处理器使用从MESI中演化出的[MESIF协议](http://www.realworldtech.com/common-system-interface/5/)，而AMD使用[MOESI协议](http://en.wikipedia.org/wiki/MOESI_protocol)。缓存一致性协议的细节超出了本文的讨论范围，有兴趣的读者可以自行研究。

传统的MESI协议中有两个行为的执行成本比较大。一个是将某个Cache Line标记为Invalid状态，另一个是当某Cache Line当前状态为Invalid时写入新的数据。所以CPU通过Store Buffer和Invalidate Queue组件来降低这类操作的延时。如图：

![cache2](http://www.0xffffff.org/images/40/cache_sync.png)

当一个核心在Invalid状态进行写入时，首先会给其它CPU核发送Invalid消息，然后把当前写入的数据写入到Store Buffer中。然后异步在某个时刻真正的写入到Cache Line中。当前CPU核如果要读Cache Line中的数据，需要先扫描Store Buffer之后再读取Cache Line（Store-Buffer Forwarding）。但是此时其它CPU核是看不到当前核的Store Buffer中的数据的，要等到Store Buffer中的数据被刷到了Cache Line之后才会触发失效操作。而当一个CPU核收到Invalid消息时，会把消息写入自身的Invalidate Queue中，随后异步将其设为Invalid状态。和Store Buffer不同的是，当前CPU核心使用Cache时并不扫描Invalidate Queue部分，所以可能会有极短时间的脏读问题。当然这里的Store Buffer和Invalidate Queue的说法是针对一般的SMP架构来说的，不涉及具体架构。事实上除了Store Buffer和Load Buffer，流水线为了实现并行处理，还有Line Fill Buffer/Write Combining Buffer 等组件，参考文献8-10给出了相关的资料可以进一步阅读。

好了，问题背景描述的差不多了，下面该解决方案登场了。

**编译器优化乱序**和**CPU执行乱序**的问题可以分别使用**优化屏障 (Optimization Barrier)**和**内存屏障 (Memory Barrier)**这两个机制来解决：

**优化屏障 (Optimization Barrier)**：避免编译器的重排序优化操作，保证编译程序时在优化屏障之前的指令不会在优化屏障之后执行。这就保证了编译时期的优化不会影响到实际代码逻辑顺序。

IA-32/AMD64架构上，在Linux下常用的GCC编译器上，优化屏障定义为（linux kernel, include/linux/compiler-gcc.h）：

```c
/* The "volatile" is due to gcc bugs */
#define barrier() __asm__ __volatile__("": : :"memory")
```

优化屏障告知编译器：

1. 内存信息已经修改，屏障后的寄存器的值必须从内存中重新获取
2. 必须按照代码顺序产生汇编代码，不得越过屏障

> C/C++的volatile关键字也能起到优化限制的作用，但是和Java中的volatile（Java 5之后）不同，C/C++中的volatile不提供任何防止乱序的功能，也并不保证访存的原子性。

**内存屏障 (Memory Barrier)**分为**写屏障（Store Barrier）**、**读屏障（Load Barrier）**和**全屏障（Full Barrier）**，其作用有两个：

1. 防止指令之间的重排序
2. 保证数据的可见性

关于第一点，关于指令重排，这里不考虑架构的话，Load和Store两种操作会有Load-Store、Store-Load、Load-Load、Store-Store这四种可能的乱序结果。 上文提到的三种屏障则是限制这些不同乱序的机制。

关于第二点。写屏障会阻塞直到把Store Buffer中的数据刷到Cache中；读屏障会阻塞直到Invalid Queue中的消息执行完毕。以此来保证核间各级数据的一致性。

这里要强调，内存屏障解决的只是顺序一致性的问题，不解决Cache一致性的问题（这是Cache一致性协议的责任，也不需要程序员关注）。Store Buffer和Load Buffer等组件是属于流水线的一部分，和Cache无关。这里一定要区分清楚这两点，Cache一致性协议只是保证了**Cache一致性（Cache Coherence）**，但是不关注**顺序一致性（Sequential Consistency）**的问题。比如，一个处理器对某变量A的写入操作仅比另一个处理器对A的读取操作提前很短的一点时间，那就不一定能确保该读取操作会返回新写入的值。这个**新写入的值多久之后能确保被读取操作读取到，这是内存一致性模型（Memory Consistency Models）要讨论的问题**。

完全的确保顺序一致性需要很大的代价，不仅限制编译器的优化，也限制了CPU的执行效率。为了更好地挖掘硬件的并行能力，现代的CPU多半都是介于两者之间，即所谓的**宽松的内存一致性模型（Relaxed Memory Consistency Models）**。不同的架构在重排上有各自的尺度，在严格排序和自由排序之间会有各自的偏向。偏向严格排序的一边，称之为**强模型（Strong Model）**，而偏向于自由排序的一边，称之为**弱模型（Weak Model）**。AMD64架构是強模型：

![memory1](http://www.0xffffff.org/images/40/memory_consistency_models.png)

特别地，早先时候，AMD64架构也会有Load-Load乱序发生（Memory Ordering in Modern Microprocessors, PaulE.McKenney, 2006）。

![memory2](http://www.0xffffff.org/images/40/memory_consistency_models_old.png)

**注意这里的IA-64（Intanium Processor Family）是弱模型，它和Intel® 64不是一回事。后者是从AMD交叉授权来的，源头就是AMD64架构。这里不讨论历史，只需要知道平时说的x86-64/x64就是指的AMD64架构即可。**

《Intel® 64 and IA-32 Architectures Software Developer’s Manual》有如下的阐述：

![memory3](http://www.0xffffff.org/images/40/memory_ordering.png)

- 读操作之间不能重新排序
- 写操作不能跟旧的读操作排序
- 主存写操作不能跟其他的写操作排序，但是以下情况除外：
  - 带有CLFLUSH（失效缓存）指令的写操作
  - 带有non-temporal move指令的流存储（写入）（MOVNTI, MOVNTQ, MOVNTDQ, MOVNTPS, 和 MOVNTPD，都是SSE/SSE2扩展的指令）
  - 字符串操作（REP STOSD等）
- 不同内存地址的读可以与较早的写排序，同一地址的情况除外
- 对I/O指令、锁指令、序列化指令的读写不能重排序
- 读不能越过较早的读屏障指令（LFENCE）或者全屏障指令（MFENCE）
- 写不能越过较早的读屏障指令（LFENCE）、写屏障指令（SFENCE）和全屏障指令（MFENCE）
- 读屏障指令（LFENCE）不能越过较早的读
- 写屏障指令（SFENCE）不能越过较早的写
- 全屏障指令（MFENCE）不能越过较早的读和写
- 在多处理器的情况下，单处理器内部的内存访问排序仍然依照以上的原则，并且规定处理器与处理器之间遵循如下的原则：

  - 某个处理器的全部写操作以同样的顺序被其它处理器观察到
  - 不同处理器之间的写操作不重排序
  - 排序遵循逻辑上的因果关系
  - 第三方总是观察到一致的写操作顺序

那么上文提到的四种可能的乱序在AMD64下明确说明不会有Load-Load乱序、Load-Store乱序，**明确会出现Store-Load乱序**，Store-Store乱序除了几种例外的情况也不会出现。参考文献5中给出了在Linux下重现出Store-Load乱序的代码，有兴趣的读者可以自行测试。

但是内存一致性模型不仅仅是没有指令重排就会保证一致的。但是如果仅仅只考虑指令重排，完全按照该规则来思考，就会遇到违反直觉的事情。特别的，在对写缓存的同步处理上，AMD64内存访问模型的 **Intra-Processor Forwarding Is Allowed**这个特性比较要命：

![memory4](http://www.0xffffff.org/images/40/intra-processor_forwarding_is_allowed.png)

只考虑指令重排的话，AMD64架构既然不会有Load-Load重排的，r2=r4=0就不可能会出现，但是实际的结果是违反直觉的。出现这个现象的原因就是Intel对Store Buffer的处理上，Store Buffer的修改对其他CPU核心是不可见的。Processor 0对_x的修改缓存在了Processor 0的Store Buffer中，还未提交到L1 Cache，自然也不会失效掉Processor 1的L1 Cache中的相关行。Processor 1对_y的修改同理。

对于以上问题，AMD64提供了三个内存屏障指令来解决：

![memory5](http://www.0xffffff.org/images/40/memory_barrier.png)

sfence指令为写屏障（Store Barrier），作用是：

1. 保证了sfence前后Store指令的顺序，防止Store重排序
2. 通过刷新Store Buffer保证sfence之前的Store要指令对全局可见

lfence指令读屏障（Load Barrier），作用是：

1. 保证了lfence前后的Load指令的顺序，防止Load重排序
2. 刷新Load Buffer

mfence指令全屏障（Full Barrier），作用是：

1. 保证了mfence前后的Store和Load指令的顺序，防止Store和Load重排序
2. 保证了mfence之后的Store指令全局可见之前，mfence之前的Store指令要先全局可见

如前文所说，AMD64架构上是不存在Load-Load重排的，但是当一个CPU核心收到其他CPU核心失效Cache Line的消息后，立即回复给对方一个应答信号。但是此时并没有立即失效掉Cache Line，而是将其包装成一个结构投递到自身的Load Buffer里。AMD64架构上不存在Load-Load重排并不意味着流水线真的就一条一条执行Load指令。在保证两个CPU核看到的Store顺序一致的情况下，是允许Load乱序的。比如连续的两个访存指令，指令1 Cache Miss，指令2 Cache Hit，实际上指令2是不会真的等待指令1的Load完成整个Cache替换过程后才执行的。实际流水线的实现中，Load先是乱序执行，然后有一个Load-ordering-Buffer（Load Buffer）的结构，在Load Commit之前检测冲突，Load过的地址是否又被其他CPU核心写过（没有存在失效信息）。只要没有冲突，这种乱序就是安全的。如果发生冲突，这种乱序就违反x86要求，需要被取消并Flush流水线。而上文提到的lfence指令会刷新Load Buffer，保证当前CPU核心立即读取到最新的数据。

另外， 除了显式的内存屏障指令，有些指令也会造成指令保序的效果，比如I/O操作的指令、exch等原子交换的指令，任何带有lock前缀的指令以及CPUID等指令都有内存屏障的作用。

下面说说锁和原子变量。对于数据竞争（Data Races）的情况，最简单和最常见的场景就是使用Mutex了，包括并不限于互斥锁、自旋锁、读写锁等。拿互斥锁来说，除了保护临界区只能有一个执行流之外，还有其他的作用。这里要引入**宽松的内存一致性模型（Relaxed Memory Consistency Models）**中的**Release Consistency模型**来解释，这个模型包含了同步操作**Acquire**和**Release**：

1. Acquire: 在此操作后的所有读写操作必然发生在Acquire这个动作之后
2. Release: 在此操作前的所有读写操作必然发生在Release这个动作之前

要注意的是Acquire和Release都只保证了一半的顺序：

- 对于Acquire来说，并没保证Acquire前的读写操作不会发生在Acquire动作之后
- 对于Release来说，并没保证Release后的读写操作不会发生在Release动作之前

因此Acquire和Release的组合便形成了内存屏障。

Mutex的Lock操作暗含了Acquire语义，Unlock暗含了Release语义。**这里是脱离架构在讨论的，在具体的平台上如果Load和Store操作暗含Acquire和Release语义的话自然保证一致，否则可以是相关的内存屏障指令**。所以Mutex不仅会保证执行的序列化，同时也保证了访存的一致性。与之类似，**平台提供的原子变量除了保证内存操作原子之外，也会保证访存的一致性**。



## Definition

顺序一致性模型(sequential consistency)，简称SC，毫无疑问是最直观的内存模型。Lamport在1979年的论文《How to Make a Multiprocessor Computer that Correctly Executes Multiprocess Programs》提出了SC。Lamport首先定义了单核单处理器的顺序一致性内存模型：程序的执行结果和按照程序代码顺序执行后的结果相同(译者注：允许指令重排序，只要程序运行结果和指令不排序的结果相同，就满足顺序一致性内存模型)。接着，Lamport又给出了多处理器环境下的顺序一致性内存模型：整个多线程程序的执行就好像是把各个线程的指令交错到一个单线程内执行一样，同时，交错指令执行时不能对原来单线程内的指令执行顺序进行重排序。

多线程程序的所有指令的执行顺序被称为memory order。**顺序一致性内存模型的Memory order不会对各个单线程的程序代码执行顺序进行指令重排序**。但其他内存模型就未必不进行指令重排序了。

考虑这个在双核情境下执行的例子。
> 这里的”核“，指的是从软件视角看的核，可能是一个实际的物理核或者是在多线程下的线程上下文。

![table3-1](screenshots/table3-1.jpg)

表3-1

![fig3-1](screenshots/fig3-1.jpg)

图3-1

图3.1演示了表3.1代码的可能的执行顺序。中间最粗的箭头就是多线程程序所有指令的执行顺序memory order(&lt;m)，两侧是每个线程内指令的执行顺序program order(&lt;p)。我们用操作符&lt;m表示memory order，op1 &lt;m op2表示指令op1在memory order中先于指令op2执行。同样地，我们用&lt;p表示一个线程的程序顺序(program order)，op1 &lt;p op2表示在这个线程中，指令op1先于op2执行。SC中memory order遵守每个线程的程序顺序，即，如果op1 &lt;p op2，则op1 &lt;m op2。



## SC Formalism

An SC execution requires:

(1) All cores insert their loads and stores into the order < m respecting their program order, regardless of whether they are to the same or different addresses(i.e., a=b or a≠b). There are four cases:

(1) 每个核(线程)在把它自己所属的L、S操作插入到memory order时，都要严格遵守自己线程内的指令排序。总共有四种情况（下面的a和b可以相等，也可以不相等）:


- b
- a


## preshing BLOG

### Introduction to lock-free

![lock-free1](https://preshing.com/images/its-lock-free.png)

lock-free中的lock不直接指mutex，更多的是指不“locking up"整个应用程序的某种可能性，可能是死锁，活锁，甚至是某种由假设的线程调度决策导致的结果。

对于一个并发实现，无论当前处于什么状态，只要运行足够长的时间，至少有一个 process 能取得进展或完成其操作，则其实现称之为 lock-free（这里的 process 代表并发操作中一条独立的逻辑流，可以是线程，也可以是进程）。

考虑如下例子：
```cpp
// C++
std::stack<T> stack;
std::mutex mutex;

void push(const T& obj) {
   mutex.lock();
   stack.push(obj);
   mutex.unlock();
}
```

假设在一个单核实时可抢占的系统中，有三个线程 A / B / C，其中 A / B 两个线程同时会访问到 push 方法，其中 A 具备最高优先级，B 优先级次之。考虑如下并发顺序

- 线程 C 进入 push 内部，获取到 mutex
- 线程 A 唤醒，并中断了 C，进入 push 内部，但是因为 锁 已经被 C 占用，所以 A 进入阻塞队列
- 线程 B 唤醒，因其比 C 的优先级高，并且其不执行 push 而是执行其他任务，所以直到 B 完成或因为其他原因被阻塞之前，C 没有机会继续执行并释放锁，A 继续等待
- 也就是说，优先级更低的 B 比 优先级更好的 A 更先执行，并且只有 B 在被阻塞的情况 A 才有机会继续

这个现象被称为优先级倒置，其本质原因是因为 push 操作不是 lock free 的，一个线程的挂起可以将整体的 push 操作全部阻塞住了。在一些关键领域，这种现象是不可接受的，有很多方法可以解决这个问题，比如当 A 发现锁被抢占时，将当前获得锁的线程 C 优先级提高到和 A 一样，这样 B 就无法抢占 C。但是根据 lock-free 的定义，无论处于什么状态，都不应该让所有的线程全部被阻塞，而在上述实现中如果线程 C 进入某种一直不会执行的状态，它将阻塞其他所有同一操作的线程。

在这个基础上，很容易理解所有基于『锁』的并发实现，都不是 lock-free 的，因为它们都会遇到同样的问题 —— 如果我们永久暂停当前占有锁的某一个线程 / 进程的执行，将会阻塞其他线程 / 进程的执行。而对于 lock-free 实现，允许部分 process 饿死但保证整体逻辑的持续前进。

这里在看另外一个反例，在这个示例中两个线程 A / B 将并发执行下列代码

```cpp
// initialize x = 0
while (x == 0) {
   x = 1 - x;
}
```

如果考虑以下执行顺序

- A 进入 while 循环，当前 x = 0
- B 进入 while 循环，当前 x = 0
- A 执行 x = 1 - x, 当前 x = 1
- B 执行 x = 1 - x，当前 x = 0
- 循环继续
这是一个无锁操作，但是也会导致整体逻辑无法向下推进，其同样不是 lock-free 的。

通过上面两个例子，大概能体会到 lock-free 最大的要求是其实现不会锁定整体逻辑，无论是死锁，还是活锁。如果有一个并发实现是 lock-free 的，那么我们就可以不用额外的改进将其使用在一个实时操作系统上，而无须担心调度系统长时间阻塞高优先级任务。那么如何实现一个 lock-free 的操作以及需要注意哪些问题，在此之前，我们将会假定一个条件来限定问题的边界。


> “In an infinite execution, infinitely often some method call finishes.” 

一个lock-free programming的重要后果是，如果你挂起某单个线程，该行为不会阻止其他线程取得进展（making progress）（反例可见上面的两个例子，其中例1中一旦暂停了线程C的执行，它将阻塞其他同一操作的线程）。

当我们准备要满足 Lock-Free 编程中的非阻塞条件时，有一系列的技术和方法可供使用，如原子操作（Atomic Operations）、内存栅栏（Memory Barrier）、避免 ABA 问题（Avoiding ABA Problem）等。那么我们该如何抉择在何时使用哪种技术呢？可以根据下图中的引导来判断。

![lock-free2](https://preshing.com/images/techniques.png)

#### 原子的Read-Modify-Write操作

没有线程可以观测到一个半完成（half-complete）的原子操作。

Read-modify-write(RMW)操作“go a step further”，允许你去原子性地执行更复杂的transaction。它们对实现一个必须支持多个writer的lock-free算法特别有效，当多个线程试图去RMW到同一个内存地址时，它们会有效地排成一列并one-at-a-time地执行这些操作。

C++11中一个简单的RMW操作例子```std::atomic<int>::fetch_add```。注意C++11的标准不保证实现在任何平台上lock-free（标准只要求atomic_flag的CAS实现必须lock-free）。

#### CAS Loop

通常被提起最多的RMW操作是compare-and-swap（CAS）。通常，程序员在一个循环中执行CAS去重复地试图完成一个事务。模式通常包括：拷贝一个共享变量到local，执行某些特定的工作，然后尝试利用CAS发布（publish）这个改动。

```cpp
int CAS(int *ptr,int newvalue,int oldvalue)
{
   // initial value of the destination location
   int temp = *ptr;
   if(*ptr == oldvalue)
       *ptr = newvalue;
   return temp;
}


LONG InterlockedCompareExchange(
  LONG volatile *Destination,
  LONG          ExChange,
  LONG          Comperand
);


// Parameters
//
// Destination
// A pointer to the destination value.
// 
// ExChange
// The exchange value.
// 
// Comperand
// The value to compare to Destination.
// 
// Return Value
// The function returns the initial value of the Destination parameter.

// The function compares the Destination value with the Comparand value. If the Destination value is equal to the Comparand value, the Exchange value is stored in the address specified by Destination. Otherwise, no operation is performed.


void LockFreeQueue::push(Node* newHead)
{
    for (;;)
    {
        // Copy a shared variable (m_Head) to a local.
        Node* oldHead = m_Head;

        // Do some speculative work, not yet visible to other threads.
        newHead->next = oldHead;

        // Set the new value only if the memory location is still the original value.
        // Next, attempt to publish our changes to the shared variable.
        // If the shared variable hasn't changed, the CAS succeeds and we return.
        // Otherwise, repeat.
        if (_InterlockedCompareExchange(&m_Head, newHead, oldHead) == oldHead)
            return;
    }
}
```

如果CAS的测试fail for one thread，这意味着肯定有别的线程CAS测试成功（有别的线程修改了共享变量） - 尽管对于某些提供了weaker variant of CAS的平台架构，这不总成立。而当前线程可以在下一个循环周期内继续判断以完成操作（下一个循环中，当前线程会重新copy修改后的共享变量m_Head到local。


#### ABA 问题

存在多个线程交错地对共享的内存地址进行处理时。

若线程对同一内存地址进行了两次读操作，而两次读操作得到了相同的值，通过判断 "值相同" 来判定 "值没变"。然而，在这两次读操作的时间间隔之内，另外的线程可能已经修改了该值，这样就相当于欺骗了前面的线程，使其认为 "值没变"，实际上值已经被篡改了。

考虑如下的操作序列：

1. T1线程从共享的内存地址读取值A
2. T1线程被抢占，线程T2开始运行
3. T2线程将共享的内存地址中的值由A修改成B，然后又修改回A
4. T1线程继续执行，读取共享的内存地址中的值仍为A，认为没有改变然后继续执行。

由于 T1 并不知道在两次读取的值 A 已经被 "隐性" 的修改过，所以可能产生无法预期的结果。

例如，使用 List 来存放 Item，如果将一个 Item 从 List 中移除并释放了其内存地址，然后重新创建一个新的 Item，并将其添加至 List 中，由于优化等因素，有可能新创建的 Item 的内存地址与前面删除的 Item 的内存地址是相同的，导致指向新的 Item 的指针因此也等同于指向旧的 Item 的指针，这将引发 ABA 问题。

![aba-1](https://images0.cnblogs.com/blog/175043/201410/231100493083369.jpg)

举个例子，假设有一个用单链表实现的栈，```A --> B --> C```，有head指针指向栈顶的A，用CAS操作，实现如下的push和pop
```
CAS(*dest, oldvalue, newvalue)    // return initial value of *dest

push(node):
    curr = head    // 将head拷贝到local
    old = curr     // old表示与共享的head比较的期值
    node->next = curr     // CAS(&head, curr, node)，将curr与head的initial value比较
    while(old != (curr = CAS(&head, curr, node))){
        // *head != curr(curr == old)，表示有新的节点被push入
        // 此时由CAS返回的curr即当前的stack head
        old = curr
        node->next = curr
    }

pop():
    curr = head
    old = curr
    next = curr->next     // CAS(&head, curr, next)，将curr与head的initial value比较
    while(old != (curr = CAS(&head, curr, next))){
        // *head != curr(curr == old)，表示有另外的节点被pop
        // 此时由CAS返回的curr即当前的stack head
        old = curr     // 下一次比较中新的stack head期值
        next = curr->next    // 拟作为的新的stack head
    }
    return curr
```

ABA的问题在于，pop函数中，next=curr->next和while之间，线程被切换，然后其他线程先把A弹出，又把B弹出，然后又把A压入（这里指内存地址相同，即为新压入的A节点分配的内存地址与被回收的A节点相同），栈变成了```A-->C```，此时head还是指向A，等pop被切换回来继续执行，认为栈顶没有改变，即其认为栈的状态仍为```A-->B-->C```把head指向B，而B已经不存在，此时正确的栈状态应为```C```。

对于上面的例子，就是A被弹出后，需要保证它的内存不能立即释放（因为还有线程引用它），也就不能立即被重用。因此在存在GC的系统里，不会有上面的例子出现，它保证新分配的节点不可能与被弹出但仍在pop线程中保留对其引用的A节点内存地址相同。

举一个更具象的例子：

- 小明在提款机，提取了50元，因为提款机问题，有两个线程，同时把余额从100变为50
- 线程1（提款机）：获取当前值100，期望更新为50，
- 线程2（提款机）：获取当前值100，期望更新为50，
- 线程1成功执行，线程2某种原因block了，这时，某人给小明汇款50
- 线程3（默认）：获取当前值50，期望更新为100，
- 这时候线程3成功执行，余额变为100，
- 线程2从Block中恢复，获取到的也是100，compare之后，继续更新余额为50！！！
- 此时可以看到，实际余额应该为100（100-50+50），但是实际上变为了50（100-50+50-50）这就是ABA问题带来的成功提交。

解决方法：
在变量前面加上版本号，每次变量更新的时候变量的版本号都+1，即```A->B->A```就变成了```1A->2B->3A```。


### Memory Ordering at Compile Time

Changes to memory ordering are made both by 编译器（在编译时）和 by CPU（在执行指令时）。

内存重排序的基本准则，无论是对编译器还是CPU侧而言：
> 不能修改单线程程序的行为。

作为这条规则的结果，编写单线程代码的程序员不会注意到内存重排序。在很大程度上对于多线程编程也是如此，由于像mutex，信号量，事件这样的机制/工具被设计出来在他们的调用侧周围提供对内存重排序的阻止。而在lock-free的世界里，内存在不被施加任何互斥的情况下被多个线程所共享，内存重排序的effect从而会被观测到。

考虑一个简单的例子，一个共享的标志（flag）被使用来指示某些其他的共享数据已经被发布（publish）。

```cpp
int Value;
int IsPublished = 0;
 
void sendValue(int x)
{
    Value = x;
    IsPublished = 1;
}
```

假设编译器将对```IsPublished```的写提前到对```Value```的写之前，这并不违反“单线程。。。”的准则。那么其他线程通过标志位认为```Value```已经被更新的时刻，实际上```Value```并未被更新。

在C++11中，任何的non-relaxed原子操作都可以充当一个compiler barrier。
```cpp
int Value;
std::atomic<int> IsPublished(0);

void sendValue(int x)
{
    Value = x;
    // <-- reordering is prevented here!
    IsPublished.store(1, std::memory_order_release);
}
```

任何包含了compiler barrier的函数本身充当一个compiler barrier，就算这个函数是内联的。
```cpp
void doSomeStuff(Foo* foo)
{
    foo->bar = 5;
    sendValue(123);       // prevents reordering of neighboring assignments
    foo->bar2 = foo->bar;
}
```

一个对external linkage函数的调用具有更强的memory barrier的保证，原因是编译器不知道这个函数可能的副作用。它必须不对这个潜在的内存副作用做任何假设（这需要过程间分析）。

### Memory Barriers Are Like Source Control Operations

不同于compiler reordering，processor reordering的effect只在多处理器系统下可见。

考虑如图所示的操作序列，X和Y都为全局变量，初值均为0:
![barrier1](https://preshing.com/images/marked-example2-2.png)

Processor1和Processor2都可能得到r1 = 0 and r2 = 0的结果。即P1/P2向X/Y中写的结果在从彼此的local cache读之后才可见。

四种memory barrier，四种fense instruction。

![barrier2](https://preshing.com/images/barrier-types.png)

```LoadLoad```阻止在barrier之前执行的load和barrier之后执行的load的重排序。

```cpp
if (IsPublished)                   // Load and check shared flag
{
    LOADLOAD_FENCE();              // Prevent reordering of loads
    return Value;                  // Load published value
}
```

It doesn’t matter exactly when 对```IsPublished```标志的写发布 happens（leak）; once the leaked flag has been observed, issues a ```#LoadLoad``` fence to prevent reading some value of Value which is older than the flag itself。

```StoreStore```与```LoadLoad```类似，读换成写。

```LoadStore```，假设Larry有一张他要执行的指令列表，有的指令需要让他从自己的cache中取出数据到寄存器，有的需要让他从自己的寄存器中写回数据到cache。当larry遇见一条读指令时，他可以lookahead这条读指令之后的任意写指令，如果这些写指令与这条读指令完全没有关系（completely unrelated)，他可以先执行这些读指令，然后回来完成那条读指令。

在真实环境下，出现这样的指令重排序可能发生在：一个发生了cache miss的读操作后面紧跟着一条cache hit的写操作。而```LoadStore```barrier simply refrains（节制） from such reordering around that barrier.

```StoreLoad```barrier ensures that all stores performed before the barrier are visible to other processors, and that all loads performed after the barrier receive the latest value that is visible at the time of the barrier.

### Acquire and Release Semantic

总的来说，在lock-free programming的世界里，有两种办法可以使多个线程操纵共享内存：彼此竞争某个资源，或是从某个线程传递信息到另一个。Acquire和Release语义对后者而言非常重要：可靠的线程间消息传递。

一种定义：

Acquire semantic：是一种性质，其只能被应用于共享内存上的读操作，无论这些操作是read-modify-write还是单纯的读。该操作就被认为是一个read-acquire。acquire semantic阻止其程序顺序（program order）之后的任意内存读/写操作被重排序到其之前（在此操作后的所有读写操作必然发生在Acquire这个动作之后）。

![read-acquire](https://preshing.com/images/read-acquire.png)

Release semantic: 是一种性质，其只能被应用于共享内存上的写操作，无论这些操作是read-modify-write还是单纯的写。该操作就被认为是一个write-release。release semantic阻止其程序顺序（program order）之前的任意内存读/写操作被重排序到其之前（在此操作前的所有读写操作必然发生在Release这个动作之前）。

![write-release](https://preshing.com/images/write-release.png)

barrier总得以某种方式被放置在read-acquire操作之后，在write-release操作之前。

![relation-with-barrier1](https://preshing.com/images/acq-rel-barriers.png)

![relation-with-barrier2](https://preshing.com/images/platform-fences.png)

release要求在此操作之前的所有读写操作必然发生在release这个动作之前。通过插入```LoadStore```和```StoreStore```。

acquire要求在此操作之后的所有读写操作必然发生在acquire这个动作之后。通过插入```LoadStore```和```LoadLoad```。

这个例子里，我们可以说对```Ready```的store synchronized-with load。

### 引入：happens-before

A和B表示多线程下执行的一系列操作。如果A happens-before B，那么A的memory effect在B执行前就要求对B可见。

### 引入：synchronizes-with

synchronizes-with -> happens-before

in §29.3.2 of working draft N3337:

> An atomic operation A that performs a release operation on an atomic object M synchronizes with an atomic operation B that performs an acquire operation on M and takes its value from any side effect in the release sequence headed by A.

![sync1](https://preshing.com/images/two-cones.png)

就像synchronizes-with不是建构happens-before关系的唯一办法，一对write-release/read-acquire操作也不是建构synchronizes-with关系的唯一办法，见下图。

![sync2](http://preshing.com/images/org-chart.png)


#### A Write-Release Can Synchronize-With a Read-Acquire

假设我们有一个Message结构，由一个线程生产而由另外的线程消费，其定义如下：
```cpp
struct Message
{
    clock_t     tick;
    const char* str;
    void*       param;
};
```

我们在线程间传递```Message```的实例，通过将其放置在一个共享的全局变量中。我们成这个共享变量为载荷（payload）。

```cpp
Message g_payload;
```

没有一个可移植的方法是我们能够在一个原子操作内填充```g_payload```，所以我们不会尝试这样做。作为替代，我们定义一个分离的原子变量```g_guard```，来标示```g_payload```是否已经就绪。```g_guard```作为我们的哨卫变量（guard variable）。这个哨卫变量必须通过原子操作来操纵，原因是多个线程可能并发地对其访问，且其中某一个线程会执行写操作。

```cpp
std::atomic<int> g_guard(0);
```

为了在线程间安全的传递```g_payload```，我们使用acquire and release semantics。

```cpp
void SendTestMessage(void* param)
{
    // Copy to shared memory using non-atomic stores.
    g_payload.tick  = clock();
    g_payload.str   = "TestMessage";
    g_payload.param = param;
    
    // Perform an atomic write-release to indicate that the message is ready.
    g_guard.store(1, std::memory_order_release);
}

bool TryReceiveMessage(Message& result)
{
    // Perform an atomic read-acquire to check whether the message is ready.
    int ready = g_guard.load(std::memory_order_acquire);
    
    if (ready != 0)
    {
        // Yes. Copy from shared memory using non-atomic loads.
        result.tick  = g_payload.tick;
        result.str   = g_payload.str;
        result.param = g_payload.param;
        
        return true;
    }
    
    // No.
    return false;
}
```

### 引入：Acquire and Release Fences

---

线程间的同步和内存顺序决定表达式的求值和side effects在程序执行的不同线程间如何排序。它们以下列项目定义：

## CPP

### Sequenced-before

在同一线程中，evaluation A可以sequenced-before evaluation B，如求值顺序中所描述。

求值任何表达式的任何部分，包括求值函数参数的顺序都是未说明的（除了下列的一些例外）。编译器能以任何顺序求值任何操作数和其他子表达式，并且可以在再次求值同一表达式时选择另一顺序。

C++ 中无从左到右或从右到左求值的概念。这不会与运算符的从左到右及从右到左结合性混淆：表达式 a() + b() + c() 由于 operator+ 的从左到右结合性被分析成 (a() + b()) + c() ，但在运行时对c的函数调用可能发生在a()之前，b()之后或者a()，b()之间。

### Sequenced-before rules

#### 定义

##### 表达式求值

  - identity，不可移动：lvalue
  - identity, 可移动：xvalue
  - 无identity,可移动：prvalue
  - identity: glvalue
  - 可移动：rvalue

每个表达式的求值包括：

- value computation: 计算表达式所返回的值。这可能涉及对object identity的确认（glvalue的evaluation，例如一个返回到某对象的引用的表达式），或读取先前赋给object的值（prvalue的求值，例如当表达式返回数或某个其他值时）

- 引发side effect：访问（读或写）volatile glvalue所指代的对象，修改（写入）对象，调用库 I/O 函数，或调用任何做出这些操作的函数。

##### 顺序

sequenced-before是同一线程中的evaluation之间非对称的(asymmetric)，传递的对偶关系。

- 若A sequenced-before B，evaluation A会在evaluation B开始前完成
- 若A不sequenced-before B，而B sequenced-before A，则evaluation B会在evaluation A开始前完成
- 若A不sequenced-before B且B不sequenced-before A，则存在两种可能：
  - A 与 B 的求值是无顺序 (unsequenced) 的：它们能以任何顺序进行，并可能重叠（在同一执行线程内，编译器可以将组成 A 与 B 的 CPU 指令交错）
  - A 与 B 的求值是顺序不确定 (indeterminately sequenced) 的：它们可以任意顺序进行但不可重叠，A 在 B 前完成，或 B 在 A 前完成。下次求值相同表达式时顺序可以相反。


##### 规则

完整的规则参见https://en.cppreference.com/w/cpp/language/eval_order

上一行的求值full expression（包含value computations和side effect），会先于下一行的求值。

然而必须注意到，实际的求值顺序并不同于sequenced-before，甚至不同于happens-before，只要求as if happens-before（见下面的“个人”及上面的“Cache Coherence"）

### Carries dependency

在同一线程中，evaluation A sequenced-before evaluation B可能carry a dependency into B(即， B 依赖于 A)，如果下列条件中任一成立：

1. evaluation A的值被用作evaluation B的operand，除非
  - evaluation B 是对```std::kill_dependency```的调用
  - evaluation A 是内建```&&```, ```||```, ```?:```或```,```运算符的左操作数
2. evaluation A写入标量对象M，B从M读取
3. A carries dependency到evaluation X, X carries dependency到B

### Modification order

所有对任一特定的原子变量的修改，以限定于此原子变量的单独全序出现（出现在一个特定于该原子变量的全序中）（All modifications to any particular atomic variable occur in a total order that is specific to this one atomic variable.）

对所有原子操作保证下列四个要求：

1. **Write-write coherence**: 如果对某个原子变量M的修改evaluation A（写）happens-before另一个修改了M的evaluation B，那么A在M的修改顺序（modification order)中先于B出现
2. **Read-read coherence**: 如果一次依赖于某个原子变量的computation A(一次读) happens-before另一个在M上的computation B(读)，且如果A读取到的M值来源于一次对M的写X，那么B读到的M的值要么是由X写入的值，要么是在 M 的修改顺序中后出现于 X 的 M 上side effect Y 所存储的值
3. **Read-write coherence**: 如果一次依赖于某个原子变量的computation A(一次读) happens-before另一个在M上的operation B(一次写)，那么A读取到的值来源于M的修改顺序中早出现于B的side effect X（写）
4. **Write-read coherence**: 如果对某个原子变量M的修改side effect X(写)happens-before一次对M的value computation B(读)，那么evaluation B得到的值要么来源于X，要么是在M的修改顺序中晚于X出现的side effect Y

### Release sequence

在原子对象M上进行的释放操作A之后，由下列内容组成的M修改顺序的最长相接子序列：

1. 与perform A的同一线程所perform的writes（C++20前）
2. 任何线程对M的原子的read-motify-write操作

被称作headed by A的release sequence。

### Dependency-ordered before（依赖先序于）

在线程间，若下列条件中任一成立，则称evaluation A dependency-ordered before evaluation B。

1. A在原子变量M上perform了一次release operation，且在另一个不同线程内，B perform了一次comsume operation于同一个原子变量M上，且B读取headed by A的release sequence中的任意部分所写入的值
2. A dependency-ordered before X且X carries a dependency into B

### Inter-thread happens-before（线程间happens-before)

在线程间，若下列条件中任一成立，则evaluation A线程间happens-before evaluation B

1. A synchronizes-with B
2. A dependency-ordered before B
3. A synchronizes-with某个evaluation X，且X sequenced-before B
4. A sequenced-before某个evaluation X，且X线程间happens-before B
5. A线程间happens-before某个evaluation X，且X线程间happens-before B

### Happens-before

无关乎线程，若下列条件中任一成立，则evaluation A happens-before evaluation B：

1. A sequenced-before B
2. A 线程间happens-before B

要求实现确保先发生于关系是非循环的，若有必要则引入额外的同步（若引入消费操作，它才可能为必要）。

若某一evaluation修改一个内存位置，而其他evaluation读或修改同一内存位置，且至少一个evaluation不是原子操作，则程序的行为未定义（程序有data race），除非这两个evaluation之间存在happens-before关系。

#### 强happens-before (C++20前)

无关乎线程，若下列条件中任一成立，则evaluation A 强happens-before evaluation B：

1. A sequenced-before B
2. A synchronizes-with B
2. A 强happens-before X，且X强happens-before B

### 个人

![relation1](http://preshing.com/images/org-chart.png)

### Visible side-effects

作用于标量M上的side-effect A（写）对M上的computaion B（读）可见，要求以下的条件都成立：

1. A happens-before B
2. 不存在这样对M的side-effect X，A happens-before X，且X happens-before B

如果side-effect A对computation B可见，那么在修改顺序（modification order）中，满足B 不happens-before的对M的side effect的最长相接子集被称为side effect的可见序列（对B而言）。（B所确定的M的值，将是这些side effect之一所存储的值）

注意：线程间同步可归结于避免data races（通过建立happens-before关系），及定义在何种条件下哪些side effect成为可见的。

### Consume operation

带```memory_order_consume```或更强标签的原子存储是消费操作。注意```std::atomic_thread_fence```会施加比消费操作更强的同步要求。

### Acquire operation

带```memory_order_acquire```或更强标签的原子存储是获得操作。互斥 (Mutex) 上的 lock() 操作亦为acquire操作。注意```std::atomic_thread_fence```会施加比获得操作更强的同步要求。

### Release operation

带```memory_order_release```或更强标签的原子存储是释放操作。互斥 (Mutex) 上的 unlock() 操作亦为release操作。注意```std::atomic_thread_fence```会施加比释放操作更强的同步要求。

```cpp
r1 = y.load()
x.store(r1)

r2 = x.load()
y.store(42)

=>

y.store(42)
r2 = x.load()

=>

y.store(42)
r1 = y.load()
x.store(r1)
r2 = x.load()
r1, r2 = 42, 42
```

### Sequentially-consistent ordering




