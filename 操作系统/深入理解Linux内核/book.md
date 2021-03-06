
* [五.内核同步](#五内核同步)
    * [1.内核如何为不同的请求提供服务](#1内核如何为不同的请求提供服务)
        * [1.1 内核抢占](#11-内核抢占)
    * [2.同步原语](#2同步原语)
        * [2.1 每CPU变量](#21-每cpu变量)
        * [2.2 原子操作](#22-原子操作)
        * [2.3 优化和内存屏障](#23-优化和内存屏障)
        * [2.4 自旋锁](#24-自旋锁)
        * [2.5 读/写自旋锁](#25-读写自旋锁)
        * [2.6 顺序锁](#26-顺序锁)
        * [2.7 读-拷贝-更新(RCU)](#27-读-拷贝-更新rcu)
        * [2.8 信号量](#28-信号量)
        * [2.9 读/写信号量](#29-读写信号量)
    * [3.对内核数据结构的同步访问](#3对内核数据结构的同步访问)

<br>
<br>
<br>

# 五.内核同步

> 可以把内核看作是不断对请求进行响应的服务器，这些请求可能来自在CPU上执行的进程，也可能来自发出中断请求的外部设备

<br>

## 1.内核如何为不同的请求提供服务

把内核看作必须满足2种请求的侍者：

1. 来自顾客的请求（相当于用户态进程发出的**系统调用**或**异常**，这章剩余部分将笼统地表示为“异常”）
2. 来自数量有限的几个不同老板的请求（相当于**中断**）

对不同请求，采用如下策略：

* 老板提出请求时，如果侍者正空闲，则为老板服务；
* 如果老板提出请求时侍者正在为顾客服务，那么停止为顾客服务，开始服务老板；
* 如果老板提出请求时侍者正在为另一个老板服务，那么停止为第一个老板服务，为第二个老板服务后再继续服务第一个老板；
* 一个老板可能命令侍者停止服务顾客。在完成对老板最近请求的服务后，可能暂时不理会原来的顾客而去为新选中的顾客服务

侍者提供的服务对应于CPU处于内核态时所执行的代码、如果CPU在用户态执行，则侍者被认为处于空闲状态

### 1.1 内核抢占

无论在抢占还是非抢占内核中，运行在内核态的进程都可以**自动放弃CPU**，比如，进程由于等待资源而不得不转入睡眠状态。我们将把这种进程切换称为**计划性进程切换**。但是，抢占式内核在响应引起进程切换的异步事件（如唤醒高优先权进程的中断处理程序）的方式上与非抢占内核有差别，我们将把这种进程切换称作**强制性进程切换**

**内核抢占的主要特点是：一个在内核态运行的进程，可能在执行内核函数期间被另外一个进程取代**

下面例子说明抢占内核与非抢占内核的区别：

* 当进程A执行异常处理程序时（肯定在内核态），一个具有较高优先级的进程B变为可执行状态。例如，发生了中断请求而且相应的处理程序唤醒了进程B
    * 如果内核是抢占的，就会发生强制性进程切换，让进程B取代进程A。异常处理程序的执行暂停，直到调度程序再次选择进程A才恢复执行
    * 如果内核是非抢占的，在进程A完成异常处理程序的执行之前是不会发生进程切换的，除非进程A自动放弃CPU
* 考虑一个执行异常处理程序的进程已经用完了它的时间配额
    * 如果内核是抢占的，进程可能会立即被取代
    * 如果内核是非抢占的，进程继续运行直到它执行完异常处理程序或自动放弃CPU

**使内核可抢占的目的是减少用户态进程的分派延迟，即从进程变为可执行状态到它实际开始运行之间的时间间隔**

使Linux2.6内核具有可抢占的特性无需对支持非抢占的旧内核在设计上做太大的改变，当被`current_thread_info()`宏所引用的`thread_info`描述符的`preempt_count`字段大于`0`时，就禁止内核抢占。下列宏处理`preempt_count`字段的抢占计数器：

<div align="center"> <img src="pic/table-5-1.png" width="70%" height="70%"/> </div>

**内核抢占会引起不容忽视的开销。Linux2.6独具特色地允许用户在编译内核时通过设置选项来禁用或启用内核抢占**

<br>

## 2.同步原语

下表是Linux内核使用的同步技术。“适用范围”一栏表示同步技术是适用于系统中所有CPU还是单个CPU：

<div align="center"> <img src="pic/table-5-2.png" width="70%" height="70%"/> </div>

### 2.1 每CPU变量

**每CPU变量主要是数据结构的数组，系统的每个CPU对应数组的一个元素**

* 一个CPU不应该访问与其他CPU对应的数组元素，另外，它可以随意读或修改自己的元素而不用担心出现竞争条件，因为它是唯一有资格这么做的CPU
* 但是，这也意味着每CPU变量基本上只能在特殊情况下使用，也就是当它确定在系统的CPU上的数据在逻辑上是独立的时候

> 每CPU的数组元素在内存中被排列以使每个数据结构存放在硬件高速缓存的不同行，因此，对每CPU数组的并发访问不会导致cache-line的窃用和失效

**在单处理器和多处理器系统中，内核抢占都可能使每CPU变量产生竞争条件。总的原则是内核控制路径应该在禁用抢占的情况下访问每CPU变量**。考虑这种情况会产生什么后果——一个内核控制路径获得了它的每CPU变量本地副本的地址，然后它因被抢占而转移到另外一个CPU上，但仍然引用原来CPU元素的地址

### 2.2 原子操作

若干汇编语言指令具有“读—修改—写”类型。也就是说，它们访问存储器单元两次，第一次读原值，第二次写新值

为了避免由于“读—修改—写”指令引起的竞争条件，最容易的就是确保这样的操作在芯片级是原子的。任何一个这样的操作都必须以单个指令执行，

1. 中间不能中断
2. 且避免其他的CPU访问同一存储器单元

80x86指令：

* 进行零次或一次对齐内存访问的汇编指令是原子的
* 如果在读操作之后，写操作之前没有其他处理器占用内存总线，那么从内存中读取数据，更新数据并写回更新数据的这些“读—修改—写”汇编语言指令（如`inc`或`dec`）是原子的。当然，在单处理器系统中，永远都不会发生内存总线窃用的情况
* 操作码前缀是`lock`字节的“读—修改—写”汇编语言指令即使在多处理器系统中也是原子的。当控制单元检测到这个前缀时，就“锁定”内存总线，知道这条指令执行完成为止。所以加锁的指令执行时，其它处理器不能访问这个内存单元

C程序中，并不能保证编译器会为`a=a+1`或甚至像`a++`这样的操作使用一个原子指令。因此，Linux内核提供了一个专门的`atomic_t`类型（一个原子访问计数器）和一些专门的函数和宏，这些函数和宏作用于`atomic_t`类型的变量，并当作单独的、原子的汇编语言指令来使用。在多处理器系统中，每条这样的指令都有一个`lock`字节的前缀

### 2.3 优化和内存屏障

> 当使用优化的编译器时，不要认为指令会严格按照源代码中出现的顺序执行（例如，编译器可能重新安排汇编语言指令以使寄存器以最优的方式使用。此外，现代CPU通常并行地执行若干条指令，且可能重新安排内存访问。这种重新排序可以极大加速程序的执行）

所有的同步原语起优化和内存屏障的作用

**优化屏障**原语保证，编译程序不会混淆放在原语操作之前的汇编语言指令和放在原语操作之后的汇编语言指令

**内存屏障**原语保证，在原语之后的操作开始执行之前，原语之前的操作已经完成

### 2.4 自旋锁

自旋锁用在**多处理器环境中**

* 如果内核控制路径发现自旋锁“开着”，就获取锁并继续自己的执行
* 如果内核控制路径发现锁由运行在另一个CPU上的内核控制路径“锁着”，则反复执行一条紧凑的循环指令进行忙等，直到锁被释放

自旋锁通常非常方便，因为很多内核资源只锁1毫秒的时间片段；所以说，释放CPU和随后又获得CPU都不会消耗多少时间

一般来说，由自旋锁所保护的每个临界区都是禁止内核抢占的。在单处理系统上，这种锁本身并不起锁的作用，自旋锁的原语仅仅是禁止或启用内核抢占（注意，自旋锁忙等期间，内核抢占还是有效的，因此，等待自旋锁释放的进程有可能被更高优先级的进程替代）

### 2.5 读/写自旋锁

**只要没有内核控制路径对数据结构进行修改，读/写自旋锁就允许多个内核控制路径同时读同一数据结构。如果一个内核控制路径想对这个结构进行写操作，那么它必须首先获取读/写锁的写锁，写锁授权独占访问这个资源**

每个读/写自旋锁都是一个`rwlock_t`结构，其`lock`字段是一个32位的字段，分为两个不同的部分：

* **24位计数器**，表示对受保护的数据结构并发地进行读操作的内核控制路径的数目，这个计数器的二进制补码存放在这个字段的`0~23`位
* **“未锁”标志字段**，当没有内核控制路径在读或写时设置该位，否则清`0`.这个“未锁”标志存放在`lock`字段的第`24`位

注意：

* 如果自旋锁为空（设置了“未锁”标志且无读者），那么`lock`字段的值为`0x01000000`
* 如果写者已经获得自旋锁（“未锁”标志清`0`且无读者），那么`lock`字段的值为`0x00000000`
* 如果一个、两个或多个进程因为读获取了自旋锁，那么`lock`字段的值为`0x00ffffff`，`0x00fffffe`等

### 2.6 顺序锁

**当使用读/写锁时，内核控制路径发出的执行`read_lock`或`write_lock`操作的请求具有相同的优先权**。读者必须等待，直到写操作完成。同样地，写者也必须等待，直到读操作完成

Linux2.6引入了**顺序锁，它与读/写自旋锁非常相似，只是它为写者赋予了较高的优先级（事实上，即使在读者正在读的时候也允许写者继续运行。这种策略的好处是写者永远不会等待，除非另一个写者正在写，缺点是有些时候读者不得不反复多次读相同的数据直到它获得有效的副本）**

每个顺序锁都是包含2个字段的`seqlock_t`结构：

```c
struct seqlock_t {
    spinlock_t    lock
    int           sequence  //顺序计数器
}
```

每个读者都必须在读数据前后两次读顺序计数器，并检查两次读到的值是否相同，如果不相同，说明新的写者已经开始写并增加了顺序计数器，因此暗示读者刚读到的数据是无效的

写者通过调用`write_seqlock()`和`write_sequnlock()`获取和释放顺序锁

* 第一个函数获取`seqlock_t`数据结构中的自旋锁，然后使顺序计数器加1
* 第二个函数再次增加顺序计数器，然后释放自旋锁

这样可以保证写者写的过程中计数器的值是奇数，当没有写者在改变数据的时候，计数器的值是偶数

注意，当读者进入临界区时，不必禁用内核抢占；另一方面，由于写者获取自旋锁，所以它进入临界区时自动禁用内核抢占

### 2.7 读-拷贝-更新(RCU)

> RCU是Linux 2.6新加的功能，用在网络层和虚拟文件系统(VFS)中

读-拷贝-更新(RCU)是为了保护在多数情况下被多个CPU读的数据结构而设计的另一种同步技术。**RCU允许多个读者和写者并发执行（相对于只允许一个写者执行的顺序锁有了改进）而且，RCU你使用锁，即不使用被所有CPU共享的锁或计数器，这一点与读/写自旋锁和顺序锁（由于cache-line窃用和失效而有很高的开销）相比，RCU具有更大的优势**

RCU的关键思想包括限制RCP的范围：

1. RCU只保护被动态分配并通过指针引用的数据结构
2. 在被RCU保护的临界区中，任何内核控制路径都不能睡眠

当内核控制路径要读取被RCU保护的数据结构时，执行宏`rcu_read_lock()`，它等同于`preempt_disable()`。接下来，读者间接引用该数据结构指针所对应的内存单元并开始读这个数据结构。读者在完成对数据结构的读操作之前，是不能睡眠的。用等同于`preempt_enable()`的宏`rcu_read_unlock()`标记临界区的结束

读者几乎不做任何事来防止竞争条件出现，所以写者不得不做更多。当写者要更新数据结构时，它间接引用指针并生成整个数据结构的副本。接下来，写者修改这个副本。一旦修改完毕，写者改变指向数据结构的指针，以使它指向被修改后的副本。由于修改指针值的操作是一个原子操作，所以旧副本和新副本对每个读者或写者都是可见的，在数据结构中不会出现数据崩溃。尽管如此，还需要内存屏障来保证：只有在数据结构被修改之后，已更新的指针对其他CPU才是可见的。如果把自旋锁和RCU结合起来以禁止写者的并发执行，就隐含地引入这样的内存屏障

然而，使用RCU技术的真正困难在于：写者修改指针时不能立即释放数据结构的旧副本。实际上，写者开始修改时，正在访问数据结构的读者可能还在读旧副本。只有在CPU上的所有（潜在的）读者都执行完宏`rcu_read_unlock()`之后，才可以释放旧副本。内核要求每个潜在的读者在下面的操作之前执行`rcu_read_unlock()`：

* CPU执行进程切换
* CPU开始在用户态执行
* CPU执行空循环

对上述每种情况，我们说CPU已经经过静止状态

写者调用函数`call_rcu()`来释放数据结构的旧副本。当所有的CPU都通过静止状态之后，`call_rcu()`接受`rcu_head`描述符（通常嵌在要被释放的数据结构中）的地址和将要调用的回调函数的地址作为参数。一旦回调函数被执行，它通常释放数据结构的旧副本

函数`call_rcu()`把回调函数和其参数的地址存放在`rcu_head`描述符中，然后把描述符插入回调函数的每CPU(per-CPU)链表中。内核每经过一个时钟滴答就周期性地检查本地CPU是否经过了一个静止状态。如果所有CPU都经过了静止状态，本地tasklet（它的描述符存放在每CPU变量`rcu_tasklet`中）就执行链表中的所有回调函数

### 2.8 信号量

> 这里主要指内核信号量，Linux提供的System V IPC信号量，由用户态进程使用

内核信号量类似于自旋锁，因为当锁关闭时，它不允许内核控制路径继续进行。然而，当内核控制路径试图获取内核信号量所保护的忙资源时，相应的进程被挂起。只有在资源被释放时，进程才再次变为可运行的。因此，只有可以睡眠的函数才能获取内核信号量：中断处理程序和可延迟函数都不能使用内核信号量

### 2.9 读/写信号量

读/写信号量类似于”读/写自旋锁“，不同之处是：信号量再次变为打开之前，等待的进程挂起而不是自旋

很多内核控制路径为读可以并发地获取读/写信号量。但是，任何写者内核控制路径必须有对被保护资源的互斥访问。因此，只有在没有内核控制路径为读访问或写访问持有信号量时，才可以为写获取信号量。读/写信号量可以提高内核中的并发度

内核以严格的FIFO顺序处理等待读/写信号量的所有进程。如果读者或写者进程发现信号量关闭，这些进程就被插入到信号量等待队列链表的末尾。当信号量被释放时，就检查处于等待队列链表第一个位置的进程。第一个进程被唤醒。如果是一个写者进程，等待队列上其他的进程就继续睡眠。如果是一个读者进程，那么紧跟第一个进程的其它所有读者进程也被唤醒并获得锁。不过，在写者进程之后排队的读者进程继续睡眠

<br>

## 3.对内核数据结构的同步访问

系统中的并发度取决于两个主要因素：

* 同时运转的I/O设备数
* 进行有效工作的CPU数

为了使I/O吞吐量最大化，应该使中断禁止保持在很短的时间。正如第四章的”IRQ和中断“一节描述的那样，当中断被禁止时，由I/O设备产生的IRQ被PIC暂时忽略，因此，就没有新的活动在这种设备上开始

为了有效地利用CPU，应该尽可能避免使用基于自旋锁的同步原语。当一个CPU执行紧指令循环等待自旋锁打开时，是在浪费宝贵的机器周期。就像我们前面所描述的，更糟糕的是：由于自旋锁对硬件高速缓存的影响而使其对系统的整体性能产生不利影响

在以下两种情况下，既可以维持较高的并发度，也可以达到同步：

* 共享的数据结构是一个单独的整数值，可以把它声明为`atomic_t`类型并使用原子操作对其更新。原子操作比自旋锁和中断禁止都快，只有在几个内核控制路径同时访问这个数据结构时速度才会慢下来
* 把一个元素插入到共享链表的操作决不是原子的，因为这至少涉及两个指针赋值。不过，内核有时并不用锁或禁止中断就可以执行这种插入操作。考虑如下代码，在汇编语言中，插入简化为两个连续的原子指令。第一条指令建立`new`元素的`next`指针，但不修改链表。因此，如果中断处理程序在第一条指令和第二条指令执行的中间查看这个链表，看到的就是没有新元素的链表。如果该处理程序在第二条指令执行后查看链表，就会看到有新的元素的链表。关键是，在任一情况下，链表都是一致的且处于未损坏状态。然而，只有在中断处理程序不修改链表的情况下才能保证这种完整性。如果修改了链表，那么在`new`元素内刚刚设置的`next`指针就可能变为无效的。同时，开发者必须确保两个赋值操作的顺序不被编译器或CPU控制单元搅乱。否则，如果中断处理程序在两个赋值之间中断了系统调用服务例程，处理程序就会看到一个损坏的链表。因此，就需要一个写内存屏障原语

    ```c
    new->next = list_element->next;
    list_element->next = new;
    ```
