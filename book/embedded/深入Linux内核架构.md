# 深入 Linux 内核架构
## 第一章-简介和概述
### 内核的任务
1. 应用程序角度-内核可以被认为是一台增强的计算机；
2. 将内核视为资源管理程序；
3. 将内核视为库。

### 实现策略
1. 微内核
2. 宏内核

### 内核的组成部分
- [ ] 图，Linux 内核的高层次概述及完整的 Linux 系统中的各个层次

1. 进程、进程切换、调度
2. UNIX 进程
   - Linux 对进程采用了一种层次系统，**每个进程都依赖于一个父进程**；
   - 内核启动 init 程序作为第一个进程，它是进程树的根，进程树可通过 `pstree` 程序查看；
   - 创建新进程的两种机制：`fork` 和 `exec`
   - 线程：
     - Linux 用 `clone` 方法创建线程；
   - 命名空间：
     - [ ] 命名空间的定义？
     - 不同的进程可以看到不同的系统视图；
     - 启用命名空间后，以前的全局资源现在具有不同的分组。每个命名空间可以包含一个特定的 PID 集合，或可以提供文件系统的不同视图，在某个命名空间中挂载的卷不会传播到其他命名空间中；
     - 用处：通过称为容器的命名空间来建立系统的多个视图，从容器内部看来这是一个完整的 Linux 系统，而且与其他容器没有交互，且每个容器实例看起来就像是运行 Linux 的一台计算机。
3. 地址空间与特权级别
   - 地址空间
     - 内存区域通过指针寻址，CPU 的字长决定地址空间最大长度；
     - 地址空间的最大长度与实际可用的物理内存数量无关，称为虚拟地址空间；
     - 在系统中的每个进程看来，地址空间中只有自身一个进程，而无法感知到其他进程的存在。应用程序无需关注其他程序的存在；
     - 虚拟地址空间分为内核空间（`TASK_SIZE~2^32 或 2^64`）和用户空间（`0~TASK_SIZE`）；
     - 由于地址空间虚拟化的效果，每个用户进程都认为自身有 3GB 内存；
     - 虚拟地址空间顶部的内核空间总是同样的，无论当前执行的是哪个进程；
     - 64 位计算机倾向于使用小于 64 的位数，如 42 位或 47 位；
     - 与寻址完整的虚拟地址空间相比，管理有效地址空间所需的位数较少，因此 CPU 可以节省一些工作量。这样，**虚拟地址空间会包含一些不可寻址的洞**；
     - **用页表来为物理地址分配虚拟地址**；
     - **虚拟地址关系到进程的用户空间和内核空间，而物理地址则用来寻址实际可用的内存**；
     - 物理内存和虚拟地址空间都被划分为很多等长的部分，称之为页，物理内存和虚拟地址空间的页大小相等；
     - **物理内存页经常称为页帧，页则专指虚拟地址空间中的页**；
     - **虚拟地址空间的所有页并非都映射到某个页帧**，这可能是因为页没有使用，或者是数据尚不需要使用而没有载入内存中，还可能是页已经换出到硬盘，将在需要时再换回内存；
   - 特权级别
     - 每个特权级别都有各种限制，例如对执行某些汇编语言指令或访问虚拟地址空间某一特定部分的限制；
     - IA-32 使用 4 中特权级别，但 **Linux 只使用两种不同的状态：核心态和用户状态**；
     - 两种状态的关键差别在于对高于 `TASK_SIZE` 的内存区域的访问；
     - 从用户状态到核心态的切换通过**系统调用**的特定转换手段完成、内核会检查进程是否允许执行想要的操作；
     - 除了代表用户程序执行代码之外，内核还可以由异步硬件**中断**激活，然后在中断上下文中运行。与在进程上下文中运行的主要区别是，**在中断上下文中运行不能访问虚拟地址空间中的用户空间部分**；
     - 内核线程也不与任何特定的用户空间进程关联，因此也无权处理用户空间。在其他许多方面，内核线程更像是普通的用户层应用程序。`ps` 命令将内核线程的名称置于方括号内；
     - 在多处理器系统上，许多线程启动时指定了 CPU，并限制只能在某个特定的 CPU 上运行。从内核线程名称之后的斜线和 CPU 编号可以看到这一点。
4. 页表
   - 用来将虚拟地址空间映射到物理地址空间的数据结构称为页表；
   - 使用功能相同但内存用量少得多的模型：多级分页；
   - 为减少页表的大小并容许忽略不需要的区域，计算机体系结构的设计会将虚拟地址划分为多个部分：（4 部分-三级页表，Linux 四级页表）
     - [ ] 图，分配虚拟地址
     - `PGD`, Page Global Directory, 全局页目录，用于索引进程中的一个数组，每个进程有且仅有一个，该数组就是所谓的全局页目录或 PGD;
     - `PMD`, Page Middle Directory, 中间页目录，数组项指向页表或页目录;
     - `PTE`, Page Table Entry, 页表数组，数组项指向页帧;
     - 偏移量：指定页内部的一个字节位置；
   - 页表的一个特色在于，对虚拟地址空间中不需要的区域，不必创建中间页目录或页表，与单个数组方法相比，多级页表节省了大量内存；
   - 多级页表缺点：地址转换需要逐级访问多个数组，CPU 使用两种方法加速该过程：
     - MMU, Memory Management Unit，该单元优化内存访问操作；
     - 最频繁使用的地址被保存到地址转换后备缓冲器（Translation Lookasize Buffer, `TLB`），是 CPU 高速缓存，从而减少对内存的访问；
   - 与 CPU 的交互
   - 内存映射
5. 物理内存的分配
   - 内核分配内存时，记录页帧的状态（空闲或已分配）、尽快完成相关操作、可以只分配完整的页帧，将划分为更小的部分的工作委托给用户空间中的标准库；
   - 伙伴系统：
     - 内存分配需求：分配连续页，为快速检测出内存中的连续区域，内核采用伙伴系统；
       - [ ] 图，伙伴系统
     - 暂且理解伙伴系统的实现类似于内存池的实现；
     - 伙伴系统可以减少内存碎片，但无法完全消除；
   - `slab` 缓存：
     - 内核本身经常需要比完整页帧小得多的内存块，而内核无法使用标准库函数，**因而必须在伙伴系统基础上自行定义额外的内存管理层**，将伙伴系统提供的页划分为更小的部分。该方法不仅可以分配内存，还为频繁使用的小对象实现了一个一般性的缓存—— slab 缓存。它可以用两种方法分配内存。
     - 对频繁使用的对象，内核定义了只包含了所需类型对象实例的缓存；
     - 对通常情况下小内存块的分配，内核针对不同大小的对象定义了一组 slab 缓存；
     - 对微小的嵌入式系统来说，slab 分配器的开销可能太大；而对于规模庞大的超级计算机来说，又出现了可伸缩性问题。内核提供了 slab 分配器的两种备选方案，可用于在相应的场景下替换 slab 分配器并提供更好的性能。对内核的其他部分而言，这 3 种方案的接口相同。
     - [ ] 图，页帧的分配由伙伴系统进行...
   - 页面交换和页面回收
     - 页面交换通过利用磁盘空间作为扩展内存，从而增大了可用的内存；通过缺页机制，换入和换出操作对应用程序是透明的；
     - 页面回收用于将内存映射被修改的内容与底层的块设备同步，为此有时也简称为**数据写回**。数据刷出后，内核即可将页帧用于其他用途（类似于页面交换）。内核的数据结构包含了与此相关的所有信息，当再次需要该数据时，可根据相关信息从硬盘找到对应的数据并加载。
6. 计时
   - 全局变量 `jiffies_64` 和 `jiffies`、内核常熟 `HZ` (通常介于 100 和 1000 之间)；
   - 在底层硬件能力允许的前提下，内核可使用高分辨率的定时器提供额外的计时手段，能够以纳秒级的精确度和分辨率来计量时间；
   - 计时的周期是可以动态改变的。在没有或无需频繁的周期性操作的情况下，周期性地产生定期中断是没有意义的，这会阻止处理器降低耗电进入睡眠状态。**动态改变计时器周期对于供电受限的系统是很有用的**，例如笔记本电脑和嵌入式系统。
7. 系统调用
   - POSIX 标准定义了许多系统调用，类别分组：
     - 进程管理
     - 信号
     - 文件
     - 目录和文件系统
     - 保护机制
     - 定时器函数
   - 对所有处理器来说，一个共同点是：用户进程要从用户态切换到核心态，并将系统关键任务委派给内核执行，系统调用是必由之路。
8. 设备驱动程序、块设备和字符设备
   - 按照经典的 UNIX 箴言“万物皆文件”（everything is a file），对外设的访问可利用 `/dev` 目录下的设备文件来完成，程序对设备的处理完全类似于常规文件。
   - 字符设备：通常不支持随机存取；
   - 块设备：支持随机访问设备数据、数据的读写只能以块（通常是 512B）的倍数进行、与字符设备不同，块设备并不支持基于字符的寻址；
   - 编写块设备的驱动程序比字符设备要复杂得多，因为内核为提供系统性能广泛地使用了缓存机制。
9. 网络
   - 网卡也可以通过设备驱动程序控制，但**网卡不能利用设备文件访问**。原因在于网络通信期间，数据打包到了各个协议层中。
   - 为支持通过文件接口处理网络连接，Linux 使用了源于 BSD 的套接字抽象。套接字可以看作应用程序、文件接口、内核的网络实现之间的代理。
10. 文件系统
    - 不同的文件系统所基于的概念抽象，在某种程度上可以说是南辕北辙；
    - `Ext2` 基于 `inode`，即它对每个文件都构造了一个单独的管理结构，称为 inode，并存储到磁盘上。inode 包含了文件所有的元信息，以及指向相关数据块的指针。目录可以表示为普通文件，其数据包括了指向目录下所有文件的 inode 的指针，因而层次结构得以建立。
    - 内核必须提供一个额外的软件层，将各种底层文件系统的具体特性与应用层（和内核自身）隔离开来。该软件层称为 `VFS`（Virtual Filesystem 或 Virtual Filesystem Switch），虚拟文件系统。
    - VFS 既是向下的接口（所有文件系统都必须实现该接口），同时也是向上的接口（用户进程通过系统调用最终能够访问文件系统功能）。
    - [ ] 图，虚拟文件系统层、...
11. 模块和热插拔
12. 缓存
    - 由于内核是通过基于页的内存映射来实现访问块设备的，因此缓存也按页组织，也就是说整页都缓存起来，故称为页缓存（page cache）。
    - 块缓存已经被页缓存取代了。
13. 链表处理
    - 双链表
    - 不是类型安全的
    - 假如链表的数据结构必须包含一个类型为 `list_head` 的成员、可用于将任何类型的数据结构彼此链接起来
    - 如果有若干链表涉及同一数据结构，那么结构中就需要同样数目的 list_head 成员
    - 链表的起点同样是 list_head 的实例，通常用 `LIST_HEAD(list_name)` 宏来声明并初始化
    - 处理链表的标准函数：...
14. 对象管理和引用计数
    - 一般性的内核对象机制可用于执行下列对象操作：
      - 引用计数
      - 管理对象链表（集合）
      - 集合加锁
      - 将对象属性导出到用户空间（通过 sysfs 文件系统）
    - 一般性的内核对象：`kobject`
    - 对象集合：`kset`
    - 引用计数：`kref`
15. 数据类型
    - 类型定义
      - 内核使用 typedef 来定义各种数据类型，以避免依赖于体系结构相关的特性。**如果某个变量的类型是 typedef 而来的，则不能直接访问，而需要通过辅助函数，这样做确保了相应数据类型的正确操作，尽管对用户来说类型定义其实是透明的**。
      - 在某些时候内核必须使用精确定义了位数的变量，为此内核定义了若干整数数据类型，不仅明确标明了是有符号数还是无符号数，而且还指定了相关类型的精确位数。例如 `__s8` 和 `__u8` 分别是有符号和无符号的 8 位整数；`__u16` 和 `__s16`、`__s32` 和 `__u32`、`__u64` 和 `__s64` 等；
    - 字节序
      - 表示如何存储**多字节**数据类型；
      - 内核提供了各种函数和宏，可以在 CPU 使用的格式与特定的表示法之间转换。`cpu_to_le64` 将 64 位数据类型转换为小端序格式、`le64_to_cpu` 则相反。如果体系结构采用的字节序是小端序格式，这两个例程当然是空操作，否则必须相应地交换字节位置。
    - per-cpu 变量
      - 通过 `DEFINE_PER_CPU(name, type)` 声明，`name` 为变量名，而 `type` 是其数据类型；
      - 在单处理器系统上，这与常规的变量声明没有不同。在有若干个 CPU 的 SMP 系统上，**会为每个 CPU 分别创建变量的一个实例**。
      - 用于某个特定 CPU 的实例可以通过 `get_cpu(name, cpu)` 获得，其中 `smp_processor_id()` 可以返回当前活动处理器的 id，用作前述的 cpu 参数。
      - 使用 per-cpu 变量的好处：所需数据很可能存在于处理器的缓存中，因此可以更快地访问。如果在多处理器系统使用可能被所有 CPU 同时访问地变量，可能会引发一些通信方面的问题，采用上述概念刚好绕过了这些问题。
    - 访问用户空间
      - 指针标记为 `__user`，标识指向用户地址空间中区域的指针；
      - 在没有进一步预防措施的情况下，不能轻易访问这些指针指向的区域；
      - 因为内存是通过页表映射到虚拟地址空间的用户空间部分的，而不是物理内存直接映射的。因此内核需要确保指针所指向的页帧确实存在于物理内存中。
      - 通过显式标记，可以支持利用自动检查工具（`sparse`）来确认实际上遵守了必要的条件。
16. 本书的局限性

### 为什么内核是特别的
- 内核调试问题
- 内核的辅助函数
- 内核错误
- 必须考虑到内核运行的许多体系结构上根本**不支持非对齐的内存访问**
- **所有的内核代码都必须是并发安全的**。由于对多处理器计算机的支持，Linux 内核代码必须是可重入和线程安全的。也就是说，程序必须允许同时执行，而数据必须针对并行访问进行保护。
- 内核代码**必须在小端序和大端序计算机上都能够工作**
- **大多数体系结构根本不允许在内核中执行浮点计算**，因此计算需要想办法用整型来替代。

### 行为注记

## 第二章-进程管理和调度
在本章中，主要讲解内核共享 CPU 时间的方法，以及如何在进程之间切换。这里有两个任务，其执行是相对独立的。
1. 内核必须决定为各个进程分配多长时间，何时切换到下一个进程，以及哪个进程是下一个进程。此类决策是平台无关的，即调度策略（CPU 时间如何分配）。
2. 在内核从进程 A 切换到进程 B 时，必须确保进程 B 的执行环境与上一次撤销其处理器资源时完全相同。这项工作与处理器极度相关，不能只用 C 语言实现，还需要汇编代码的帮助。
这两个任务是称之为**调度器**的内核子系统的职责。

### 进程优先级
进程有**优先级**，还有**关键度**。
1. 实时进程和非实时进程（较粗糙的划分）
   - *硬实时*进程有严格的时间限制。
     - 关键特征是它们必须在可保证的时间范围内得到处理，但这并不意味着所要求的时间范围特别短，而是系统必须保证绝不会超过某一时间范围。
     - Linux 不支持硬实时处理，但在一些修改版本中如 RTLinux、Xenomai、RATI 提供了该特性。
     - 在这些修改后的方案中，Linux 内核作为独立的“进程”运行来处理次重要的软件，而实时的工作则在内核外部完成。只有当没有实时的关键操作执行时，内核才会运行。
     - 降低内核整体延迟的相关工作：可抢占的内核机制、实时互斥量、完全公平调度器。
   - *软实时*进程是硬实时进程的一种弱化形式。
     - 仍然需要快速得到结果，但允许出现较小延迟。
     - 进程在需要 CPU 时间时应该能够得到保证，至少优先于所有其他普通进程。
   - *普通进程*，没有特定时间约束。
     - 仍然可以根据重要性来分配优先级。
2. 抢占式多任务处理（preemptive multitasking），时间段到期后，内核会从进程收回控制权，让一个不同的进程运行，而不考虑前一进程所执行的上一个任务。
3. 在某些时间无事可做的进程不能执行（不能立即运行的进程）、支持不同的调度类别（在进程之间完全公平的调度和实时调度）、重要进程就绪时可抢占当前进程。
4. 调度器的变化：
   - `O(1)` 调度器
   - 完全公平调度器（completely fair scheduler）。该调度器的关键特性是，它视图**尽可能地模仿理想情况下的公平调度**。此外，**它不仅可以调度单个进程，还能够处理更一般性的调度实体（scheduling entity）**。例如，该调度器分配可用时间时，可以首先在不同用户之间分配，接下来在各个用户的进程之间分配。


### 进程生命周期
将 CPU 时间分配到无事可做的进程，显然是没有意义的。进程在各个状态之间的转换也同样重要。当调度器在进程之间切换时，必须知道系统中每个进程的状态。

进程可能的状态：
- 运行
- 等待
- 睡眠
  - 根据是否可以被信号中断，有两种睡眠状态。
- [ ] 图，进程状态之间的转换
  - 没有列出的一个特殊进程状态是“僵尸”状态。实际上，说这些进程死了，是因为其资源（内存、与外设的连接等等）已经释放，因此它们无法也绝不会再次运行。说它们仍然活着，是因为进程表中仍有对应的表项。
  - 在进程终止之后，其数据尚未从进程表删除之前，进程总是暂时处于“僵尸”状态。
  - 有时候（例如，如果父进程编程及其糟糕，没有发出 `wait` 调用），僵尸进程可能稳定地寄身于进程表中，直至下一次系统重启。从进程工具（如 `ps` 或 `top`）的输出，可以看到僵尸进程。*因为残余的数据在内核中占据的空间极少，所以这几乎不是问题*。

**系统将所有进程保存在一个进程表中，无论其状态是什么。但睡眠进程会特别标记出来，调度器会知道它们无法立即运行。睡眠进程会分类到若干队列中**。

### 抢占式多任务处理
Linux 进程管理的结构中还需要另外两种进程状态选项：用户态和核心态。

内核的抢占调度模型建立了一个层次结构，用于判断哪些进程状态可以由其他状态抢占。
- 普通进程总是可能被抢占，甚至是由其他进程抢占。
- 如果系统处于核心态并正在处理系统调用，那么系统中的其他进程是无法夺取 CPU 时间的。调度器必须等到系统调用执行结束，才能选择另一个进程执行，但中断可以终止系统调用。
  - 在内核 2.5 开发期间，一个称之为内核抢占（kernel preemption）的选项添加到内核。该选项支持在紧急情况下切换到另一个进程，甚至当前是处于核心态执行系统调用（中断处理期间是不行的）。
- 中断可以暂停处于用户状态和核心态的进程，中断具有最高优先级，因为在中断触发后需要尽快处理。

### 进程表示
Linux 内核涉及进程和程序的所有算法都围绕一个名为 `task_struct` 的数据结构建立，该结构定义在 `include/sched.h` 中。

`task_struct` 包含很多成员，将进程与各个内核子系统联系起来。

定义内容，部分略：
```c
struct task_struct {
    volatile long state;    // -1 表示不可运行，0 表示可运行，> 0 表示停止
    void *stack;
    atomic_t usage;
    unsigned long flags;    // 每进程标志，下文定义
    unsigned long ptrace;
    int lock_depth;         // 大内核锁深度

    int prio, static_prio, normal_prio;
    struct list_head run_list;
    const struct sched_class *sched_class;
    struct sched_entity se;

    unsigned short ioprio;

    unsigned long policy;
    cpumask_t cpus_allowed;
    unsigned int time_slice;

#if defined(CONFIG_SCHEDSTATS) || defined(CONFIG_TASK_DELAY_ACCT)
    struct sched_info sched_info;
#endif

    struct list_head tasks;

    /* 
     * ptrace_list/ptrace_children 链表是 ptrace 能够看到的当前进程的子进程列表
     */
    struct list_head ptrace_children;
    struct list_head ptrace_list;
    struct mm_struct *mm, *active_mm;

    /* 进程状态 */
    struct linux_binfmt *binfmt;
    long exit_state;
    int exit_code, exit_signal;
    int pdeath_signal;      // 父进程终止时发送的信号

    unsigned int personality;
    unsigned did_exec:1'
    pid_t pid;
    pid_t tgid;
    /* 分别是指向（原）父进程、最年轻的子进程、年幼的兄弟进程
     * 年长的兄弟进程的指针
     * (p->father 可以替换为 p->parent->pid)
     */
    struct task_struct *real_parent;    // 真正的父进程（在被调试的情况下）
    struct task_struct *parent;
    /* children/sibling 链表外加当前调试的进程，构成了当前进程的所有子进程 */
    struct list_head children;          // 子进程链表
    struct list_head sibling;           // 连接到父进程的子进程链表
    struct task_struct *group_leader;   // 线程组组长

    /* PID 与 PID 散列表的联系 */
    struct pid_link pids[PIDTYPE_MAX];
    struct list_head thread_group;

    struct completion *vfork_done;      // 用于 vfork()
    int __user *set_child_tid;          // CLONE_CHILD_SETTID
    int __user *clear_child_tid;        // CLONE_CHILD_CLEARTID

    unsigned long rt_priority;
    cputime_t utime, stime, utimescaled, stimescaled;
    unsigned long nvcsw, nivcsw;        // 上下文切换计数
    struct timespec start_time;         // 单调时间
    struct timespec real_start_time;    // 启动以来的时间
    /* 内存管理器失效和页交换信息，这个有一点争论。它既可以看作是
     * 特定于内存管理器的，也可以看作是特定于线程的
     */
    unsigned long min_flt, maj_flt;

    cputime_t it_prof_expires, it_virt_expires;
    unsigned long long it_sched_expires;
    struct list_head cpu_timers[3];

    /* 进程身份凭据 */
    uid_t uid, euid, suid, fsuid;
    gid_t gid, egid, sgid, fsgid;
    struct group_info *group_info;
    kernel_cap_t cap_effective, cap_inheritable, cap_permitted;

    unsigned keep_capabilities:1;
    struct user_struct *user;

    char comm[TASK_COMM_LEN];   // 除去路径后的可执行文件名称，
                                // 用 [gs]et_task_comm 访问（其中用 task_lock() 锁定它）
                                // 通常由 flush_old_exec 初始化
    /* 文件系统信息 */
    int link_count, total_link_count;
    /* ipc 相关 */
    struct sysv_sem sysvsem;
    /* 当前进程特定于 CPU 的状态信息 */
    struct thread_struct thread;
    /* 文件系统信息 */
    struct fs_struct *fs;
    /* 打开文件信息 */
    struct files_struct *files;
    /* 命名空间 */
    struct nsproxy *nsproxy;
    /* 信号处理程序 */
    struct signal_struct *signal;
    struct sighand_struct *sighand;

    sigset_t blocked, read_blocked;
    sigset_t saved_sigmask;     // 用 TIF_RESTORE_SIGMASK 恢复
    struct sigpending pending;

    unsigned long sas_ss_sp;
    size_t sas_ss_size;
    int (*notifier)(void *priv);
    void *notifier_data;
    sigset_t *notifier_mask;

#ifdef CONFIG_SECURITY
    void *security;
#endif

    /* 线程组跟踪 */
    u32 parent_exec_id;
    u32 self_exec_id;
    /* 日志文件系统信息 */
    void *journal_info;
    /* 虚拟内存状态 */
    struct reclaim_state *reclaim_state;
    
    struct backing_dev_info *backing_dev_info;

    struct io_context *io_context;

    unsigned long ptrace_message;
    siginfo_t *last_siginfo;        // 由 ptrace 使用
}
```

该结构的内容可以分解为各个部分，每个部分表示进程的一个特定方面：
- **状态和执行信息**：如待决信号、使用的二进制格式（和其他系统二进制格式的任何仿真信息）、进程 ID 号、到父进程及其他有关进程的指针、优先级和程序执行有关的时间信息（例如 CPU 时间）。
- 有关已经分配的虚拟内存的信息。
- 进程身份凭据，如用户 ID、组 ID 以及权限等。
- 使用的文件包含程序代码的二进制文件，以及进程所处理的所有文件的文件系统信息。
- 线程信息记录该进程特定于 CPU 的运行时间数据（该结构的其余字段与所使用的硬件无关）。
- 在与其他应用程序协作时所需的进程间通信有关的信息。
- 该进程所使用的信号处理程序，用于响应到来的信号。

`task_struct` 中对进程管理的实现特别重要的一些成员。
- `state`: 指定进程的当前状态：
  - `TASK_RUNNING`，进程处于可运行状态。该状态确保进程可以立即运行，而无需等待外部事件。
  - `TASK_INTERRUPTIBLE`，针对等待某事件或其他资源的睡眠进程设置的。在内核发送信号给该进程表明事件已经发生时，进程状态变为 `TASK_RUNNING`，它只要调度器选中该进程即可恢复执行。
  - `TASK_UNINTERRUPTIBLE`，不能由外部信号唤醒，只能由内核亲自唤醒。
  - `TASK_STOPPED`，表示进程特意停止运行，例如由调试器暂停。
  - `TASK_TRACED`，本来不是进程状态，用于从停止的进程中，将当前被调试的那些（使用 `ptrace` 机制）与常规的进程区分开来。
  - `EXIT_ZOMBIE`，也可用于 `exit_state` 字段，表示僵尸状态；
  - `EXIT_DEAD`，也可用于 `exit_state` 字段，指 `wait` 系统调用已经发出，而进程完全从系统移除之前的状态。只有多个线程对同一个进程发出 `wait` 调用时，该状态才有意义。
- `struct rlimit`，Linux 提供资源限制（resource limit, rlimit）机制，对进程使用系统资源施加某些限制。
  - 数据描述：
    ```c
    struct rlimit {
      unsigned long rlim_cur;
      unsigned long rlim_max;
    }
    ```
    - `rlim_cur` 是进程当前的资源限制，也称软限制（soft limit）；
    - `rlim_max` 是该限制的最大容许值，因此也称为硬限制（hard limit）；
  - `rlim` 数组中的位置标识了受限制资源的类型；
  - 系统调用 `setrlimit` 来增减当前限制，但不能超出 `rlimi_max` 指定的值。`getrlimits` 用于检查当前限制。
  - [ ] 表，特定于进程的资源限制
  - 如果某一类资源没有使用限制（几乎所有资源的默认设置），则将 `rlim_max` 设置为 `RLIM_INFINITY`。例外情况：
    - 打开文件的数目 `RLIMIT_NOFILE`，默认限制在 1024；
    - 每用户的最大进程数 `RLIMIT_NPROC`，定义为 `max_threads/2`。`max_threads` 是一个全局变量，指定了在把八分之一可用内存用于管理线程信息的情况下，可以创建的线程数目。在计算时，提前给定了 20 个线程的最小可能内存用量。
  - init 进程的限制在系统启动时即生效，定义在 `include/asm-generic-resource.h` 中的 `INIT_RLIMITS`。
  - 当前 rlimit 值的查看：`cat /proc/self/limits`

1. 进程类型
   - UNIX 进程包括的内容
   - 新进程的生成：fork 和 exec
   - clone 系统调用
2. 命名空间
  - 创建
    - `fork` 或 `clone`
    - `unshare`
  - 内核文档：`Documentation/namespaces/*`
  - 实现
    - 两部分：每个子系统的命名空间结构（`struct nsproxy`），将此前所有的全局组件包装到命名空间中；将给定进程关联到所属各个命名空间的机制。
    - [ ] 图，进程和命名空间之间的联系
  - `struct nsproxy`：
    ```c
    struct nsproxy {
      atomic_t count;
      struct uts_namespace *uts_ns;
      struct ipc_namespace *ipc_ns;
      struct mnt_namespace *mnt_ns;
      struct pid_namespace *pid_ns;
      struct user_namespace *user_ns;
      struct net *net_ns;
    };
    ```
   - 可以感知到命名空间的内核范围：
     - UTS 命名空间包含了运行内核的名称、版本、底层体系结构类型等信息。UTS 是 UNIX Timesharing System 的简称；
     - 保存在 struct ipc_namespace 中的所有与进程间通信（IPC）有关的信息；
     - 已经装载的文件系统的视图，在 struct mnt_namespace 中给出；
     - 有关进程 ID 的信息，由 struct pid_namespace 提供；
     - struct user_namespace 保存的用于限制每个用户资源使用的信息；
     - struct net_ns 包含所有网络相关的命名空间参数。
   - 使用 fork 建立新的命名空间，提供的相关标志：
    ```c
    <sched.h>
    #define CLONE_NEWUTS    0x04000000    /* 创建新的 utsname 组 */
    #define CLONE_NEWIPC    0x08000000    /* 创建新的 IPC 命名空间 */
    #define CLONE_NEWUSER   0x10000000    /* 创建新的用户命名空间 */
    #define CLONE_NEWPID    0x20000000    /* 创建新的 PID 命名空间 */
    #define CLONE_NEWNET    0x40000000    /* 创建新的网络命名空间 */
    ```
   - 每个进程都关联到自身的命名空间视图：
    ```c
    <sched.h>
    struct task_struct {
      ...
      struct nsproxy *nsproxy;
      ...
    }
    ```
    - 因为使用了指针，多个进程可以共享一组子命名空间。这样，修改给定的命名空间，对所有属于该命名空间的进程都是可见的。
   - 对命名空间的支持必须在编译时启用，而且必须逐一指定需要支持的命名空间。但对命名空间的一般性支持总是会编译到内核中。但如果内核编译时没有指定对具体命名空间的支持，默认命名空间的作用类似于不启用命名空间，所有的属性都相当于全局的。
   - `init_nsproxy` 定义了初始的全局命名空间：
    ```c
    <kernel/nsproxy.c>
    struct nsproxy init_nsproxy = INIT_NSPROXY(init_nsproxy);

    <init_task.h>
    #define INIT_NSPROXY(nsproxy) { \
      .pid_ns = &init_pid_ns, \
      .count  = ATOMIC_INIT(1), \
      .uts_ns = &init_uts_ns, \
      .mnt_ns = NULL, \
      INIT_NET_NS(net_ns) \
      INIT_IPC_NS(ipc_ns) \
      .user_ns = &init_user_ns, \
    }
    ```
   - UTS 命名空间
     - 几乎不需要特别处理，因为它只需要简单量，没有层次组织；
     - 所有相关信息都汇集到下列结构的一个实例中：
      ```c
      <utsname.h>
      struct uts_namespace {
        struct kref kfre;
        struct new_utsname name;
      };

      // uts_namespace 所提供的属性信息本身包含在 struct new_utsname 中
      struct new_utsname {
        char sysname[65];
        char nodename[65];
        char release[65];
        char version[65];
        char machine[65];
        char domainname[65];
      };
      // 各个字符串分别存储了系统的名称、内核发布版本、机器名等等。
      // 使用 uname 工具可以取得这些属性的当前值
      // 也可以在 /proc/sys/kernel 中看到：cat /proc/sys/kernel/ostype...

      // 初始设置保存在 init_uts_ns 中
      // init/version.c
      struct uts_namespace init_uts_ns = {
        ...
        .name = {
          .sysname = UTS_SYSNAME,
          .nodename = UTS_NODENAME,
          .release = UTS_RELEASE,
          .version = UTS_VERSION,
          .machine = UTS_MACHINE,
          .domainname = UTS_DOMAINNAME,
        },
      };
      ```
     - 在某个进程调用 fork 并通过 CLONE_NEWUTS 标志指定创建新的 UTS 命名空间时，调用 `copy_utsname` 函数。在这种情况下，会生成先前的 `uts_namespace` 实例的一份副本，当前进程的 nsproxy 实例内部的指针会指向新的副本。如此而已。
   - 用户命名空间
     - 要求创建新的用户命名空间时，则生成当前用户命名空间的一份副本；
     - 数据描述：
      ```c
      <user_namespace.h>
      struct user_namespace {
        struct kref kref;
        struct hlist_head uidhash_table[UIDHASH_SZ];
        struct user_struct *root_user;
      }
      ```
      - 对命名空间中的每个用户，都有一个 struct user_struct 的实例负责记录其资源消耗，各个实例可以通过散列表 uidhash_table 访问；
      - struct user_struct 维护了一些统计数据，如进程和打开文件的数目；
    - 每个用户命名空间对其用户资源使用的统计，与其他命名空间完全无关，对 root 用户的统计也是如此。这是因为在克隆一个用户命名空间时，为当前用户和 root 都创建了新的 user_struct 实例：
      ```c
      // kernel/user_namespace.c
      static struct user_namespace *clone_user_ns(struct user_namespace *old_ns) {
        ...
      }
      ```
3. 进程 ID 号
  UNIX 进程总是会分配一个号码用于**在其命名空间**中唯一地标识它们。该号码被称作进程 ID 号，简称 PID。用 fork 或 clone 产生的每个进程都由内核自动地分配了一个新的唯一的 PID 值。
  - 进程 ID
    - PID
    - 线程组 ID (TGID)，如果进程没有使用线程，则其 PID 和 TGID 相同；
    - 进程组组长的 PID，即 task_struct->pgrp、task_struct->group_leader 指向组长的 task_struct 实例；
    - 会话 ID (SID)，保存在 task_struct->session 成员中；
    - 在创建一个新命名空间时，该命名空间中的所有 PID 对父命名空间都是可见的，但子命名空间无法看到父命名空间的 PID。但这意味着某些进程具有多个 PID，凡可以看到该进程的命名空间，都会为其分配一个 PID。这必须反映在数据结构中，我们必须区分局部 ID 和全局 ID。
      - 全局 ID：在内核本身和初始命名空间中的唯一 ID 号，在系统启动期间开始的 init 进程即属于初始命名空间。对每个 ID 类型，都有一个给定的全局 ID，保证在整个系统中是唯一的。
      - 局部 ID：属于某个特定的命名空间，不具备全局有效性。对每个 ID 类型，它们在所属的命名空间内部有效，但类型相同、值也相同的 ID 可能出现在不同的命名空间中。
      - 全局 PID 和 TGID 直接保存在 task_struct 中，分别是 pid 和 tgid 成员。
   - 管理 PID
     - 数据结构 `struct pid_namespace`、`struct upid`、 `struct pid`
     - [ ] 图，实现可感知命名空间的 ID 表示所用的数据结构
     - 给出局部数据 ID 和对应的命名空间，查找此二元组描述的 task_struct （将数字 PID 和命名空间转换为 pid 实例）
     - 给出 task_struct、ID 类型、命名空间，取得命名空间布局的数字 ID（将 task_struct 实例变为数字 ID）
     - 生成唯一 PID
4. 进程关系
   - 父子进程
   - 兄弟关系
  新的子进程置于 sibling 链表的起始位置，这意味着可以重建进程分支的时间顺序。

### 进程管理相关的系统调用
讨论 fork 和 exec 系列系统调用的实现。通常这些调用不是由应用程序直接发出的，而是通过一个中间层调用，即负责与内核通信的 C 标准库。

1. 进程复制
   - fork
     - 重量级调用，建立父进程的一个完整副本；
     - 使用写时复制（copy-on-write）技术；
   - vfork
     - 不创建副本，父子进程之间共享数据，节省大量 CPU 时间；
     - 用于子进程形成后立即执行 execve 系统调用加载新程序的情形；
     - 由于 fork 使用了写时复制技术，vfork 速度方面不再有优势，因此应该避免使用它；
   - clone
     - 产生线程
     - 可以对父子进程之间的共享、复制进行精确控制
   - 写时复制
     - 进程通常只使用了其内存页的一小部分。（进程访问最频繁的页的集合被称为工作区（working set））在调用 fork 时，内核通常对父进程的每个页，都为子进程创建一个相同的副本，这有两种很不好的负面效应（如果应用程序在进程复制后使用 exec 立即加载新程序，那么负面效应会更严重）：
       - 使用了大量内存
       - 复制操作耗费很长时间
     - 并不复制进程的整个地址空间，而是只复制其页表。这样就建立了虚拟地址空间和物理内存页之间的联系。因此，fork 之后父子进程的地址空间指向相同的物理内存页
     - 父子进程不能允许修改彼此的页，（两个进程显式共享的页除外）这也是两个进程的页表对页标记了只读访问的原因，即使在普通环境下允许写入也是如此。
     - 只要有一个进程试图向复制的内存页写入，处理器会向内核报告访问错误（此类错误被称作缺页异常）。内核然后查看额外的内存管理数据结构，检查该页是否可以用读写模式访问，还是只能以只读模式访问。如果是后者，则必须向进程报告段错误。如果页表项将一页标记为“只读”，但通常情况下该页应该是可写的，内核可以根据此条件来判断该页实际上是 COW 页。因此内核会创建该页专用于当前进程的副本，当然也可以用于写操作。
     - COW 机制使得内核可以尽可能延迟内存页的复制，更重要的是，在很多情况下不需要复制。这节省了大量时间。
   - 执行系统调用
     - 上述系统调用的入口点分别是 sys_fork、sys_vfork 和 sys_clone 函数。其定义依赖于具体的体系结构，因为在用户空间和内核空间之间传递参数的方法因体系结构而异。
     - 上述函数的任务是从处理器寄存器中提取由用户空间提供的信息，调用体系结构无关的 `do_fork` 函数，后者负责进程复制。
     - do_fork 函数：
      ```c
      // kernel/fork.c
      long do_fork(unsigned long clone_flags, unsigned long stack_start,
        struct pt_regs *regs, unsigned long stack_size, 
        int __user *parent_tidptr, int __user *child_tidptr);
      ```
      - `clone_flags`：是一个标志集合，用来指定控制复制过程的一些属性。最低字节指定了在子进程终止时被发给父进程的信号号码。其余的高位字节保存了各种常数。
      - `stack_start`：用户状态下栈的起始地址。
      - `regs`：指向寄存器集合的指针，其中以原始形式保存了调用参数。该参数使用的数据类型是特定于体系结构的 struct pt_regs，其中按照系统调用执行时寄存器在内核栈上存储顺序，保存了所有的寄存器。
      - `stack_size` 是用户状态下栈的大小，该参数通常是不必要的，设置为 0。
      - 后续两个参数分别指向父子进程的 PID（指向用户空间中地址）。
    - 不同的 fork 变体，主要是通过标志集合区分。
   - do_fork 的实现
    - [ ] 图，do_fork 的代码流程图
   - 复制进程
    - [ ] 图，copy_process 的代码流程图
    - 虽然一般而言指针可以指向内存中的任意位置，而 Linux 支持的每个体系结构的虚拟地址空间中都有一个从虚拟地址 0 到 4KiB 的区域，该区域中没有任何有意义的信息。因此内核可以重用该地址范围来编码错误码。
    - 如果 fork 的返回值指向前述的地址范围内部，那么该调用就失败了，其原因可以由指针的数值判断。`ERR_PTR` 是一个辅助宏，用于将数组常数（例如 EINVAL，非法操作）编码为指针。
    - 一方面请求创建一个新命名空间（`CLONE_NEWNS`），而同时要求与父进程共享所有的文件系统信息（`CLONE_FS`）是没有意义的；
    - 用 `CLONE_THREAD` 创建一个线程时，必须用 `CLONE_SIGHAND` 激活信号共享。通常情况下，一个信号无法发送到线程组中的各个线程；
    - 只有在父子进程之间共享虚拟地址空间时（`CLONE_VM`），才能提供共享的信号处理程序。因此类似的想法是，要想达到同样的效果，线程也必须与父进程共享地址空间。
    - 在内核建立了自洽的标志集之后，则用 `dup_task_struct()` 来建立父进程 *task_struct* 的副本。用于子进程的新的 *task_struct* 实例可以在任何空闲的内核内存位置分配。父子进程的 *task_struct* 实例只有一个成员不同：新进程分配了一个新的核心态栈，即 `task_struct->stack`。
    - 栈和 `thread_info`
      - 通常栈和 thread_info 一同保存在一个联合中，thread_info 保存了线程所需的所有特定于处理器的底层信息：
        ```c
        <sched.h>
        union thread_union {
          struct thread_info thread_info;
          unsigned long stack[TRHEAD_SIZE/sizeof(long)];
        };
        ```
     - 原则上，只要设置了预处理器常数 `__HAVE_THREAD_FUNCTIONS` 通知内核，那么各个体系结构可以随意在 `stack` 数组中存储什么数据。在这种情况下，它们必须自行实现 `task_thread_info()` 和 `task_stack_page()`，这两个函数用于获取给定 `task_struct` 实例的线程信息和核心态栈。另外，它们必须实现 `dup_task_struct()` 中调用的函数 `setup_thread_stack()`，以便确定 `stack` 成员的具体内存布局。
     - 在大多数体系结构上，使用一两个内存页来保存一个 thread_union 的实例。
     - 在 IA-32 上，两个内存页是默认设置，因此可用的内核栈长度略小于 8KiB，其中一部分被 thread_info 实例占据。配置选项 4KSTACKS 会将栈长度降低到 4KiB，即一个页面。
       - 如果系统上有许多进程在运行，这样做是有利的，因为每个进程可以节省一个页面；
       - 另一方面，对于经常趋向于使用过多栈空间的外部驱动程序来说，这可能导致问题。标准发布版所提供的内核，其所有核心部分都已经设计为能够在 4 KiB 栈长度配置下运转流畅。
     - thread_info 保存了特定于体系结构的汇编语言代码需要访问的那部分进程数据。大多数对 thread_info 的描述类似于：
      ```c
      <asm-arch/thread_info.h>
      struct thread_info {
        struct task_struct *task;           // 当前进程 task_struct 指针
        struct exec_domain *exec_domain;    // 执行区间
        unsigned long flags;                // 底层标志
        unsigned long status;               // 线程同步标志
        __u32 cpu;                          // 当前 CPU
        int preempt_count;                  // 0 => 可抢占，<0 => BUG
        mm_segment_t addr_limit;            // 线程地址空间
        struct restart_block restart_block;
      }
      ```
      - `exec_domain` 用于实现执行区间，后者用于在一类计算机上实现多种的 ABI。例如，在 AMD64 系统的 64bit 模式下运行 32bit 应用程序；
      - `flags` 可保存各种特定于进程的标志，对其中两个特别感兴趣：
        - `TIF_SIGPENDING`：进程有待决信号则对其置位；
        - `TIF_NEED_RESCHED`：表示该进程应该或想要调度器选择另一个进程替换本进程执行。
        - 其他可用的常数是特定于硬件的，几乎从不使用，可以参见 `<asm-arch/thread_info.h>`
      - `cpu` 说明进程正在其上执行的 CPU 数目；
      - `preempt_count`：实现内核抢占所需的一个计数器；
      - `addr_limit`：指定了进程可以使用的虚拟地址的上限。该限制适用于普通进程，但内核线程可以访问整个虚拟地址空间，包括只有内核能访问的部分。这并不意味着限制进程可以分配的内存数量。
      - `restart_block`：用于实现信号机制。
     - [ ] 进程的 task_struct、thread_info 和内核栈之间的关系
       - 在内核的某个特定组件使用了过多栈空间时，内核栈会溢出到 thread_info 部分，这很可能会导致严重的故障。此外在紧急情况下输出调用栈回溯时将会导致错误的信息出现，因此内核提供了 `kstack_end` 函数，用于判断给出的地址是否位于栈的有效部分之内。
       - `dup_task_struct` 会复制进程 task_struct 和 thread_info 实例的内容，但 stack 则与新的 thread_info 实例位于同一个内存区域。这意味着父子进程的 task_struct 此时除了栈指针之外是完全相同的，但子进程的 task_struct 实例会在 copy_process 过程中修改。
       - `current_thread_info` 可获得指向当前执行进程的 thread_info 实例的指针。其地址可以根据内核栈指针确定，因为 **thread_info 实例总是位于栈顶**。因为每个进程分别使用各自的内核栈，进程到栈的映射是唯一的。（指向内核栈的指针通常保存在一个特别保留的寄存器中）
       - `current` 给出了当前进程 task_struct 实例的地址。该函数在源码中出现非常频繁。该地址可以使用 current_thread_info() 确定：current = current_thread_info()->task。
     - dup_task_struct 之后，内核检查当前的特定用户在创建新进程之后，是否超出了允许的最大进程数据；
     - 接下来调用 sched_fork，以便使调度器有机会对新进程进行设置。本质上，该例程会初始化一些统计字段，在多处理器系统上，如果有必要可能还会在各个 CPU 之间对可用的进程重新均衡以下；
     - 调用形如 `copy_xyz` 的例程，以便复制或共享特定的内核子系统的资源；
       - `COPY_SYSVSEM` 置位，...
       - `COPY_FILES` 置位，...
       - `COPY_FS` 置位，...
       - `CLONE_SIGHAND` 或 `CLONE_THREAD` 置位，...
       - `CLONE_THREAD` 置位，...
       - `COPY_MM` 置位，...
       - `COPY_MM` 没有置位，...
       - `copy_namespace`
       - `copy_thread`：这是一个特定于体系结构的函数，用于复制进程中特定于线程（thread-specific）的数据。重要的是填充 task_struct->thread 的各个成员。这是一个 `thread_struct` 类型的结构，其定义是体系结构相关。它包含了所有寄存器（和其他信息），内核在进程之间切换时需要保存和恢复进程的内容，该结构可用于此。
     - 内核填好 task_struct 中对父子进程不同的各个成员。
   - 创建线程时的特别问题
     - 在 Linux 内核中，线程和一般进程之间的差别不是那么刚性，这两个名词经常用作同义词（如前所述，线程也经常用于指进程的体系结构相关部分）。
     - `CLONE_PARENT_SETTID`
     - `CLONE_CHILD_SETTID`
     - `CLONE_CHILD_CLEARTID`

2. 内核线程
   - 内核线程是直接由内核本身启动的进程。
   - 内核线程经常称之为（内核）守护进程。它们用于执行下列任务：
     - 周期性地将修改的内存页与页来源块设备同步（例如，使用 mmap 的文件映射）；
     - 如果内存页很少使用，则写入交换区；
     - 管理延时动作（deferred action）
     - 实现文件系统的事务日志
   - 两种内核线程：
     - 线程启动后一直等待，直至内核请求线程执行某一特定操作；
     - 线程启动后按周期性间隔运行，检测特定资源的使用。内核使用这类线程用于连续监测任务。
   - 内核线程会出现在系统进程列表中，但在 `ps` 的输出中由方括号包围，以便与普通进程区分。如果内核线程绑定到特定的 CPU，CPU 的编号在斜线后给出。
   - 启动内核线程函数（其定义是特定于体系结构的，但原型总是相同的）：
    ```c
    <asm-arch/processor.h>

    int kernel_thread(int (*fn)(void *), void *arg, unsigned long flags);
    ```
   - 产生的线程将执行用 `fn` 指针传递的函数，而用 `arg` 指定的参数将自动传递给该函数。`flags` 中可指定 `CLONE` 标志。
   - `kernel_thread` 的第一个任务是构建一个 `pt_regs` 实例，对其中的寄存器指定适当的值。这与普通的 `fork` 系统调用类似。接下来调用 `do_fork()`。
   - 内核线程，在 CPU 管态（supervisor mode）执行、只可以访问虚拟地址空间的内核部分（高于 TASK_SIZE 的所有地址），但不能访问用户空间。
   - 在内核代表用户层程序运行时（例如执行系统调用），虚拟地址空间的用户空间部分由 task_struct->mm 指向的 mm_struct 实例描述。每当内核执行上下文切换时，虚拟地址空间的用户层部分都会切换，以便与当前运行的进程匹配。
   - 惰性 TLB 处理（lazy TLB handling），没有 mm 指针的进程称作惰性 TLB 进程。假如内核线程之后运行的进程与之前是同一个，在这种情况下，内核并不需要修改用户空间地址表，地址转换后备缓冲器（即 TLB）中的信息仍然有效。只有在内核线程之后执行的进程是与此前不同的用户层进程时，才需要切换（并对应清除 TLB 数据）。
   - 由于内核线程不与任何特定的用户层进程相关，内核并不需要倒换虚拟地址空间的用户层部分，保留旧设置即可。由于内核线程之前可能是由任何用户层进程在执行，因此用户空间部分的内容本质上是随机的，内核线程绝不能修改其内容。为强调用户空间部分不能访问，mm 设置为空指针。但由于内核必须知道用户空间当前包含了什么，所以在 task_struct->active_mm 中保存了指向 mm_struct 的一个指针来描述它。
   - 当内核在进程上下文下运转时，mm 和 active_mm 的值相同。
   - 内核线程启动的多种方式：
     - `kernel_thread`-古老的方法。将一个函数直接传递给 `kernrl_thread`。该函数接下来负责帮助内核调用 `daemonize` 以转换为守护进程。这依次引发下列操作：
       - 该函数从内核线程释放其父进程（用户进程）的资源（如内存上下文、文件描述符，等等），不然这些资源会一直锁定到线程结束，这是不可取的，因为守护进程通常运行到系统关机为止。因为守护进程只操作内核地址区域，它甚至不需要这些资源。
       - `daemonize` 阻塞信号的接收。
       - 将 `init` 用作守护进程的父进程。
     - `kthread_create`-更现代的方法。
       - 该函数创建一个新的内核线程，其名称由函数参数 `namefmt` 给出；
       - 最初线程是停止的，需要使用 `wake_up_process` 启动它。
       - 唤醒后，会调用通过函数参数 `thread_fn` 给出的线程函数，而参数 `data` 则作为参数。
     - `kthread_run`，备选方案，参数与 `kthread_create` 相同。
       - 调用 `kthread_create` 创建新线程，但立即唤醒它。
     - `kthread_create_cpu` 创建绑定到特定 cpu 的内核线程。

3. 启动新程序
  通过用新代码替换现存程序，即可启动新程序。Linux 提供的 execve 系统调用可用于该目的。
   - execve 的实现
     - 入口点：sys_execve
     - 执行函数：do_execve
      ```c
      // kernel/exec.c
      int do_execve(char *filename, char __user *__user *argv,
        char __user *user *envp, struct pt_regs *regs);
      ```
   - [ ] do_execve 的代码流程图
   - 二进制格式处理程序执行的操作：...
   - Linux 支持的二进制格式：...
   - 解释二进制格式：
    ```c
    <binfmts.h>
    struct linux_binfmt {
      struct linux_binfmt *next;
      struct module *module;
      int (*load_binary)(struct linux_binprm *, struct pt_regs *regs);
      int (*load_shlib)(struct file *);
      int (*core_dump)(long signr, struct pt_regs *regs, struct file *file);
      unsigned long min_coredump;   // minimal dump size，通常是一个内存页的长度
    }
    ```
    - 每种二进制格式首先必须使用 `regiseter_binfmt` 向内核注册。相关内容见 `fs/exec.c-formats`。

1. 退出进程
  进程必须用 exit 系统调用终止。这使得内核有机会将该进程使用的资源释放回系统。
  
  程序员可以显式调用 exit。但编译器会在 main 函数（或特定语言使用的 main 函数）末尾自动添加相应的调用。

  该调用的入口点是 sys_exit，需要一个错误码作为参数。它很快将工作委托给 do_exit。

  简而言之，该函数的实现就是将各个引用计数器减一，如果引用计数器归 0 而没有进程再使用对应的结构，那么相应的内存区域返还给内存管理模块。

### 调度器的实现
任务分为两个不同部分：一个涉及策略调度，另一个涉及上下文切换。

1. 概观
   - `schedule` 函数是理解调度操作的起点，定义于 `kernel/sched.c` 中。
   - 调度器的实现受若干因素的影响而稍显模糊：
     - 多处理器系统
     - 优先调度 + Posix 标准需要的其他两种软实时策略
     - 使用 goto 以生成最优的汇编语言代码
   - 完全公平调度器
     - 不需要时间片概念，至少不需要传统的时间片（之前需要）；
     - 只考虑进程的等待时间，即进程在就绪队列（run-queue）中已经等待了多长时间。对 CPU 时间需求最严格的进程被调度执行。
   - 调度器的一般原理是，按所能分配的计算能力，向系统中的每个进程提供最大的公正性。
   - [ ] 调度器通过将进程在红黑树中排序，跟踪进程的等待时间
   - 就绪队列还装备了虚拟时钟。该时钟的时间流逝速度慢于实际的时钟，精确的速度依赖于当前等待调度器挑选的进程的数目。
   - 进程的等待时间保存在 wait_runtime，虚拟时间由 fair_clock 给出，为排序红黑树上的进程，内核使用差值 fair_clock - wait_runtime。fair_clock 是完全公平调度的情况下进程将会得到的 CPU 时间的度量，而 wait_runtime 直接度量了实际系统的不足造成的不公平。
   - 在进程允许运行时，将从 wait_runtime 减去它已经运行的时间。
   - 该策略受若干现实问题的影响，已经变得复杂了：
     - 进程的不同优先级（即 nice 值）必须考虑，更重要的进程必须比次要进程更多的 CPU 时间份额；
     - 进程不能切换得太频繁，上下文切换开销、两次相邻的任务切换之间，时间不能太长，否则会累积比较大的不公平值。
   - 理解调度决策的一个好方法是，在编译时激活调度器统计。这会在运行时生成文件 `/proc/sched_debug`，其中包含了调度器当前状态所有方面的信息。
   - `Documentation/` 目录下的文件可能已经过时了。

2. 数据结构
   - [ ] 图，调度子系统各组件概观
   - 激活调度：主动放弃 CPU + 周期性调度 -> 通用调度器或核心调度器；
   - 调度器类用于判断接下来与性能哪个进程，内核支持不同的调度策略（完全公平调度、实时调度、在无事可做时调度空闲进程）、以模块化方法实现策略；
   - 通用调度器自身完成不涉及进程管理，其工作都委托给调度器类。
   - `task_struct` 的成员
    ```c
    <sched.h>
    struct task_struct {
      ...
      int prio, static_prio, normal_prio;
      unsigned int rt_priority;

      struct list_head run_list;
      const struct shced_class *sched_class;
      struct sched_entity se;

      unsigned int policy;
      cpumask_t cpus_allowed;
      unsigned int time_slice;
      ...
    }
    ```
    - `prio` 和 `normal_prio` 表示动态优先级、`static_prio` 表示进程的静态优先级
      - `static_prio` 是进程启动时分配的优先级，可以用 `nice` 和 `sched_setscheduler` 系统调用修改，否则在进程运行期间会一直保持恒定）；
      - `normal_prio` 是基于进程的静态优先级和调度策略计算出的优先级。因此，即使普通进程和实时进程具有相同的静态优先级，其 normal_prio 也是不同的。进程分支时，子进程会继承 normal_prio；
      - 调度器考虑的优先级保存在 `prio`。
    - `rt_priority`：0-99，值越大优先级越高
    - `sched_class`：表示进程所属的调度器类
    - sched_entity，调度器不限于调度进程，还可以处理更大的实体。这可以用于实现组调度：可用的 CPU 时间可以首先在一般的进程组（例如按所有者分组）之间分配，接下来分配的时间在组内分配。这种一般性要求调度器不直接操作进程，而是处理可调度实体。由于调度器设计为处理可调度的实体，在调度器看来各个进程必须也像是这样的实体。因此在 task_struct 内嵌了 sched_entity 实例，注意这里不是指针，调度器可据此操作各个 task_struct。
    - `policy`，对该进程应用的调度策略，Linux 支持 5 种：
      - `SCHED_NORMAL`：用于普通进程，它们**通过完全公平调度器来处理**。`SCHED_BATCH` 和 `SCHED_IDLE` 也通过完全公平调度器来处理，不过可用于次要的进程。`SCHED_BATCH` 用于非交互、CPU 使用密集的批处理进程。它们绝不会抢占 CF 调度器处理的另一个进程，因此不会干扰交互式进程。如果不打算用 nice 降低进程的静态优先级，同时又不希望该进程影响系统的交互性，此时最适合使用该调度类。`SCHED_IDLE` 进程的重要性也比较低，因为其相对权重总是最小的。`SCHED_IDLE` 不负责调度空闲进程，空闲进程由内核提供的单独机制来处理。
      - `SCHED_RR` 和 `SCHED_FIFO` 用于实现软实时进程。`SCHED_RR` 实现了一种循环方法，而 `SCHED_FIFO` 则使用先进先出机制。**这些不是完全公平调度器类处理，而是由实时调度器类处理**。辅助函数 `rt_policy` 用于判断给出的调度策略是否属于实时类（`SCHED_RR` 和 `SCHED_FIFO`）。`task_has_rt_policy` 用于对给定进程判断该性质。
    - `cpus_allowed` 是一个位域，在多处理器系统上使用，用来限制进程可以在哪些 CPU 上运行。可使用 `sched_setaffinity` 系统调用设置该位图。
    - `run_list` 和 `time_slice` 是循环实时调度器所需要的，但不用于完全公平调度器。`run_list` 用于维护包含各进程的一个运行表，而 `time_slice` 则指定进程可使用 CPU 的剩余时间段。
    - 前文提到的 `TIF_NEED_RESCHED` 标志，对调度器而言，和 `task_struct` 中上述与调度相关的成员同样重要。如果对活动进程设置该标志，调度器即知道 CPU 将从该进程收回并授予新进程，这可能是自愿的，也可能是强制的。
   - 调度器类
    ```c
    <sched.h>
    struct sched_class {
      ...
    }
    ```
    - 调度类之间的层次结构是平坦的：实时进程最重要，在完全公平进程之前处理；而完全公平进程则优先于空闲进程；空闲进程只有 CPU 无事可做时才处于活动状态。
    - next 成员将不同调度类的 sched_class 实例，按上述顺序连接起来。要注意**这个层次结构在编译时已经建立：没有在运行时动态增加新调度器类的机制**。
    - `enqueue_task` 向就绪队列添加一个新进程。在进程从睡眠状态变为可运行状态时，即发生该操作。

## 第三章-内存管理
内存管理是内核最复杂同时也是最重要的一部分。其特点在于非常需要处理器和内核之间的协作（所需执行的任务决定了二者必须紧密合作）。

### 概述
1. 内存管理的实现涵盖的领域
   - 内存中的物理内存页的管理；
   - 分配大块内存的伙伴系统；
   - 分配较小块内存的 slab、slub 和 slob 分配器；
   - 分配非连续内存块的 vmalloc 机制；
   - 进程地址空间

2. Linux 内核一般将处理器的虚拟地址空间划分为两个部分。底部较大的部分用于用户进程，顶部则专用于内核。虽然（在两个用户进程之间的）上下文切换期间会改变下半部分，但虚拟地址空间的内核部分总是保持不变。典型的划分比例是 3：1。

3. 可用的物理内存将映射到内核的地址空间中。访问内存时，如果所用的虚拟地址与内核区域的起始地址之间的偏移量不超出可用物理内存的长度，那么该虚拟地址会自动关联到物理页帧。因为在内核区域中的内存分配总是落入到物理内存中。

4. 如果物理内存比可以映射到内核地址空间中的数量要多，那么内核必须借助于高端内存（highmem）方法来管理“多余的”内存。
   - 只有内核自身使用高端内存页时，才会有问题。
   - 在内核使用高端内存页之前，必须使用 kmap 和 kunmap 函数将其映射到内核虚拟地址空间中，对普通内存页这是不必要的。
   - 对用户空间进程来说，是高端内存页还是普通内存页完全没有任何差别。因为用户空间进程总是通过页表访问内存，绝不会直接访问。

5. PAE, page address extension，在启用 PAE 模式的情况下，IA-32 可以管理多达 64 GiB 内存。
   - 但并非所有 64 GiB 都可以同时寻址，而是每次只能寻址一个 4 GiB 的内存段。
   - 由于内存超过 4 GiB 的 IA-32 系统比较罕见，这种情况下实际上一般会使用 64 位体系结构 AMD64 来代替 IA-32，提供一个简洁得多的解决方案。
   - 在 64 位计算计上，由于可用的地址空间非常巨大，因此不需要高端内存模式。即使物理寻址受限于地址字的比特位数（例如 48 或 52），也是如此。

6. 管理物理内存的两种不同方法
   - UMA, uniform memory access，即一致内存访问。将可用内存以连续方式组织起来（可能有小缺口），SMP 系统中每个处理器访问各个内存区都是同样快。
   - NUMA，non-uniform memory access，即非一致内存访问。总是多处理器计算机。系统的各个 CPU 都有本地内存，可支持特别快速的访问。各个处理器之间通过总线连接起来，以支持对其他 CPU 的本地内存的访问，当然比访问本地内存慢些。
   - [ ] 图，UMA 和 NUMA 系统

7. 两种类型计算机的混合也是可能的，其中使用不连续的内存。即在 UMA 系统中，内存不是连续的，有比较大的洞。在这里应用 NUMA 体系结构的原理通常有帮助，可以使内核的内存访问更简单。
   - 由于 UMA 系统可以在地址空间包含比较大的空洞时选择配置选项 `CONFIG_DISCONTIGMEM`，这种情况下在不采用 NUMA 计数的系统上也会有多个内存节点。
   - [ ]，概述在 UMA 和 NUMA 计算机上可能的内存配置：...

8. 内存组织类型的 3 种配置：
   - `FLATMEM`
   - `DISCONTIGMEM`
   - `SPARSEMEM`
  `SPARSEMEM` 和 `DISCONTIGMEM` 实际上作用相同，但从开发者角度看，对应的代码质量有所不同。前者被认为更多是试验性的，不那么稳定，但有一些性能优化；而后者相关代码更稳定一些，但不具备内存热插拔之类的新特性。
  大多数配置中都使用 `FLATMEM`，通常它也是内核的默认值。
  由于所有内存模型实际上都使用同样的数据结构，因此不讨论其他选项也没多大损失。

9. 真正的 NUMA 会设置配置选项 `CONFIG_NUMA`，相关的内存管理代码与上述两种变体有所不同。由于平坦内存模型在 NUMA 计算机上没有意义，只有不连续内存模型和稀疏内存模型是可用的。`NUMA_EMU` 配置选项用于平坦内存模型对 NUMA 系统的模拟。NUMA 计算机过于昂贵。

10. 分配阶（allocation order），它表示内存区中页的数目取以 2 为底的对数。例如，阶 0 的分配由一个页面组成，阶 1 的分配包括 2 个页...

### (N)UMA 模型中的内存组织
1. Linux 支持的各种不同体系结构在内存管理方面差别很大，由于内核的明智设计，以及某些情况下插入的兼容层，这些差别被很好地隐藏起来了（一般性的代码通常无需注意这些）。
   - 页表中不同数目的间接层
   - NUMA 和 UMA 系统的划分

2. 内核对 UMA 和 NUMA 系统使用相同的数据结构，因此针对各种不同形式的内存布局，各个算法几乎没什么差别。在 UMA 系统上，只使用一个 NUMA 结点来管理整个系统内存。而内存管理的其他部分则相信它们是在处理一个伪 NUMA 系统。

3. 概述
   - 内存划分为结点。每个结点关联到系统中的一个处理器，在内核中表示为 `pg_data_t` 实例；
   - 各个结点又划分为内存域；
    ```c
    // 内核引入了下列常量来枚举系统中的所有内存域
    <mmzone.h>

    enum zone_type {
    #ifdef CONFIG_ZONE_DMA
      ZONE_DMA,
    #endif
    #ifdef CONFIG_ZONE_DMA32
      ZONE_DMA32,
    #endif
      ZONE_NORMAL,
    #ifdef CONFIG_HIHGMEM
      ZONE_HIGHMEM,
    #endif
      ZONE_MOVABLE,
      MAX_NR_ZONES,
    };
    ```
    - `ZONE_DMA32` 标记了使用 32 位地址字可寻址、适合 DMA 的内存域。只有在 64 位系统上，与 `ZONE_DMA` 才有差别。在 32 位计算机上，本内存域是空的，即长度为 0 MiB。在 64 位系统上，该内存域的长度可能从 0 到 4 GiB。
    - `ZONE_NORMAL`：可直接映射到内核段的普通内存域。在所有体系结构上保证都会存在的唯一内存域，但无法保证该地址范围对应了实际的物理内存。例如，如果 AMD64 系统上有 2 GiB 内存，那么所有内存都属于 ZONE_DMA32，而 ZONE_NORMAL 则为空。
    - `ZONE_HIGHMEM` 标记了超出内核段的物理内存。
    - `ZONE_MOVABLE`，伪内存域，在防止物理内存碎片的机制中需要使用到该内存域。
    - 根据编译时的配置，可能无需考虑某些内存域。例如在 64 位系统中，并不需要高端内存域。如果支持了只能访问 4 GiB 以下内存的 32 位外设，才需要 DMA32 内存域。
   - 各个内存域都关联了一个数组，用来组织属于该内存域的物理内存页（内核中称之为页帧）。
   - 对每个页帧，都分配了一个 `struct page` 实例以及所需的管理数据。
     - 系统中所有结点的页帧是依次编号的；
     - 每个页帧的号码都是全局唯一的，不只是结点内唯一；
     - 编号从 0 开始；
   - 各个内存结点保存在一个单链表中，供内核遍历。出于性能考虑，在为进程分配内存时，内核总是试图在当前运行的 CPU 相关联的 NUMA 结点上进行。但这并不总是可行的，例如该结点的内存可能已经用尽，对此类情况，每个结点都提供了一个备用列表（借助于 `struct zonelist`）。该列表包含了其他结点（和相关的内存域），可用于代替当前结点分配内存。列表项的位置越靠后，就越不适合分配。

4. 数据结构
   - 结点管理：`pg_data_t`、`node_states`
   - 内存域：`zone`、内存域水印计算
   - 冷热页、页帧
   - `pg_data_t`
    ```c
    <mmzone.h>
    typedef struct pglish_data {
      struct zone node_zones[MAX_NR_ZONES];
      struct zonelist node_zonelists[MAX_ZONELISTS];
      int nr_zones; // 结点中不同内存域的数目
      struct page *node_mem_map;  // page 数组，结点的所有物理内存页，包含了结点中所有内存域的页
      struct bootmem_data *bdata; // 系统启动期间，内存管理子系统初始化之前使用，指向自举内存分配器

      unsigned long node_start_pfn; // 第一个页帧的逻辑编号
      unsigned long node_present_pages; // 结点中页帧的数目
      unsigned long node_spanned_pages; // 结点以页帧为单位计算的长度。与上述值不一定相同，因为结点中可能有些空洞，并不对应真正的页帧。

      int node_id;  // 全局结点 ID，从 0 开始编号
      struct pglist_data *pgdat_next; // 下一个内存结点，单链表，以空指针结尾
      wait_queue_head_t kswapd_wait;  // 交换守护进程（swap daemon）的等待队列，在将页帧换出结点时会用到
      struct task_struct *kswapd;     // 负责该结点的交换守护进程的 task_struct
      int kswapd_max_order;           // 用于页交换子系统的实现，用来定义需要释放的区域的长度
    } pg_data_t
    ```
   - `node_states`
     - 如果系统中结点多于一个，内核会维护一个位图，用以提供各个结点的状态信息。状态是用位掩码指定的，可使用的值：
      ```c
      <nodemask.h>
      enum node_states {
        N_POSSIBLE,       // 结点在某个时候可能变为 online
        N_ONLINE,         // 结点是 online 的
        N_NORMAL_MEMORY,  // 结点有 NORMAL_MEMORY
      #ifdef CONFIG_HIGHMEM
        N_HIGH_MEMORY,    // 结点有 NORMAL_MEMORY 或 HIGH_MEMORY
      #else
        N_HIGH_MEMORY = N_NORMAL_MEMORY,
      #endif
        N_CPU,            // 结点有一个或多个 CPU
        N_NODE_STATES
      };

      // 设置或清除一个比特位
      void node_set_state(int node, enum node_states state)
      void node_clear_state(int node, enum node_states state)

      // 用来迭代处于特定状态的所有结点
      for_each_node_state(__node, __state)
      // 迭代所有活动结点
      for_each_online_node(node)
      ```
      - `N_POSSIBLE`、`N_ONLINE` 和 `N_CPU` 用于 CPU 和内存的热插拔；
      - 如果结点有普通或高端内存则使用 `N_HIGH_MEMORY`，仅当结点没有高端内存才设置 `N_NORMAL_MEMORY`；
      - 如果内核编译为只支持单个结点（即平坦内存模型），则没有结点位图，上述操作该位图的函数则变为空操作。
   - `struct zone`
    ```c
    <mmzone.h>
    struct zone {
      // 通常由页分配器访问的字段
      unsigned long pages_min, pages_low, pages_high;

      unsigned long lowmem_reserve[MAX_NR_ZONES];

      struct per_cpu_pageset pageset[NR_CPUS];

      // 不同长度的空闲区域
      spinlock_t lock;
      struct free_area free_area[MAX_ORDER];

      ZONE_PADDING(_pad1_)

      // 通常由页面回收扫描程序访问的字段
      spinlock_t lru_lock;
      struct list_head active_list;
      struct list_head inactive_list;
      unsigned long nr_scan_active;
      unsigned long nr_scan_inactive;
      unsigned long pages_scanned;    // 上一次回收以来扫描的页
      unsigned long flags;

      // 内存域统计量
      atomic_long_t vm_stat[NR_VM_ZONE_STAT_ITEMS];

      int prev_priority;

      ZONE_PADDING(_pad2_)
      
      // 很少使用或大多数情况下只读的字段
      wait_queue_head_t *wait_table;
      unsigned long wait_table_hash_nr_entries;
      unsigned long wait_table_bits;

      // 支持不连续内存模型的字段
      struct pglist_data *zone_pgdat;
      unsigned long zone_start_pfn;

      unsigned long spanned_pages;  // 总长度，包含空洞
      unsigned long present_pages;  // 内存数量（除去空洞）

      // 很少使用的字段
      char *name;
    } __cacheline_maxaligned_in_smp;
    ```
     - 内核对该结构的访问非常频繁；
     - 由 `ZONE_PADDING` 分隔为几个部分，将两个自旋锁成员 `lock` 和 `lru_lock` 分隔在两个不同的部分，在使用高速缓冲时确保每个自旋锁都处于自身的缓冲行中；
     - 编译关键字 `__cacheline_maxaligned_in_smp` 用以实现最优的高速缓存对齐方式；
     - `pages_min`, `pages_high`, `pages_low` 是页换出时使用的“水印”。
       - `pages_high`: 如果空闲页多于该值，则内存域的状态是理想的；
       - `pages_low`: 如果空闲页少于该值，则内核开始将页换出到硬盘；
       - `pages_min`: 如果空闲页少于该值，则页回收工作压力较大，因内存域急需空闲页；
     - `lowmem_reserve`: 分别为各种内存域指定了若干页，用于一些无论如何都不能失败的关键性内存分配；
     - `pageset`: 数组，用于实现每个 CPU 的热/冷页帧列表；热页-存在于高速缓存中的页帧、冷页-未缓存的页帧；与之相关的数据结构：`struct per_cpu_pageset`；
     - `free_area`: 用于伙伴系统，每个数组元素都表示某种固定长度的一些连续内存区，对于包含在每个区域中的空闲内存页的管理，free_area 是一个起点；
     - 第二部分涉及的结构成员，用来根据活动情况对内存域中使用的页进行编目。如果页访问频繁，则内核认为它是活动的。在需要换出页时，如果可能的话，频繁使用的页应该保持不动，而多余的不活动页则可以换出而没有什么损害；
     - `active_list`：活动页集合；
     - `inactive_list`：不活动页集合；
     - `nr_scan_active` 和 `nr_scan_inactive`：指定在回收内存时需要扫描的活动和不活动页的数目；
     - `pages_scanned`：指定上次换出一页以来，有多少页未能成功扫描；
     - `flags`：描述内存域的当前状态；可能的标志：`typedef enum {...} zone_flags_t`，也可能这些标志均未设置；测试和设置内存域标志的函数：`zone_set_flag, zone_test_and_set_flag, zone_clear_flag`；
     - `vm_stat`：维护大量有关内存域的统计信息，函数 `zone_page_stat` 用来读取 `vm_stat` 中的信息；
     - `prev_priority`：存储上一次扫描操作扫描该内存域的优先级，`try_to_free_pages()` 函数进行扫描，直至释放足够的页帧，扫描会根据该值判断是否换出映射的页；
     - `wait_table`, `wait_table_bits` 和 `wait_table_hash_nr_entries`：实现一个等待队列，可供等待某一页变为可用的进程使用。进程排成一队列，等待某些条件，在条件变为真时，内存会通知进程恢复工作。
     - `zone_pgdat`：内存域与父节点之间的关联由该成员建立；
     - `zone_start_pfn`：内存域第一个页帧的索引；
     - 剩余 3 个字段很少使用，因此置于数据结构末尾：`name`（3 个选项可用：`Normal`, `DMA`, `HighMem`）、`spanned_pages`（内存域中页的总数，并非都可用，因为空洞）、`present_pages`（实际可用的页数目）。
   - 在计算各种水印之前，内核首先确定需要为关键性分配保留的内存空间最小值，该值随可用内存的大小而非线性增长，并保存在全局变量 `min_free_kbytes` 中。一个不变的约束是，不能少于 128 KiB，也不能多于 64 MiB。用户层可通过文件 `/proc/sys/vm/min_free_kbytes` 来读取和修改该设置。
     - [ ] 表，主内存大小与可用于关键性分配的内存空间最小值之间的关系
     - 数据结构中水印值的填充由 `init_per_zone_pages_min` 处理，该函数由内核在启动期间调用；
     - `setup_per_zone_pages_min` 设置 `struct zone` 的 `pages_min`、`pages_low`、`pages_high` 成员；在计算出高端内存域之外页面的总数之后（保存在 `lowmem_pages`），内核迭代系统中的所有内存域并执行此函数；
     - `lowmem_reserve` 的计算由 `setup_per_zone_lowmem_reserve` 完成。内核迭代系统的所有结点，对每个结点的各个内存域分别计算预留内存最小值。
   - `struct zone` 的 `pageset` 成员用于实现冷热分配器（hot-n-cold allocator）。
     - 在多处理器系统上每个 CPU 都有一个或多个高速缓存，各个 CPU 的管理必须是独立的；
     - 尽管内存域可能属于一个特定的 NUMA 结点，因而关联到某个特定的 CPU，但其他 CPU 的高速缓存仍然可以包含该内存域中的页。最终的效果是，每个处理器都可以访问系统中所有的页，尽管速度不同。因此，特定于内存域的数据结构不仅要考虑到所属 NUMA 结点相关的 CPU，还必须照顾到系统中其他的 CPU；
     - `NR_CPUS` 并不是系统中实际存在的 CPU 数目，而是内核支持的 CPU 的最大数目；
    ```c
    <mmzone.h>
    struct per_cpu_pageset {
      struct per_cpu_pages pcp[2];  // 索引 0 对应热页，索引 1 对应冷页
    } __cacheline_aligned_in_smp;

    struct per_cpu_pages {
      int count;            // 列表中页数
      int high;             // 页数上限水印
      int batch;            // 添加/删除多页块的时候，块的大小
      struct list_head list;  // 页的链表
    };
    ```
    - `count` 记录了与该列表相关的页的数目，`high` 是一个水印，如果 `count` 的值超出了 `high`，则表明列表中的页太多了。对容量过低的状态没有显式使用水印；
    - `list` 保存了当前 CPU 的冷页或热页；
    - `batch`，如有可能，CPU 的高速缓存不是用单个页来填充的，而是用多个页组成的块，`batch` 是每次添加页数的一个参数值；
   - 页帧代表系统内存的最小单位，对内存中的每个页都会创建 `struct page` 的一个实例。
     - 内核程序员需要注意保持该结构尽可能小，因为即使在中等程度的内存配置下，系统的内存同样会分解为大量的页。IA-32，页大小-4 KiB，在主内存大小为 384 MiB，大约 100 000 页；
     - 内核管理的许多部分都使用页，但是它们依赖 `struct page` 提供的信息可能完全没有交集，为保持结构长度，C 语言的联合很适合于该问题。
    ```c
    <mm.h>
    struct page {
      unsigned long flags;    // 原子标志，有些情况下会异步更新
      atomic_t _count;        // 使用计数
      union {
        atomic_t _mapcount;   // 内存管理子系统中映射的页表项计数
                              // 用于表示页是否已经映射，还用于限制逆向映射搜索
        unsigned int inuse;   // 用于 SLUB 分配器：对象的数目
      };
      union {
        struct {
          unsigned long private;  // 由映射私有，不透明数据；
                                  // 如果设置了 PagePrivate，通用用于 buffer_heads；
                                  // 如果设置了 PageSwapCache，则用于 swp_entry_t；
                                  // 如果设置了 PG_buddy，则用于表示伙伴系统中的阶
          struct address_space *mapping;  // 如果最低位为 0，则指向 inode
                                          // address_space，或为 NULL。
                                          // 如果页映射为匿名内存，最低位置位，
                                          // 而且该指针指向 anon_vma 对象：
                                          // 参见下文的 PAGE_MAPPING_ANON。
        };
        ...
        struct kmem_cache *slab;  // 用于 SLUB 分配器，指向 slab 的指针
        struct page *first_page;  // 用于复合页的尾页，指向首页
      };
      union {
        pgoff_t index;  // 在映射内的偏移量
        void *freelist; // SLUB: freelist req. slab lock
      };
      struct list_head lru; // 换出页列表，例如由 zone->lru_lock 保护的 active_list
    
    #if defined(WANT_PAGE_VIRTUAL)
      void *virtual;  // 内核虚拟地址（如果没有映射则为 NULL，即高端内存）
    #endif
    };
    ```
    - `slab, freelist, inuse` 成员用于 slub 分配器，如果内核编译没有启用 slub 分配器支持，则不会使用这些成员；
    - 该结构的格式是体系无关的，不依赖于使用的 CPU 类型，每个页帧都由该结构描述；
    - `flags`：存储体系结构无关的标志，用于描述页的属性；
    - `_count`：使用计数，表示内核中引用该页的次数；在其值到达 0 时，内核就知道 page 实例当前不使用，因此可以删除；如果其值大于 0，该实例决不会从内存删除；
    - `_mapcount`：表示在页表中有多少页指向该页；
    - `lru`：是一个表头，用于在各种链表上维护该页，以便将页按不同类别分组，最重要的类别是活动和不活动页；
    - ...

### 页表
内核内存管理总是假定使用四级页表，而不管底层处理器是否如此。

页表管理分为两个部分，第一部分依赖于体系结构，第二部分是体系结构无关的。有趣的是，所有数据结构和操作数据结构的几乎所有函数都是定义在特定于体系结构的文件中。特定于不同 CPU 的实现有一些比较大的差别（因为使用了各种不同的 CPU 概念）。

涉及的头文件：
- `include/asm-arch/page.h`
- `include/asm-arch/pgtable.h`

内核源代码假定 `void *` 和 `unsigned long` 类型所需的比特位数相同，因此它们之间可以进行强制转换而不损失信息。该假定的形式表示为 `sizeof(void *) == sizeof(unsigned long)`，在 Linux 支持的所有体系结构上都是正确的。

内存地址分解
- 四级页表，5 个部分（4 个表项用于选择页 + 1 个索引表示页内位置）
- 各个体系结构不仅地址字长不同，而且地址字拆分的方式也不同
- `BITS_PER_LONG` 定义用于 `unsigned long` 变量的比特位数目
- [ ] 图，分解虚拟内存地址
  - `PAGE_SHIFT`
  - `PMD_SHIFT`
  - `PUD_SHIFT`
  - `PGDIR_SHIFT`
- 在各级页目录/页表中所能存储的指针数目，也可以通过宏定义确定
  - `PTRS_PER_PGD`: ...
  - `PTRS_PER_PMD`: ...
  - `PTRS_PER_PUD`: 上层页目录中项的数目；
  - `PTRS_PER_PTE`: 页表中项的数目；
- 两级页表的体系结构会将 `PTRS_PER_PMD` 和 `PTRS_PER_PUD` 定义为 1。由于只有极少数系统使用四级页表，内核使用头文件 `include/asm-generic/pgtable-nopud.h` 来提供模拟上层页目录所需的所有声明。`include/asm-generic/pgtable-nopmd.h` 同理。
- 大小宏定义：
  ```c
  #define PAGE_SIZE (1UL << PAGE_SHIFT)
  #define PUD_SIZE  (1UL << PUD_SHIFT)
  #define PMD_SIZE  (1UL << PMD_SHIFT)
  #define PGDIR_SIZE  (1UL << PGDIR_SHIFT)
  ```
- 从给定地址中提取各个分量：
  ```c
  #define PAGE_MASK (~(PAGE_SIZE - 1))
  #define PUD_MASK  (~(PUD_SIZE - 1))
  #define PMD_MASK  (~(PMD_SIZE - 1))
  #define PGDIR_MASK  (~(PGDIR_SIZE - 1))
  ```

页表格式
- 内核提供 4 个数据结构（定义在 `page.h` 中）来表示页表项的结构
  - `pgd_t`
  - `pud_t`
  - `pmd_t`：用于中间页目录项
  - `pte_t`：用于直接页表项
- [ ] 表，用于分析页表项的函数
- `PAGE_ALIGN` 是另一个每种体系结构都必须定义的标准宏（通常在 `page.h` 中）。为用好处理器的高速缓存资源，将地址对齐到页边界是很重要的。
- 尽管使用了 C 结构来表示页表项，但大多数页表项都只有一个成员，通常是 `unsigned long` 类型（使用 `struct` 而不是基本类型，以确保页表项的内容只能由相关的辅助函数处理，而决不能直接访问。另外，也便于扩展。）：
  ```c
  // 以 AMD64 体系结构为例
  // include/asm-x86_64/page.h
  typedef struct { unsigned long pte; } pte_t;
  typedef struct { unsigned long pmd; } pmd_t;
  typedef struct { unsigned long pud; } pud_t;
  typedef struct { unsigned long pgd; } pgd_t;
  ```
- 内核（以及处理器）使用 32 位或 64 位类型来表示页表项（不管页表的级数）。这意味着并非表项的所有比特位都存储了有用的数据，即下一级表的基地址。多余的比特位用于保存额外的信息。

特定于 PTE 的信息
- 尽管这些数据是特定于 CPU 的，它们至少提供了有关页访问控制的一些信息。
  - `_PAGE_PRESENT`
  - `_PAGE_ACCESSED`
  - `_PAGE_DIRTY`
  - `_PAGE_FILE`
  - `_PAGE_USER`
  - `_PAGE_READ`
  - `_PAGE_WRITE`
  - `_PAGE_EXECUTE`
- 每种体系结构都必须提供两个东西，使得内存管理子系统能够修改 `pte_t` 项中额外的比特位，即保存额外的比特位的 `__pgprot` 数据类型，以及修改这些比特位的 `pte_modify` 函数。上述的预处理器符号可用于选择适当的比特位。
- 用于查询和设置内存页与体系结构相关的状态（并非所有处理器都定义了所有这些函数）
  - `pte_present`
  - `pte_dirty`
  - `pte_write`
  - `pte_file`：用于非线性映射，通过操作页表提供了文件内容的一种不同视图。该函数检查页表项是否属于这样的一个映射。只有在 `pte_present` 返回 false 时，才能调用 `pte_file`，即与该页表项相关的页不在内存中。由于内核的通用代码对 `pte_file` 的依赖，在某个体系结构并不支持非线性映射的情况下也需要定义该函数。在这种情况下，该函数总是返回 0。
  - [ ] 表，用于处理内存页的体系结构相关状态的函数

页表项的创建和操作
- [ ] 表，用于创建新页表项的函数
  - 所有体系结构都必须实现表中的函数，以便内存管理代码创建和销毁页表。

### 初始化内存管理
因为内存管理初始化中特定于 CPU 的部分使用了底层体系结构许多次要、微妙的细节，这些与内核的结构没什么关系，最多不过是汇编语言程序设计的最佳实践而已，因此这里只是从一个较高的层次来考虑初始化相关的工作。关键是 `pg_data_t` 数据结构的初始化（及其下级结构），其内容已经是与机器无关的了。

忽略前述的特定于处理器的操作，这些操作的主要意图在于确认系统中内存的总数量，及其在各个结点和内存域之间的分配情况。

建立数据结构
- 对相关数据结构的初始化是从全局启动例程 `start_kernel` 中开始的，该例程在加载内核并激活各个子系统之后执行；
- 由于内存管理是内核一个非常重要的部分，因此在特定于体系结构的设置步骤中检测内存并确定系统中内存的分配情况后，会立即执行内存管理的初始化。此时，已经对各种系统内存模式生成了一个 `pgdata_t` 实例，用于保存诸如结点中内存数量以及内存在各个内存域之间分配情况的信息。
- 所有平台上都实现了特定于体系结构的 `NODE_DATA` 宏，用于通过结点编号，来查询一个 NUMA 结点相关的 `pgdata_t` 实例。

先决条件
- 只考察只有一个内存结点的系统。
- 为确保内存管理代码可移植，内核在 `mm/page_alloc.c` 中定义了一个 `pg_data_t` 实例（称作 `contig_page_data`）管理所有的系统内存。这不是特定于 CPU 的实现。大多数体系结构都采用了该方案。`NODE_DATA` 的实现：
  ```c
  <mmzone.h>
  #define NODE_DATA(nid)  (&contig_page_data)
  ```
- 内核也可以依赖于下述事实：体系结构相关的初始化代码将 `numnodes` 变量设置为系统中结点的数目。在 UMA 系统上因为只有一个（形式上的）结点，因此该数量是 1。

系统启动
- [ ] 图，从内存管理视角来看内核初始化
- `setup_arch`，特定于体系结构的设置函数，其中一项任务是负责初始化自举分配器；
- `setup_per_cpu_areas`（在 SMP 系统上）初始化源代码中（使用 `per_cpu` 宏）定义的静态 `per-cpu` 变量，这种变量对系统中的每个 CPU 都有一个独立的副本。此类变量保存在内核二进制映像的一个独立段中。在非 SMP 系统上该函数是一个空操作；
- `build_all_zonelist`，建立结点和内存域的数据结构；
- `mem_init`，特定于体系结构的函数，用于停用 `bootmem` 分配器并迁移到实际的内存管理函数；
- `kmem_cache_init`，初始化内核内部用于小块内存区的分配器；
- `setup_per_cpu_pageset`，为 `pageset` 数组的第一个数组元素分配内存，意味着为第一个系统处理器分配。系统的所有内存域都会考虑进来。
- 该函数还负责设置冷热分配器的限制。在 SMP 系统上对应于其他 CPU 的 pageset 数组成员，将会在相应的 CPU 激活时初始化。

结点和内存域初始化
- `build_all_zonelist` 建立管理结点及其内存域所需的数据结构
 ```txt
 build_all_zonelists
  __build_all_zonelists
    build_zonelists(pgdat)    // 对系统上的每个 online 结点
 ```
- `build_zonelists` 的任务是，在当前处理的结点和系统中其他结点的内存域之间建立一种等级次序。接下来依据这种次序分配内存。如果在期望的结点内存域中，没有空闲内存，那么这种次序就很重要。
  - 内核定义了内存的一个层次结构，首先试图分配“廉价的”内存。如果失败，则根据访问速度和容量，逐渐尝试分配“更昂贵的”内存；
  - 高端内存是最廉价，因为内核没有任何部分依赖于该内存域分配的内存，如果高端内存域用尽，对内核没有任何副作用；接下来是普通内存域，许多内核数据结构都必须保存在该内存域，只要高端内存域的内存没有用尽，都不会从普通内存域分配内存；最昂贵的是 DMA 内存域，因为它用于外设和系统之间的数据传输；
  - 备选结点应该尽可能靠近主结点，以最小化由于访问非本地内存引起的性能损失；当前结点所有内存域的内存都用尽时，确定一个备选结点；
  - 内核使用 `pg_data_t` 中的 `zonelist` 数组来表示所描述的层次结构。它包括所有结点的所有内存域。
  - [ ] 图，连续填充备用列表
  - 对总数 N 个结点中的结点 m 来说，内核生成备用列表时，选择备用结点的顺序总是：m、m + 1、m + 2、...、N - 1、0、1、...、m - 1，这确保了不过度使用任何结点。
  - [ ] 图，完成的备用列表

### 物理内存管理
在内核初始化完成后，内存管理的责任由伙伴系统承担。伙伴系统基于一种相对简单然而令人吃惊的强大算法，已经伴随我们几乎 40 年。它结合了优秀内存分配器的两个关键特征：速度和效率。

1. 伙伴系统的结构
   - 系统内存中的每个物理内存页（页帧），都对应于一个 `struct page` 实例。每个内存域都关联了一个 `struct zone` 的实例，其中保存了用于管理伙伴数据的主要数组：
    ```c
    <mmzone.h>
    struct zone {
      ...
      /* 不同长度的空闲区域 */
      struct free_area free_area[MAX_ORDER];
      ...
    };

    // nr_free 统计了所有列表上空闲页的数目
    // 每种迁移类型都对应于一个空闲列表
    struct free_area {
      struct list_head free_list[MIGRATE_TYPES];
      unsigned long nr_free;
    };

    // order 的范围从 0 到 MAX_ORDER
    #ifndef CONFIG_FORCE_MAX_ZONEORDER
    #define MAX_ORDER 11
    #else
    #define MAX_ORDER CONFIG_FORCE_MAX_ZONEORDER
    #endif
    #define MAX_ORDER_NR_PAGES (1 << (MAX_ORDER - 1))
    ```
    - [ ] `MAX_ORDER_NR_PAGES` 与 `MAX_ORDER` 定义的大小不匹配？两者之间有 2 倍的差距？
    - 宏 `for_each_migratetype_order(order, type)` 可用于迭代指定迁移类型的所有分配阶。
   - [ ] 伙伴系统中相关连接的内存区
     - 内存区中第一页内的链表元素，可用于将内存区维持在链表中
   - 伙伴不必是彼此连接的。如果一个内存区在分配期间分解为两半，内核会自动将未用的一半加入到对应的链表中。如果在未来的某个时刻，由于内存释放的缘故，两个内存区都处于空闲状态，可通过其地址判断是否为伙伴系统。管理工作较少，是伙伴系统的一个主要优点。
   - 基于伙伴系统的内存管理专注于某个结点的某个内存域。但所有内存域和结点的伙伴系统都通过备用分配列表连接起来。在首选的内存域无法满足内存分配请求时，首先尝试同一个结点的另一个内存域，接下来再尝试另一个结点，直至满足要求。
     - [ ] 伙伴系统和内存域/结点之间的关系
   - 伙伴系统当前状态信息可以在 `/proc/buddyinfo` 中获得：
    ```bash
    cat /proc/buddyinfo
    # 输出，阶数从左至右依次升高
    Node 0, zone DMA  3 5 7 4 6 3 3 3 1 1 1
    Node 0, zone DMA32  ...
    Node 0, zone Normal ...
    ```
2. 避免碎片
   - 依据可移动性组织页：对用户空间应用程序没有问题，因为内存是通过页表映射的，无论空闲页在物理内存中的分布如何，应用程序看到的内存似乎总是连续的。但对内核来说，碎片是一个问题。（大多数）物理内存一致映射到地址空间的内核部分。
   - 提到的内存碎片只涉及内核，这只是部分正确的。CPU 提供使用巨型页的可能性（比普通页大得多、对内存使用密集的应用程序有好处，TLB 只需处理较少的项，降低 TLB 缓存失效的可能性），但分配巨型页需要连续的空闲物理内存。
   - 在内核 2.6.24 开发期间，防止碎片的方法最终加入内核。
   - 文件系统也有碎片，该领域的碎片问题主要通过碎片合并工具解决。它们分析文件系统，重新排序已分配存储块，从而建立较大的连续存储区。但由于许多物理内存页不能移动到任意位置，阻碍了该方法的实施。
   - 内核的方法是反碎片（anti-fragmentation），即试图从最开始尽可能防止碎片。
   - 内核将已分配页划分为 3 种不同类型：
     - 不可移动页：在内存中有固定位置，不能移动到其他地方。核心内核分配的大多数内存属于该类别。
     - 可回收页：不能直接移动，但可以删除。其内容可以从某些源重新生成。例如，映射自文件的数据属于该类别。`kswapd` 守护进程会根据可回收页访问的频繁程度，周期性释放此类内存。
     - 可移动页：可以随意移动。用户空间应用程序的页属于该类别，它们是通过页表映射的，如果它们复制到新位置，页表项可以相应地更新，应用程序不会注意到任何事。
   - 内核使用的反碎片技术，即基于将具有相同可移动性的页分组的思想。例如，不可移动的页不能位于可移动内存区的中间，否则就无法从该内存区分配较大的连续内存块。
   - 从最初开始，内存并未划分可移动性不同的区，这些是在运行时形成的。内核的另一种方法确实将内存分区，分别用于可移动页和不可移动页的分配。
   - 尽管内核使用的反碎片技术卓有成效，它对伙伴分配器的代码和数据结构几乎没有影响。
   - 迁移类型定义：
    ```c
    <mmzone.h>
    #define MIGRATE_UNMOVABLE 0
    #define MIGRATE_RECLAIMABLE 1
    #define MIGRATE_MOVABLE 2
    #define MIGRATE_RESERVE 3
    #define MIGRATE_ISOLATE 4 // 不能从这里分配
    #define MIGRATE_TYPES 5
    ```
    - `MIGRATE_RESERVE`：如果向具有特定可移动性的列表请求分配内存失败，这种紧急情况下可以从 `MIGRATE_RESERVE` 分配内存（对应的列表在内存子系统初始化期间用 `setup_zone_migrate_reserve` 填充）；
    - `MIGRATE_ISOLATE`：用于跨域 NUMA 结点移动物理内存页。在大型系统上，它有益于将物理内存移动到接近于使用该页最频繁的 CPU。
    - `MIGRATE_TYPES` 只是表示迁移类型的数目，不代表具体区域。
   - 内核规定了在指定列表中无法满足分配请求时，接下来应使用哪一种迁移类型：
    ```c
    // mm/page_alloc.c
    static int fallbacks[MIGRATE_TYPES][MIGRATE_TYPES - 1] = {
      [MIGRATE_UNMOVABLE] = {MIGRATE_RECLAIMABLE, MIGRATE_MOVABLE, MIGRATE_RESERVE},
      [MIGRATE_RELAIMABLE] = {MIGRATE_UNMOVABLE, MIGRATE_MOVABLE, MIGRATE_RESERVE},
      [MIGRATE_MOVABLE] = {MIGRATE_RECLAIMABLE, MIGRATE_UNMOVABLE, MIGRATE_RESERVE},
      [MIGRATE_RESERVE] = {MIGRATE_RESERVE, MIGRATE_RESERVE, MIGRATE_RESERVE},  // 从来不用
    };
    ```
   - 全局变量：`pageblock_order`、`pageblock_nr_pages`，前者表示内核认为是“大”的一个分配阶，后者表示该分配阶对应的页数。如果体系结构定义提供了巨型页机制，则 `pageblock_order` 通常定义为巨型页对应的分配阶；如果体系结构不支持巨型页，则将其定义为第二高的分配阶：
    ```c
    <pageblock-flags.h>
    // 支持巨型页
    #define pageblock_order HUGETLB_PAGE_ORDER
    // 不支持
    #define pageblock_order (MAX_ORDER - 1)
    ```
   - 如果各迁移类型的链表中没有一块较大的连续内存，那么页面迁移不会提供任何好处，因此在可用内存太少时内核会关闭该特性。`build_all_zonelists()` 函数完成检查（实际检查的是链表中页的数目），如果没有足够的内存可用，则全局变量 `page_group_by_mobility` 设置为 0，反之设置为 1。
   - 有关各个内存分配的细节都通过分配掩码指定。内核提供了两个标志，分别用于表示分配的内存是可移动的（`__GFP_MOVABLE`）或可回收的（`__GFP_RECLAIMABLE`）。如果这些标志都没有设置，则分配的内存假定为不可移动的。如果停用了页面迁移特性，则所有的页都是不可移动的。
    ```c
    <gfp.h>
    // 转换分配标志及对应的迁移类型
    static inline int allocflags_to_migratetype(gfp_t gfp_flags) {
      ...
    }
    ```
   - 每个内存域都提供了一个特殊的字段，可以跟踪包含 `pageblock_nr_pages` 个页的内存区的属性：
    ```c
    <mmzone.h>
    struct zone {
      ...
      unsigned long *pageblock_flags;
      ...
    }
    ```
   - 在初始化期间，内核自动确保对内存域中的每个不同的迁移类型分组，在 `pageblock_flags` 中都分配了足够存储 `NR_PAGEBLOCK_BITS` 个比特位的空间。当前，表示一个连续内存区的迁移类型需要 3 个比特位。`set_packblock_migratetype(struct page *page, int migratetype)` 负责设置以 page 为首的一个内存区的迁移类型。`migratetye` 参数可以通过 `allocflags_to_migratetype` 辅助哈数构建。
   - 页的迁移类型是预先分配好的，对应的比特位总是可用，与页是否由伙伴系统管理无关。在释放内存时，页必须返回到正确的迁移链表。
   - 当前页面的分配状态可以从 `/proc/pagetypeinfo` 获得。
   - 在内存子系统初始化期间，`memmap_init_zone` 负责处理内存域的 page 实例。该函数会将所有的页最初都标记为可移动的。
   - 在分配内存时，如果必须“盗取”不同于预定迁移类型的内存区，内核在策略上倾向于“盗取”更大的内存区。由于所有页最初都是可移动的，那么在内核分配不可移动的内存区时，则必须“盗取”。实际上，在启动期间分配可移动内存区的情况较少，那么分配器有很高的几率分配长度最大的内存区，并将其从可移动列表转换到不可移动列表。由于分配的内存区长度是最大的，因此不会向可移动内存中引入碎片。
   - 依据可移动性组织页是防止内存碎片的一种可能方法，内核还提供了另一种阻止该问题的手段：虚拟内存域 `ZONE_MOVABLE`。与可移动性分组相反，`ZONE_MOVABLE` 特性必须由管理员显式激活。
   - 基本思想：可用的物理内存划分为两个内存域：一个用于可移动分配，一个用于不可移动分配。这会自动防止不可移动页向可移动内存域引入碎片。
   - `kernelcore` 参数用来指定用于不可移动分配的内存数量，即用于即不能回收也不能迁移的内存数量。剩余的内存用于可移动分配。在分析该参数后，结果保存在全局变量 `required_kernelcore` 中。还可以使用参数 `movablecore` 控制用于可移动内存分配的内存数量。`required_kernelcore` 的大小将会据此计算。如果同时指定两个参数，内核取两种方式计算得到的 `required_kernelcore` 中较大的一个。
   - 取决于体系结构和内核配置，`ZONE_MOVABLE` 内存域可能位于高端或普通内存域：
    ```c
    <mmzone.h>
    enum zone_type {
      ...
      ZONE_NORMAL,
    #ifdef CONFIG_HIGHMEM
      ZONE_HIGHMEM,
    #endif
      ZONE_MOVABLE,
      MAX_NR_ZONES
    };
    ```
   - 与系统中所有其他的内存域相反，`ZONE_MOVABLE` 并不关联到任何硬件上有意义的内存范围。该内存域中的内存取自高端内存或普通内存域，称 `ZONE_MOVABLE` 是一个虚拟内存域。
   - `find_zone_movable_pfns_for_nodes` 用于计算进入 `ZONE_MOVABLE` 的内存数量。如果 `kernelcore` 和 `movablecore` 参数都没有指定，`find_zone_movable_pfns_for_nodes` 会使 `ZONE_MOVABLE` 保持为空，该机制处于无效状态。
   - 用于为虚拟内存域 `ZONE_MOVABLE` 提取内存页的物理内存域，保存在全局变量 `movable_zone` 中；对每个结点来说，`zone_movable_pfn[node_id]` 表示 `ZONE_MOVABLE` 在 `movable_zone` 内存域中所取得内存的起始地址。
   - 所有可移动分配都必须指定 `__GFP_HIGHMEM` 和 `__GFP_MOVABLE`。
3. 初始化内存域和结点数据结构
   - 体系结构相关代码在启动期间建立以下信息：系统中各个内存域的页帧边界，保存在 `max_zone_pfn` 数组；各个结点页帧的分配情况，保存在全局变量 `early_node_map` 中。
     - [ ] 图，在建立结点和内存域内存管理数据结构时...
   - `free_area_init_nodes`：
     - 对照在 `zone_max_pfn` 和 `zone_min_pfn` 中指定的内存域的边界，计算各个内存域可使用的最低和最高的页帧编号。使用了两个全局数组来存储这些信息：
      ```c
      // mm/page_alloc.c
      static unsigned long __meminitdata arch_zone_lowest_possibile_pfn[MAX_NR_ZONES];
      static unsigned long __meminitdata arch_zone_highest_possible_pfn[MAX_NR_ZONES];
      ```
4. 分配器 API
   - 就伙伴系统的接口而言，NUMA 和 UMA 体系结构是没有差别的，二者的调用语法都是相同的。所有函数的一个共同点是：只能分配 2 的整数幂个页。内核中细粒度的分配只能借助于 slab 分配器（或者 slub、slob 分配器），后者基于伙伴系统。
   - 接口：
     - `alloc_pages(mask, order)` 分配 `2^order` 页并返回一个 `struct page` 的实例，表示分配的内存块的起始页。`alloc_page(mask)` 是前者在 `order = 0` 情况下的简化形式，只分配一页。
     - `get_zeroed_page(mask)` 分配一页并返回一个 page 实例，页对应的内存填充 0（所有其他函数，分配之后页的内容是未定义的）。
     - `__get_free_pages(mask, order)` 和 `__get_free_page(mask)` 的工作方式与上述函数相同，但返回分配内存块的虚拟地址，而不是 page 实例。
     - `get_dma_pages(gfp_mask, order)` 用来获得适用于 DMA 的页。
   - 在空闲内存无法满足请求以至于分配失败的情况下，所有上述函数都返回空指针（`alloc_pages` 和 `alloc_page`）或者 0（`get_zeroed_page`、`__get_free_pages` 和 `__get_free_page`）。
   - 内核除了伙伴系统函数之外，还提供了其他内存管理函数。它们以伙伴系统为基础，但并不属于伙伴分配器自身。
     - `vmalloc` 和 `vmalloc_32`，使用页表将不连续的内存映射到内核地址空间中，使之看上去是连续的。
     - `kmalloc`，用于分配小于一整页的内存区。
   - 释放不再使用的页：
     - `free_page(struct page *)` 和 `free_pages(struct page *, order)`，用于将页返回给内存管理子系统。
     - `__free_page(dir)` 和 `__free_pages(addr, order)`，使用虚拟内存地址而不是 page 实例。
   - 分配掩码
     - 内核提供了内存域修饰符（zone modifier）（在掩码的最低 4 个比特位定义），来指定从哪个内存域分配所需的页。
      ```c
      <gfp.h>
      /* GFP_ZONEMASK 中的内存域修饰符（参见 linux/mmzone.h，低 3 位）*/
      #define __GFP_DMA     ((__force gfp_t)0x01u)
      #define __GFP_HIGHMEM ((__force gfp_t)0x20u)
      #define __GFP_DMA32   ((__force gfp_t)0x40u)
      ...
      #define __GFP_MOVABLE ((__force gfp_t)0x100000u)  // 页是可移动的
      ```
      - GFP，get free page
      - `__GFP_MOVABLE` 不表示物理内存域，但通知内核应该在特殊的虚拟内存域 `ZONE_MOVABLE` 进行相应的分配。
      - 没有 `__GFP_NORMAL` 常数，而内存分配的主要负担却落到 `ZONE_NORMAL` 内存域。内核考虑到这一点，提供了一个函数来计算与给定分配标志兼容的最高内存域。那么内存分配可以从该内存域或更低的内存域进行：
        ```c
        // mm/page_alloc.c
        static inline enum zone_type gfp_zone(gfp_t flags) {
          ...
        }
        ```
       - [ ] 表，内存域修饰符和扫描的内存域之间的关联
       - 设置 `__GFP_MOVABLE` 不会影响内核的决策，除非它与 `__GFP_HIGHMEM` 同时指定。在这种情况下，会使用特殊的虚拟内存域 `ZONE_MOVABLE` 满足内存分配请求。
       - [ ] 图，GFP 掩码的布局
       - 与内存域修饰符相反，这些额外的标志并不限制从哪个物理内存段分配内存，但确实可以改变分配器的行为：
        ```c
        <gfp.h>
        #define __GFP_WAIT      ((__force gfp_t)0x10u)    // 可以等待和重调度？
        ...

        // 由于这些标志几乎总是组合使用，内核作了一些分组，包含了用于
        // 各种标准情形的适当的标志。如果有可能的话，在内存管理子系统
        // 外，总是把下列分组之一用于内存分配。
        // 在内核源代码中，双下划线通常用于内部数据和定义
        <gfp.h>
        #define GFP_ATOMIC      (__GFP_HIGH)
        #define GFP_NOIO        (__GFP_WAIT)
        #define GFP_NOFS        (__GFP_WAIT | __GFP_IO)
        #define GFP_KERNEL      (__GFP_WAIT | __GFP_IO | __GFP_FS)
        #define GFP_USER        (__GFP_WAIT | __GFP_IO | __GFP_FS | __GFP_HARDWALL)
        #define GFP_HIGHUSER    (__GFP_WAIT | __GFP_IO | __GFP_FS | __GFP_HARDWALL | __GFP_HIGHMEM)
        #define GFP_HIGHUSER_MOVABLE  (__GFP_WAIT | __GFP_IO | __GFP_FS | __GFP_HARDWALL | __GFP_HIGHMEM | __GFP_MOVABLE)
        #define GFP_DMA         __GFP_DMA
        #define GFP_DMA32       __GFP_DMA32
        ```
     - 内存分配宏
       - 所有分配接口函数都可以追溯到一个简单的基本函数：`alloc_pages`，它会进一步调用 `alloc_pages_node`；
        ```c
        <gfp.h>
        #define alloc_page(gfp_mask)  alloc_pages(gfp_mask, 0)
        ...
        #define __get_free_page(gfp_mask) __get_free_pages((gfp_mask), 0)
        
        <mm.h>
        #define __get_dma_pages(gfp_mask, order) __get_free_pages((gfp_mask) | GFP_DMA, (order))

        // get_zeroed_page，对 alloc_pages 使用 __GFP_ZERO 标志，
        // 即可分配填充字节 0 的页，再返回与页关联的内存区地址即可。
        // 所有体系结构都必须实现的标准函数 clear_page，可以帮助
        // alloc_page 对页填充字节 0。也可以用通用的、处理器无关的
        // 代码对页填充 0，但大多数 CPU 都提供了特殊的命令，可以更
        // 快地完成该操作。

        <gfp.h>
        #define alloc_pages(gfp_mask, order) alloc_pages_node(numa_node_id(), gfp_mask, order)

        // mm/page_alloc.c
        fastcall unsigned long __get_free_pages(gfp_t gfp_mask, unsigned int order) {
          struct page *page;
          page = alloc_pages(gfp_mask, order);
          if (!page)
            return 0;
          return (unsigned long) page_address(page);
        }

        // page_cache_alloc 和 page_cache_alloc_cold 也是两个便捷
        // 函数，可根据 __GFP_COLD 修饰符的设置，分别获取热页和冷页
        ```
       - [ ] 伙伴系统的各个分配函数之间的关系
       - 类似地，内存释放函数也可以归纳到一个主要函数：`__free_pages`。
       - [ ] 伙伴系统的各个内存释放函数之间的关系
5. 分配页
   - 所有 API 函数都追溯到 `alloc_pages_node`：
    ```c
    <gfp.h>
    static inline struct page *alloc_pages_node(int nid, gfp_t gfp_mask, unsigned int order) {
      if (unlikely(order >= MAX_ORDER))
        return NULL;
      /* 未知节点即当前结点 */
      if (nid < 0)
        nid = numa_node_id();
      return __alloc_pages(gfp_mask, order, NODE_DATA(nide)->node_zonelists + gfp_zone(gfp_mask));
    }
    ```
   - 内核源代码将 `__alloc_pages` 称之为“伙伴系统的心脏”，因为它处理的是实质性的内存分配。
   - 选择页
    ```c
    // flags-用于控制到达各个水印指定的临界状态时的行为
    // mm/page_alloc.c
    #define ALLOC_NO_WATERMARKS   0x01    // 完全不检查水印
    #define ALLOC_WMARK_MIN       0x02    // 使用 pages_min 水印
    #define ALLOC_WMARK_LOW       0x04    // 使用 pages_low 水印
    #define ALLOC_WMARK_HIGH      0x08    // 使用 pages_high 水印
    #define ALLOC_HARDER          0x10    // 试图更努力地分配，及放宽限制
    #define ALLOC_HIGH            0x20    // 设置了 __GFP_HIGH
    #define ALLOC_CPUSET          0x40    // 检查内存结点是否对应着指定的 CPU 集合
    ```
    - 默认情况下，只有内存域包含页的数目至少为 `zone->pages_high` 时，才能分配页。这对应于 `ALLOC_WMARK_HIGH` 标志。
    - `ALLOC_HARDER` 通知伙伴系统再急需内存时放宽分配规则。在分配高端内存域的内存时，`ALLOC_HIGH` 进一步放宽限制。
    - `ALLOC_CPUSET` 告知内核，内存只能从当前进程允许运行的 CPU 相关联的内存结点分配。只对 NUMA 系统有意义。
    - 设置的标志在 `zone_watermark_ok` 函数中检查，该函数根据设置的标志判断是否能从给定的内存域分配内存：
      ```c
      // mm/page_alloc.c
      int zone_watermark_ok(struct zone *z, int order, unsigned long mark, int classzone_idx, int alloc_flags);

      // zone_per_state 用来访问每个内存域的统计量，在上述函数中，得到的是空闲页的数目。
      ```
    - `get_page_from_freelist` 是伙伴系统使用的另一个重要辅助函数。它通过标志集和分配阶来判断是否能进行分配。如果可以，则发起实际的分配操作。
      ```c
      // mm/page_alloc.c
      static struct page *get_page_from_freelist(gfp_t gfp_mask, unsigned int order, struct zonelist *zonelist, int alloc_flags);

      // 遍历 zonelist 指向的备用列表
      // cpuset_zone_allowed_softwall，检查给定内存域是否属于该进程允许运行的 CPU
      // zone_watermark_ok，检查内存域是否有足够的空闲页
      // 如果内存域适用于当前的分配请求，那么 buffered_rmqueue 试图从中分配所需数目的页
      ```
   - `buffered_rmqueue`，如果分配成功，则将页返回给调用者。
   - `__alloc_pages` 通过调用 `get_page_from_freelist` 实现页的分配。
     - 在最简单的情形中，分配空闲内存区只涉及调用一次 `get_page_from_freelist`，然后返回所需数目的页。第一次内存分配尝试不会特别积极，如果在某个内存域中无法找到空闲内存，则意味着内存没剩下多少了，内核需要增加较多的工作量才能找到更多内存。
     - 内核再次遍历备用列表中的所有内存域，每次都调用 `wakeup_kswapd`。该函数会唤醒负责换出页的 `kswapd` 守护进程。空闲内存可以通过缩减内核缓存和页面回收获得，即写回或换出很少使用的页，这两种措施都是由该守护进程发起的。在这之后开始新的尝试，这次搜索更为积极，对分配标志进行了调整，修改为一些在当前特定情况下更有可能分配成功的标志。同时，将水印降低到最小值。对实时进程和指定了 `__GFP_WAIT` 标志因而不能睡眠的调用，会设置 `ALLOC_HARDER`。然后用修改的标志集，再一次调用 `get_page_from_freelist`。
     - 如果再次失败，内核会借助于更强有力的措施。在进行一些标志判断后（`PF_MEMALLOC` 或 `TIF_MEMDIE`），再次调用 `get_page_from_freelist`，但这次会完全忽略水印。
       - 在忽略水印情况下，`get_page_from_freelist` 仍然失败了。在这种情况下，会放弃搜索并报告错误信息。如果设置了 `__GFP_NOFAIL`，内核会进入无限循环，直至成功。
       - 设置了 `__GFP_NOMEMALLOC`，该标志禁止使用紧急分配链表（如果忽略水印，这可能是最佳途径），因此无法在禁用水印的情况下调用 `get_page_from_freelist`。在这种情况下内核最终只能失败，通过内核消息将失败报告给用户，并将 NULL 指针返回给调用者。
     - 如果没有设置 `PF_MEMALLOC`，内核仍然还有一些选项可以尝试，但这些都需要睡眠。为使得 `kswapd` 取得一些进展，睡眠是必要的。从这里内核进入了一条低速路径（slow path），其中会开始一些耗时的操作。前提是分配掩码中设置了 `__GFP_WAIT` 标志，因为随后的操作可能使进程睡眠：
      ```c
      // mm/page_alloc.c
      /* 原子分配：我们无法进行“均衡” */
      if (!wait)
        goto nopage;
      cond_schedule();  // 防止花费过多时间搜索内存，以至于使其他进程处于饥饿状态
      ```
    - 分页机制提供了一个目前尚未使用的选项，将很少使用的页换出到块介质，以便在物理内存中产生更多空间。但该选项非常耗时，还可能导致进程睡眠。`try_to_free_pages` 是对应的辅助函数，用于查找当前不急需的页，以便换出。在该分配任务设置了 `PF_MEMALLOC` 标志后，会调用该函数，用于向其余的内核代码表明所有后续的内存分配都需要这样的搜索。
      ```c
      // mm/page_alloc.c
      /* 我们现在进入同步回收状态 */
      p->flags |= PF_MEMALLOC;
      ...
      did_some_progress = try_to_free_pages(zonelist->zones, order, gfp_mask);
      ...
      p->flags &= ~PF_MEMALLOC;

      cond_resched();
      ...
      ```
      - `try_to_free_pages` 自身可能也需要分配新的内存。由于为获得新内存还需要额外分配一点内存（相当矛盾的情形），该进程当然应该在内存管理方面享有最高优先级，上述标志的设置即达到了这一目的。
      - 此外，设置该标志确保了 `try_to_free_pages` 不会递归调用，因为如果此前设置了 `PF_MEMALLOC`，那么 `__alloc_pages` 肯定已经返回。
      - `try_to_free_pages` 选择最近不十分活跃的页，将其写到交换区，在物理内存中腾出空间。它返回增加的空闲页数目。
    - 如果需要分配多页，那么 per-CPU 缓存中的页也会拿回到伙伴系统：
      ```c
      mm/page_alloc.c
      if (order != 0)
        drain_all_local_pages();
      ```
    - 接下来，如果 `try_to_free_pages` 释放了一些页，那么内核再次调用 `get_page_from_freelist` 尝试分配内存。
    - 如果内核可能执行影响 VFS 层的调用而又没有设置 `GFP_NORETRY`，那么调用 `OOM (out of memory) kill`。
      ```c
      // mm/page_alloc.c
      /* OOM killer 无助于高阶分配，因此失败 */
      if (order > PAGE_ALLOC_COSTLY_ORDER) {
        clear_zonelist_oom(zonelist);
        goto nopage;
      }
      out_of_memory(zonelist, gfp_mask, order);
      goto restart;
      ```
      - `out_of_memory()` 函数选择一个内核认为犯有分配过多内存“罪行”的进程，并杀死该进程。这有很大概率腾出较多的空闲页，然后跳转到标号 restart，重试分配内存的操作。
      - 但杀死一个进程未必立即出现多于 `2^PAGE_COSTLY_ORDER` 页的连续内存（`PAGE_COSTLY_ORDER_PAGES` 通常设置为 3），因此如果当前要分配如此大的内存区，那么内核会饶恕所选择的进程，不执行杀死进程的任务，而是承认失败并跳转到 nopage。
    - 如果设置了 `__GFP_NORETRY`，或内核不允许使用可能影响 VFS 层的操作。在这种情况下，会判断所需分配的长度，作出不同的决定：
      - 如果分配长度小于 `2^PAGE_ALLOC_COSTLY_ORDER` 或设置了 `__GFP_REPEAT` 标志，则内核进入无限循环。内核会跳转回 rebalance 标号。
      - 在要求的分配大于 `PAGE_ALLOC_COSTLY_ORDER (3)` 但设置了 `__GFP_NOFAIL` 标志的情况下，内核也会进入上述无限循环。
      - 除此以外，内核只能放弃，并向用户返回 NULL 指针，并输出一条内存请求无法满足的警告消息。
   - 移除选择的页
     - 如果内核找到适当的内存域，具有足够的空闲页可供分配，那么还有两件事需要完成：
       - 首先，必须检查这些页是否是连续的（到目前为止，只知道有许多空闲页）；
       - 其次，必须按伙伴系统的方式从 `free_list` 移除这些页，这可能需要分解并重排内存区；
     - 内核将该工作委托给 `buffered_rmqueue()`：
       - 如果只分配一页，内核会进行优化。该页不是从伙伴系统直接取得，而是取自 per-CPU 的页缓存（该缓存提供了 CPU 本地的热页和冷页的列表）；

## 第四章-进程虚拟内存
- 用户层进程的虚拟地址空间是 Linux 的一个重要抽象：它向每个运行进程提供了同样的系统视图，这使得多个进程可以同时运行，而不会干扰到其他进程内存中的内容。
- 它容许使用各种高级的程序设计技术，如内存映射。
- 可用物理内存中的页帧与所有的进程虚拟地址空间中的页之间的关联：逆向映射（`reverse mapping`）技术有助于从虚拟内存页跟踪到对应的物理内存页，而缺页处理（`page fault handling`）则允许从块设备按需读取数据填充虚拟地址空间。

### 简介
- 第三章讨论的所有内存管理方法都关注物理内存的组织，或者内核的虚拟地址空间的管理。
- 这里考察内核用于管理用户虚拟地址空间的方法。
  - 每个应用程序都有自身的地址空间，与所有其他应用程序分隔开；
  - 通常在巨大的线性地址空间中，只有很少的段可用于各个用户空间进程，这些段彼此有一定的距离。内核需要一些数据结构，来有效地管理这些（随机）分布的段；
  - 地址空间只有极小的一部分与物理内存页直接关联。不经常使用的部分，则仅当必要时与页帧关联；
  - 内核信任自身，但无法信任用户进程。因此，各个操作用户地址空间的操作都伴随有各种检查，以确保程序的权限不会超出应有的限制，进而危及系统的稳定性和安全性。
  - `fork-exec` 模型在 UNIX 操作系统下用于产生新进程。如果实现得较为粗劣，该模型的功能并不强大。因此内核必须借助于一些技巧，来尽可能高效地管理用户地址空间。
- 假定：系统有一个内存管理单元（MMU），该单元支持使用虚拟内存。

### 进程虚拟地址空间
- 各个进程地虚拟地址空间起始于地址 0，延伸到 `TASK_SIZE - 1`，其上是内核地址空间。
- 与系统完整性相关的非常重要的一方面是，用户程序只能访问整个地址空间的下半部分，不能访问内核部分。如果没有预先达成“协议”，用户进程也不可能操作另一个进程的地址空间，因为后者的地址空间对前者不可见。
- 无论当前哪个用户进程处于活动状态，虚拟地址空间内核部分的内容总是同样的。取决于具体的硬件，这可能是通过操作各用户进程的页表，使得虚拟地址空间的上半分布看上去总是相同的。也可能是指示处理器为内核提供一个独立的地址空间，映射在各个用户地址空间之间。
- 虚拟地址空间由许多不同长度的段组成，用于不同的目的，必须分别处理。例如在大多数情况下，不允许修改 text 段，但必须可以执行其内容。另一方面，必须可以修改映射到地址空间中的文本文件内容，而不能允许执行器内容。因为这没有意义，文件的内容只是数据，并非机器代码。

#### 进程地址空间的布局
- 虚拟地址空间中包含了若干区域。其分布方式是特定于体系结构的，但所有方法都有下列共同成分：
  - 当前运行代码的二进制代码，即 text，对应虚拟内存区域为 text 段；
  - 程序使用的动态库的代码；
  - 存储全局变量和动态产生的数据的堆；
  - 用于保存局部变量和实现函数/过程调用的栈；
  - 环境变量和命令行参数的段；
  - 将文件内容映射到虚拟地址空间中的内存映射；
- 系统中的各个进程都具有一个 `struct mm_struct` 的实例，可以通过 `task_struct` 访问。这个实例保存了进程的内存管理信息：
  ```c
  <mm_types.h>
  struct mm_struct {
    ...
    unsigned long (*get_unmapped_area)(struct file *filp, unsigned long addr,
      unsigned long len, unsigned long pgoff, unsigned long flags);
    ...
    unsigned long mmap_base;  // mmap 区域的基地址
    unsigned long task_size;  // 进程虚拟内存空间的长度
    ...
    unsigned long start_code, end_code, start_data, end_data;
    unsigned long start_brk, brk, start_start;
    unsigned long arg_start, arg_end, env_start, env_end;
    ...
  }
  ```
  - 可执行代码占用的虚拟地址空间区域：`start_code ~ end_code`；
  - 已初始化数据的区域：`start_data ~ end_data`；
  - 在 ELF 二进制文件映射到地址空间后，上述区域的长度不再改变。
  - 堆：`start_brk ~ brk`，堆的起始地址在进程生命周期是不变的，但堆的长度会发生变化，因而 `brk` 的值也会变；
  - 参数列表和环境变量的位置分别由 `arg_start` 和 `arg_end`、`env_start` 和 `env_end` 描述。两个区域都位于栈中最高的区域。
  - `mmap_base` 表示虚拟地址空间中用于内存映射的起始地址，可调用 `get_unmapped_area` 在 `mmap` 区域中为新映射找到适当的位置。
  - `task_size`，存储了对应进程的地址空间长度。对本机应用程序来说，该值通常是 `TASK_SIZE`。但 64 位体系结构与前辈处理器通常是二进制兼容的。如果在 64 位计算机上执行 32 位二进制代码，则 `task_size` 描述了该二进制代码实际可见的地址空间长度。
- 各个体系结构可以通过几个配置选项影响虚拟地址空间的布局：
  - `HAVE_ARCH_PICK_MMAP_LAYOUT`：在不同 mmap 区域布局之间做出选择，需提供 `arch_pick_mmap_layout` 函数。
  - `HAVE_ARCH_UNMAPPED_AREA`：在创建新的内存映射时，除非用户指定了具体的地址，否则内核需要找到一个适当的位置。如果体系结构自身想要选择合适的位置，则必须设置预处理器符号 `HAVE_ARCH_UNMAPPED_AREA`，并相应地定义 `arch_get_unmapped_area` 函数。

#### 建立布局

### 内存映射的原理

### 数据结构
```c
<mm_types.h>
struct mm_struct {
  struct vm_area_struct *mmap;        // 虚拟内存区域列表
  struct rb_boot mm_rb;
  struct vm_area_struct *mmap_cache;  // 上一次 find_vma 的结果
  ...
}
```
#### 树和链表
#### 虚拟内存区域的表示
```c
<mm_types.h>
struct vm_area_struct {
  struct mm_struct *vm_mm;    // 所属地址空间
  unsigned long vm_start;     // vm_mm 内的起始地址
  unsigned long vm_end;       // 在 vm_mm 内结束地址之后的第一个字节的地址

  /* 各进程的虚拟内存区域链表，按地址排序 */
  struct vm_area_struct *vm_next;

  pgprot_t vm_page_prot;      // 该虚拟内存区域的访问权限
  unsigned long vm_flags;

  struct rb_node vm_vb;

  /* 对于有地址空间和后备存储器的区域来说 
   * shared 连接到 address_space->i_mmap 优先树，
   * 或连接到悬挂在优先树结点之外、类似的一组虚拟内存区域的链表，
   * 或连接到 address_space->i_mmap_nonlinear 链表中的虚拟内存区域
   */
  union {
    struct {
      struct list_head list;
      void *parent;   // 与 prio_tree_node 的 parent 成员在内存中位于同一位置
      struct vm_area_struct *head;
    } vm_set;
    struct raw_prio_tree_node prio_tree_node;
  } shared;

  /* 在文件的某一页经过写时复制之后，文件的 MAP_PRIVATE 虚拟内存区域
   * 可能同时在 i_mmap 树和 anon_vma 链表中。MAP_SHARED 虚拟内存区域
   * 只能在 i_mmap 树中。
   * 匿名的 MAP_PRIVATE、栈或 brk 虚拟内存区域（file 指针为 NULL）
   * 只能处于 anon_vma 链表中
   */
  struct list_head anon_vma_node;   // 对该成员的访问通过 anon_vma->lock 串行化
  struct anon_vma *anon_vma;        // 对该成员的访问通过 page_table_lock 串行化

  /* 用于处理该结构的各个函数指针 */
  struct vm_operations_struct *vm_ops;

  /* 后备存储器的有关信息 */
  unsigned long vm_pgoff;           // (vm_file 内)的偏移量，单位是 PAGE_SIZE，不是 PAGE_CACHE_SIZE
  struct file *vm_file;             // 映射到的文件，可能是 NULL
  void *vm_private_data;            // vm_pte，即共享内存
}
```
#### 优先查找树
- 优先查找树（priority search tree）用于建立文件中的一个区域与该区域映射到的所有虚拟地址空间之间的关联。

1. 附加的数据结构
   - 每个打开文件（和每个块设备，因为这些也可以通过设备文件进行内存映射）都表示为 `struct file` 的一个实例。该结构包含了一个指向地址空间对象 `struct address_space` 的指针。该对象是优先查找树（prio tree）的基础，而文件区间与其映射到的地址空间之间的关联即通过优先树建立。
    ```c
    <fs.h>
    struct address_space {
      struct inode *host;                 // owner: inode, block_device
      ...
      struct prio_tree_root i_mmap;       // 私有和共享映射的树
      struct list_head i_mmap_nonlinear;  // VM_NONLINEAR 映射的链表
      ...
    }

    <fs.h>
    struct file {
      ...
      struct address_space *f_mapping;
      ...
    }
    ```
   - 每个文件和块设备都表示为 `struct inode` 的一个实例。`struct file` 是通过 open 系统调用打开的文件的抽象。与此相反，inode 表示文件系统自身中的对象。
    ```c
    <fs.h>
    struct inode {
      ...
      struct address_space *i_mapping;
      ...
    }
    ```
   - 尽管这里只讨论了文件区间的映射，但实际上也可以映射不同的东西。例如，直接映射裸（raw）块设备上的区间，而不通过文件系统迂回。
   - 在打开文件时，内核将 `file->f_mapping` 设置到 `inode->i_mapping`。这使得多个进程可以访问同一个文件，而不会直接干扰到其他进程。
   - `inode` 是一个特定于文件的数据结构，而 `file` 则是特定于给定进程的。
   - [ ] 图，借助于优先树，跟踪文件的给定区间所映射到的虚拟地址空间
   - 给出 `struct addree_space` 的实例，内核可以推断相关的 inode，而后者可用于访问实际存储文件数据的后备存储器。
   - 地址空间是优先树的基本要素，而优先树包含了所有相关的 `vm_area_struct` 实例，描述了与 inode 关联的文件区间到一些虚拟地址空间的映射。由于每个 `struct vm_area` 的实例都包含了一个指向所属进程的 `mm_struct` 的指针，关联就已经建立起来了。要注意，`vm_area_struct` 还可以通过以 `i_mmap_nonlinear` 为表头的双链表与一个地址空间关联。
   - 通过链表和红黑树组织 `vm_area_struct` 实例，这些与优先树管理的 `vm_area_struct` 实例实际上是相同的。一个给定的 `struct vm_area` 实例可以包含在两个数据结构中，一个建立进程虚拟地址空间中的区域与潜在的文件数据之间的关联，一个用于查找映射了给定文件区间的所有地址空间。
2. 优先树的表示
   - 要求该数据结构不仅能够处理重叠，还要能处理相同的文件区间。
     - [ ] 图，多个进程将一个文件的相同或重叠区域映射到其虚拟地址空间中
   - 如果区间 B、C 和 D 完全包含在另一个区间 A 中，那么 A 将是 B、C 和 D 的父节点。
   - [ ] 图，管理共享的相同映射所涉及各个数据结构的关联
   - `vmset` 的 `head` 成员与 `prio_tree_node` 不重叠，因此二者尽管在同一个联合中，也可以同时使用。

### 对区域的操作
- 创建和删除区域
- 查找用于新区域的适当的内存位置
- 管理数据结构时进行优化
- [ ] 图，对区域的操作

#### 将虚拟地址关联到区域
- `struct vm_area_struct *find_vma(struct mm_struct *mm, unsigned long addr)`
  - `mm` 指定哪个进程的地址空间；
- `struct vm_area_struct *find_vma_intersection(struct mm_struct *mm, unsigned long start_addr, unsigned long end_addr)`；

#### 区域合并
- 在新区域被加到进程的地址空间时，内核会检查它是否可以与一个或多个现存区域合并。
- `vm_merge` 在可能的情况下，将一个新区域与周边区域合并：
  ```c
  // mm/mmap.c
  struct vm_area_struct *vma_merge(struct mm_struct *mm, struct vm_area_struct *prev, 
    unsigned long addr, unsigned long end, unsigned long vm_flags,
    struct anon_vma *anon_vma, struct file *file, pgoff_t pgoff,
    struct mempolicy *policy) {
      ...
    }
  
  can_vma_merge_after(...)

  can_vma_merge_before(...)

  is_mergeable_anon_vma(...)

  vma_adjust(...)
  ```

#### 插入区域
- `insert_vm_struct`

#### 创建区域
- 在向数据结构插入新的内存区域之前，内核必须确认虚拟地址空间中有足够的空闲空间，可用于给定长度的区域。给工作分配给 `get_unmapped_area()` 完成。
  ```c
  mm/mmap.c
  unsigned long get_unmapped_area(struct file *file, unsigned long addr, 
    unsigned long len, unsigned long pgoff, unsigned long flags);
  
  // 实际的遍历，或开始于虚拟地址空间中最后一个“空洞”的地址
  // 或开始于全局的起始地址 TASK_UNMAPPED_BASE
  arch_get_unmapped_area() - MAP_FIXED
  // 如果 mmap 区域自顶向下扩展，那么分配新区域的函数是：
  arch_get_unmapped_area_topdown()
  ```

### 地址空间

### 内存映射

### 反向映射

### 堆的管理

### 缺页异常的处理
`do_page_fault()`-*IA-32* 体系结构上采用的方法：
```c
fastcall void __kprobes do_page_fault(struct pt_regs *regs, unsigned long error_code);
```
- `error_code`: 提供异常原因信息的错误代码，使用了前 5 个比特位（0、1、2、3、4）
  - 0 位：置位-缺页；未置位-保护异常（没有足够的访问权限）
  - 1 位：置位-读方位；未置位-写访问
  - 2 位：置位-和心态；未置位-用户状态
  - 3 位：置位-表示检测到使用了保留位
  - 4 位：置位-表示缺页异常是在取指令时出现

- 如果地址超出用户地址空间的范围，则表明是 *vmalloc* 异常。因此该进程的页表必须与内核的主页表中的信息同步。实际上，只有访问发生在核心态，而且该异常不是由保护错误触发时，才能允许这样做。内核使用 `vmalloc_fault()` 同步页表，该函数从 *init* 的页表（在 *IA-32* 系统上，这是内核的主页表）复制相关的项到当前页表。如果其中没有找到匹配项，则内核调用 `fixup_exception()`，作为试图从异常恢复的最后尝试。

- 如果异常是在中断期间或内核线程中触发，也没有自身的上下文因而也没有独立的 *mm_struct* 实例，则内核会跳转到 `bad_area_nosemaphore` 标号（代码处于原子操作范围内也会）。如果异常源自用户空间，则返回段错误；但如果异常源自内核空间，则调用 `fixup_exception()`。

- 如果异常并非出现在中断期间，也有相关上下文，则内核检查进程的地址空间是否包含异常地址所在的区域。它调用了 `find_vma()`。在内核发现地址有效或无效时，会分别跳转到 `good_area` 和 `bad_area` 标号。

- 存在对应于异常地址的映射，并不一定意味着实际上可以允许访问。内核必须检查访问权限，可能有以下情况：
  1. 在写访问的情况下，必须置位 `VM_WRITE`。否则，访问无效，跳转到 `bad_area` 标号恢复执行。
  2. 如果是读访问现存页，该异常必定是硬件检测到的权限异常。接下来跳转到 `bad_area` 恢复执行。
  3. 如果读取不存在的页，则内核必须检查是否置位 `VM_READ` 或 `VM_EXEC`，在置位的情况下访问有效。否则拒绝读访问，内核跳转到 `bad_area`。

- 如果内核没有跳转到 `bad_area`，则代码会达到 `handle_mm_fault()` 调用。该函数负责校正缺页异常（即，读取所需数据）。

- `handle_mm_fault()` 是一个体系结构无关的例程，用于选择适当的异常恢复方法（按需调页、换入，等等），并应用选择的方法。如果页成功建立，则例程返回 `VM_FAULT_MINOR`（数据已经在内存中）或 `VM_FAULT_MAJOR`（数据需要从块设备读取）。内核接下来更新进程的统计量。

- 但在创建页时，也可能发生异常。如果用于加载页的物理内存不足，内核会强制终止该进程，在最低限度上维持系统的运行。如果对数据的访问已经允许，但由于其他原因失败（如访问的映射已经在访问的同时被另一个进程收缩，不再存在于给定的地址），则将 `SIGBUS` 信号发送给进程。

#### 用户空间缺页异常的校正

#### 内核缺页异常

#### 在内核和用户空间之间复制数据

## 第五章-锁与进程间通信
内核不仅提供了共享数据的机制，同样提供了协调对数据访问的机制。

在 SMP 系统上，各个 CPU 可能同时处于核心态，在理论上可以操作所有现存的数据结构。为阻止 CPU 彼此干扰，需要通过锁保护内核的某些范围。锁可以确保每次只能有一个 CPU 访问被保护的范围。

### 控制机制
#### 竞态条件

#### 临界区

### 内核锁机制
#### 对整数的原子操作

#### 自旋锁

#### 信号量

#### RCU 机制

#### 内存和优化屏障
现在编译器和处理器试图从代码中“压榨”出每一点性能，但其也有缺点。一个有利于提高性能的技术是指令重排。只要结果不变，这完全没问题。但编译器或处理器很难判定重排的结果是否确实与代码原本的意图匹配，特别是需要考虑副效应的时候，例如在数据写入 I/O 寄存器时，副效应是常见且必要的。

尽管锁足以确保原子性，但对编译器和处理器优化过的代码，锁不能永远保证时序正确。与竞态条件相比，这个问题不仅影响 SMP 系统，也影响单处理器计算机。

内核提供了如下几个函数，可阻止处理器和编译器进行代码重排：
1. `mb()`, `rmb()`, `wmb()`：将硬件内存屏障插入到代码流程中。
   - `rmb()`：读访问内存屏障，它保证在屏障之后发出的任何读取操作执行之前，屏障之前发出的所有读取操作都已完成。
   - `wmb()`：写访问内存屏障，语义与 `rmb()` 类似。
   - `mb()`：合并上述两者的语义。
2. `barrier()`：插入一个优化屏障（编译器屏障）。该指令告知编译器，保存在 CPU 寄存器中、在屏障之前有效的所有内存地址，在屏障之后都将失效。本质上，这意味着编译器在屏障之前发出的读写请求完成之前，不会处理屏障之后的任何读写请求。但 CPU 仍然可以重排时序（软件没有重排不代表硬件在执行时不会重排）。
3. `smp_mb()`, `smp_rmb()`, `smp_wmb()`：相当于上述的硬件内存屏障，但只适用于 SMP 系统。它们在单处理器系统上产生的是软件屏障。
4. `read_barrier_depends()`：是一种特殊形式的读访问屏障，它会考虑读操作之间的依赖性。如果屏障之后的读请求，依赖于屏障之前执行的读请求的数据，那么编译器和硬件都不能重排这些请求。
注：上述的所有命令都会影响运行时的性能。

优化屏障的一个特定应用是内核抢占机制：`preempt_disable()` 对抢占计数器加 1 因而停用了抢占；`preempt_disable()` 通过对抢占计数器减 1 而再次启用抢占。在这两个命令之间的代码，可免受抢占的影响：
```c
preempt_disable();
function_which_must_not_be_preempted();
preempt_enable();

// 如果编译器会重排，可能导致：
function_which_must_not_be_preempted();
preempt_disable();
preempt_enable();
// 或者
preempt_disable();
preempt_enable();
function_which_must_not_be_preempted();
```
考虑到上述问题，`preempt_disable()` 和 `preempt_enable()` 在实现上都插入了一个内存屏障：
```c
#define preempt_disable() \
do { \
  inc_preempt_count(); \
  barrier(); \
} while (0)
// 这防止了编译器将 inc_preempt_count() 与后续的语句交换位置

#define preempt_enable() \
do { \
  ...
  barrier(); \
  preempt_check_resched(); \
} while (0)
// 这种措施可以防止第二种错误的重排
```

到现在为止讨论的所有屏障指令，只要包含了 `<system.h>` 就可以使用。读者可能觉得内存屏障使用起来很复杂，确实是这样，正确使用内存和优化屏障需要高度的技巧。因此应该注意到，一些内核维护者不怎么喜欢内存屏障，使用该特性的代码很难进入到内核的主流版本中。因此，首先试着看一下是否能够在没有屏障的情况下完成工作，永远是值得的。这是可能的，因为在许多体系结构上锁指令也相当于内存屏障。但这需要根据实际使用内存屏障的具体实例来确认，因而无法提供一般性的建议。

#### 读者/写者锁

#### 大内核锁
这是内核锁遗产之一，它可以锁定整个内核，确保没有处理器在核心态并行运行。该锁称为大内核锁（big kernel lock），通常使用缩写表示，即 BKL。

使用 `lock_kernel()` 可以锁定整个内核，对应的解锁使用 `unlock_kernel`。

BKL 的一个特性是，它的锁深度也会进行计数。这意味着在内核已经锁定时，仍然可以调用 `lock_kernel()`。对应的解锁操作（`unlock_kernel()`）必须调用相同的次数以解锁内核，使其他处理器能够进入。

BKL 已过时，内核开发者废弃了对它的使用。新的代码决不应该使用 BKL，而应该采用上述的细粒度锁。

注：SMP 系统和启用了内核抢占的单处理器系统如果设置了配置选项 `PREEMPT_BKL`，则允许抢占大内核锁。尽管这有助于降低内核延迟，但它并非 BKL 产生的问题的正确对策，只能作为应急措施和临时解决方案，在可能的范围内做到比较好。

#### 互斥量
尽管信号量可用于实现互斥量的功能，但是信号量的通用性导致的开销通常是不必要的。因此，内核包含了一个专用互斥量的独立实现，它们不依赖信号量。或确切地说，内核包含互斥量的两种实现，一种是经典的互斥量，另一种是用来解决优先级反转问题的实时互斥量。

1. 经典的互斥量
  ```c
  struct mutex {
    atomic_t count;
    spinlock_t wait_lock;
    struct list_head wait_list;
  };
  ```
  - 如果互斥量未锁定，则 `count` 为 1。锁定分两种情况，如果只有一个进程在使用互斥量，则 `count` 为 0；如果胡质量被锁定，且有进程在等待互斥量解锁（在解锁时需要唤醒等待进程），则 `count` 为负值。
  - ...

2. 实时互斥量
  考虑一种情况，系统上有两个进程运行：进程 A 优先级高，进程 C 优先级低。假定进程 C 已经获取了一个互斥量，正在所保护的临界区中运行，且短时间内不打算退出。但在进程 C 进入临界区之后不久，进程 A 也试图获取保护临界区的互斥量。由于进程 C 已经获取该互斥量，因而进程 A 必须等待。这导致高优先级的进程 A 等待低优先级的进程 C。如果有第 3 个进程 B，优先级介于进程 A 和进程 C 之间，情况会更加糟糕。假定进程 C 仍然持有锁，进程 A 在等待，现在进程 B 开始运行。由于进程 B 的优先级高于进程 C，因此可以抢占进程 C，但它实际上也抢占了进程 A，尽管进程 A 的优先级高于进程 B。如果进程 B 继续运行，那么它可以让进程 A 等待更长时间，因为进程 C 被进程 B 抢占，所以它只能更慢地完成其操作。因此看起来仿佛进程 B 的优先级高于进程 A 一样，这种糟糕的情况称为无限制优先级反转（unbounded priority inversion）。

  该问题可以通过优先级继承解决。如果高优先级进程阻塞在互斥量上，该互斥量当前由低优先级进程持有，那么进程 C 的优先级（在上述例子中）临时提高到进程 A 的优先级。如果进程 B 现在开始运行，只能得到与进程 A 竞争情况下的 CPU 时间，从而理顺了优先级的问题。

  实时互斥量：
   - 需要在编译内核时通过配置选项 `CONFIG_RT_MUTEX` 显式启用。
   - 与普通的互斥量相比，它们实现了优先级继承（priority inheritance），该特性可用于解决（或在最低限度上缓解）优先级反转的影响。
   - 定义：
    ```c
    struct rt_mutex {
      spinlock_t wait_lock;
      struct plist_head wait_list;
      struct task_stuct *owner;
    };
    ```
    - 互斥量的所有者通过 `owner` 指定，`wait_lock` 提供实际的保护。所有等待的进程都在 `wait_list` 中排队。
    - 与普通互斥量相比，决定性的改变是等待列表中的进程按优先级排序。在等待列表改变时，内核可相应地校正锁持有者的优先级。这需要用到调度器的一个接口，可由函数 `rt_mutex_setprio()` 提供。该函数更新动态优先级 `task_struct->prio`，而普通优先级 `task_struct->normal_priority` 不变。
    - 此外，内核提供了几个标准函数：`rt_mutex_init()`、`rt_mutex_lock()`、`rt_mutex_unlock()`、`rt_mutex_trylock()`，工作方式与普通互斥量完全相同。

#### 近似的 per-CPU 计数器

#### 锁竞争与细粒度锁

### System V 进程间通信
Linux 使用 System V (SysV) 引入的机制，来支持用户进程的进程间通信和同步。内核通过系统调用提供了各种例程，使得用户库（通常是 C 标准库）能够实现所需的操作。

除了信号量之外，SysV 的进程间通信方案还包括进程间的消息交换和共享内存区域。

#### System V 机制

#### 信号量
System V 信号量在 `sem/sem.c` 实现，对应的头文件是 `<sem.h>`，这种信号量与上述的内核信号量没有任何关系。

1. 使用 System V 信号量
2. 数据结构
3. 实现系统调用
4. 权限检查

#### 消息队列

#### 共享内存

### 其他 IPC 机制
#### 信号

#### 管道和套接字

## 第六章-设备驱动程序
设备驱动程序是内核的关键领域，因为许多用户判断操作系统性能时，主要是通过有驱动程序可用的外设数目和驱动程序对外设的支持程度来判断。

### I/O 体系结构
与外设的通信通常称之为输入输出，一般缩写为 I/O。

在实现外设的 I/O 时，内核必须处理 3 个可能出现问题的领域：
1. 首先，必须根据具体的设备类型和模型，使用各种方法对硬件寻址
2. 其次，内核必须向用户应用程序和系统工具提供访问各种设备的方法，且尽量采用统一的方案
3. 用户空间需要知道内核中有哪些设备可用

与外设的通信是层次化的

扩展硬件
1. 总线系统
   - PCI, Peripheral Component Interconnect
   - ISA, Industrial Standard Architecture
   - SBus
   - IEEE 1394 (FireWire, I.link)
   - USB, Universal Serial Bus
   - SCSI, Small Computer System Interface
   - 并口与串口（Parallel and Serial Interface）
2. 与外设的交互
  有几种方法可以与连接到系统的硬件进行通信
   - I/O 端口
   - I/O 内存映射
   - 轮询与中断
3. 通过总线控制设备
  并非所有设备都是直接通过 I/O 语句寻址，也有通过总线系统访问。具体的方式与所用的总线和设备相关。
   - 不同的总线类型称作系统和扩展总线
   - 就系统总线而言（对很多处理器类型和体系结构来说，是 PCI 总线），可使用 I/O 语句和内存映射与总线自身和附接的设备通信。
   - 扩展总线如 USB、IEEE1394、SCSI 等，通过明确定义的总线协议与附接的设备交换数据和命令。内核通过 I/O 语句和内存映射与总线自身通信（这些总线通常是 PCI 插槽中的扩展卡，它们必须以适当方式进行编址），同时提供了平台无关的例程，使总线能够与附接的设备通信。
   - 与总线上附接的设备通信，不见得一定在内核空间中由设备驱动程序进行，有时也可能在用户空间中实现。

### 访问设备
设备特殊文件（设备文件）用于访问扩展设备。这些文件并不关联到硬盘或任何其他存储介质上的数据段，而是建立了与某个设备驱动程序的连接，以支持与扩展设备的通信。

#### 设备文件
设备并不是通过其文件名标识，而是通过文件的主、从设备号标识。

#### 字符设备、块设备和其他设备
根据外设与系统之间交换数据的方法，可以将设备分为几种类型。有些设备非常适合于面向字符的数据交换，因为数据传输量很低；有的设备则更适合于处理包含固定数目字节的数据块。

1. 标识设备文件
   - `ls` 命令输出结果：字符设备-c、块设备-b
   - 设备文件没有长度，而增加另外两个值，分别是主设备号和从设备号
   - 注：名称无法表示设备文件的实际功能，这主要是通过主从设备号表示的；`mknod` 用于创建设备文件

2. 动态创建设备文件
   - 几乎所有的发布版都将 `/dev` 内容的管理工作切换到 udevd，这是一个守护进程，允许从用户层动态创建设备文件。
   - 由于引入了 udev 机制，`/dev` 不再放置到基于磁盘的文件系统中，而是使用 tmpfs，这是 RAM 磁盘文件系统 ramfs 的一种轻型变体。这意味着设备结点不是持久性的，系统关机/重启后就会消失；`/dev` 中没有旧的、过时的设备文件
   - udev 守护进程还有一些职责，如确保无论采用何种设备拓扑结构，特定设备对应的设备结点总是名称相同。有关 udev 守护进程处理此类情况的方式，更多信息参考手册页 udevd(5)。这完全是用户空间问题，内核无需关注

#### 使用 ioctl 进行设备寻址

#### 主从设备号的表示

#### 注册
1. 数据结构
2. 注册过程

### 与文件系统关联
设备文件也是通过虚拟文件系统管理，普通文件和设备文件都是通过完全相同的接口访问。

#### inode 中的设备文件成员
虚拟文件系统中的每个文件都关联到恰好一个 inode，`struct inode` 中与设备驱动程序有关的成员：
```c
struct inode {
  ...
  dev_t i_rdev;
  ...
  umode_t i_mode;
  ...
  struct file_operations *i_fop;
  ...
  union {
    ...
    struct block_device *i_bdev;
    struct cdev *i_cdev;
  };
  ...
};
```
- 为唯一地标识与一个设备文件关联的设备，内核在 `i_mode` 中存储了文件类型（面向块，或者面向字符）

#### 标准文件操作
在打开一个设备文件时，各种文件系统的实现会调用 `init_special_inode()`，为块设备或字符设备创建一个 inode。
```c
void init_special_inode(struct inode *inode, umode_t mode, dev_t rdev) 
{
  inode->i_mode = mode;
  if (S_ISCHR(mode)) {
    inode->i_fop = &def_chr_fops;
    inode->i_rdev = rdev;
  } else if (S_ISBLK(mode)) {
    inode->i_fop = &def_blk_fops;
    inode->i_rdev = rdev;
  } else 
    printk(KERN_DEBUG "init_special_inode: bogus i_mode (%o)\n", mode);
}
```

#### 用于字符设备的标准操作
字符设备的情况最初非常含混，因为只有一个文件操作可用：
```c
struct file_operations def_chr_fops = {
  .open = chrdev_open,
};
```
- `chrdev_open()` 函数的主要任务就是向该结构填入适用于已打开设备的函数指针，使得能够在设备文件上执行有意义的操作，并最终能够操作设备自身。

#### 用于块设备的标准操作
相比字符设备，块设备遵循的方案更加一致。这使得内核刚开始就有很多操作可供选择。这些操作的指针群集到一个称作 `blk_fops` 的通用结构中：
```c
const struct file_operations def_blk_fops = {
  .open = blkdev_open,
  .release = blkdev_close,
  .llseek = block_llseek,
  .read = do_sync_read,
  .write = do_sync_write,
  .aio_read = generic_file_aio_read,
  .aio_write = generic_file_aio_write_nolock,
  .mmap = generic_file_mmap,
  .fsync = block_fsync,
  .unlocked_ioctl = block_ioctl,
  .splice_read = generic_file_splice_read,
  .splice_write = generic_file_splice_write,
};
```
- 读写操作由通用的内核例程进行
- 内核中的缓存自动用于块设备

与字符设备相比，上述数据结构无法完全描述块设备，因为对块设备的访问不是分别处理单个的请求，而是通过由缓存和请求队列构成的精细、复杂的系统来高效地管理。缓存主要由通用的内核代码操作，而请求队列则由块设备层管理。

### 字符设备操作
字符设备的硬件通常非常简单，而且相关的驱动程序并不难于实现。

#### 表示字符设备
```c
struct cdev {
  struct kobject kobj;
  struct module *owner;
  const struct file_operations *ops;
  struct list_head list;
  dev_t dev;
  unsigned int count;
}
```
- `list` 用来实现一个链表，其中包含所有表示该设备的设备特殊文件的 inode。

#### 打开设备文件
`chrdev_open()`...

### 块设备操作

### 资源分配

### 总线系统

## 第七章-模块

## 第八章-虚拟文件系统
每种操作系统都至少有一种“标准文件系统”

*Linux* 附带的 *Ext2/3* 文件系统是一种标准文件系统，该文件系统被证实非常健壮且适于日常使用。

为支持各种本机文件系统，且在同时允许访问其他操作系统的文件，*Linux* 内核在用户进程（或 C 标准库）和文件系统实现之间引入了一个抽象层。该抽象层称之为虚拟文件文件系统（*Virtual File System, VFS*）
- [ ] 图，用作文件系统抽象的 VFS 层

*VFS* 的任务并不简单。一方面，它用来提供一种操作文件、目录及其他对象的统一方法。另一方面，它必须能够与各种具体文件系统给出的方法实现达成妥协，这些实现在具体细节、总体设计方面都有一些不同之处。但 *VFS* 的回报很高，它使得 *Linux* 内核更加灵活了。

内核支持 40 多种文件系统：来自 *MS-DOS* 的 *FAT* 文件系统、*UFS (Berkeley UNIX)*、用于 *CD-ROM* 的 *iso9660*、网络文件系统（如 *coda* 和 *NFS*）和虚拟的文件系统（如 *proc*）

### 文件系统类型
- 文件系统一般可以分为下面 3 种：
  1. 基于磁盘的文件系统（*Disk-based Filesystem*）
     - 所有这些文件系统都使用面向块的介质，必须解决的问题：如何将文件内容和结构信息存储在目录层次结构上。在这里我们对与底层块设备通信的方法不感兴趣，内核中对应的驱动程序对此提供了统一的接口。从文件系统的角度来看，底层设备无非是存储块组成的一个列表，文件系统相当于对该列表实施一个适当的组织方案。
  2. 虚拟文件系统（*Virtual Filesystem*）
     - 在内核中生成，是一种使用户应用程序与用户通信的方法。
     - （*proc*）它不需要在任何种类的硬件设备上分配存储空间，文件的内容通过从内核内存中的数据结构提取而来。
  3. 网络文件系统（*Network Filesystem*）
     - 允许访问另一台计算上的数据，该计算机通过网络连接到本地计算机
     - 内核无需关注文件存取、数据组织和硬件通信的细节，这些由远程计算机的内核处理
     - 对此类文件系统中文件的操作都通过网络连接进行，使用特定的协议
     - 即使在内核处理网络文件系统时，仍然需要文件长度、文件在目录层次中的位置以及文件的其他信息

由于 *VFS* 抽象层的存在，用户空间进程不会看到上述文件系统之间的区别。

### 通用文件模型
并非每一种文件系统都支持 *VFS* 中的所有抽象。设备文件无法存储在源自其他系统的文件系统中（如 *FAT*），后者的设计没有考虑到此类对象。

*VFS* 提供一种结构模型，包含了一个强大文件系统所应具备的所有组件。但该模型只存在于虚拟中，必须使用各种对象和函数指针与每种文件系统适配。所有文件系统的实现都必须提供与 *VFS* 定义的结构配合的例程，以弥合两种视图之间的差异。

在 *Ext2* 和 *VFS* 结构之间转换，几乎不会损失时间

在处理文件时，内核空间和用户空间使用的主要对象是不同的。对用户程序来说，一个文件由一个文件描述符标识。该描述符是一个整数，在所有有关文件的操作中用作标识文件的参数。文件描述符是在打开文件时由内核分配，只在一个进程内部有效。两个不同进程可以使用同样的文件描述符，但二者并不指向同一个文件。基于同一个描述符来共享文件是不可能的。

内核处理文件的关键是 *inode*。每个文件（和目录）都有且只有一个对应的 *indoe*，其中包含元数据（如访问权限、上次修改的日期，等等）和指向文件数据的指针。但 *inode* 并不包含一个重要的信息项，即文件名。

#### inode
用数据结构表示目录的层次结构

目录只是一种特殊的文件

*inode* 的成员分类：
1. 描述文件状态的元数据
2. 保存实际文件内容的数据段（或指向数据的指针）

根目录 `/` *inode*
- 对系统来说必须总是已知的
- 该目录由一个 *inode* 表示，其数据段并不包含普通数据，而是根目录下的各个目录项。这些项可能代表文件或其他目录。每个项由两个成员组成：
  - 该目录项的数据所在 *inode* 的编号
  - 文件或目录的名称

系统中所有 *inode* 都有一个特定的编号，用于唯一地标识各个 *inode*。文件名和 *inode* 之间的关联即通过该编号建立。

- [ ] 图，查找 `/usr/bin/emacs` 的操作

#### 链接
链接（*link*）用于建立文件系统对象之间的联系，这不符合经典的树模型。有两种类型的链接，符号链接（软连接）与硬链接。

1. 符号链接
   - 对每个符号链接都使用了一个独立的 *inode*。相应 *inode* 的数据段包含一个字符串，给出了链接目标的路径。
   - 目标文件删除时，符号链接仍然继续保持

2. 硬链接
   - 在硬链接建立时，创建的目录项使用了一个现存的 *inode* 编号（共享同一个 *inode*）
   - 在 *inode* 中加入一个计数器，只有在计数器归 0 时，才能确认该 *inode* 不再使用，可以从系统删除

#### 编程接口
用户进程和内核的 *VFS* 实现之间的接口照例由系统调用组成，其中大多数涉及对文件、目录和一般意义上的文件系统的操作。

对上述的操作，内核提供了 50 多个系统调用。（利用文件进行通信，不仅可以通过文件描述符进行，还可以借助于流（*stream*）。后者提供了一个方便的接口。但后者在 C 标准库中实现，并非由内核实现。流在内部使用了通常的文件描述符。）

文件使用之前，必须用 `open()` 或 `openat()` 系统调用打开。在成功打开文件之后，内核向用户层返回一个非负的整数。这种分配的文件描述符起始于3。尽管没有明确规定，这个标识符号之所以不从 0 开始，是因为所有的进程都分配了前 3 个标识符（0-2）。0 表示标准输入，1 表示标准输出，2 表示标准错误输出。

在文件已经打开后，其名称就没什么用处了。它现在由其文件描述符唯一标识，所有其他库函数都需要传递文件描述符作为一个参数（进一步传递到系统调用）。尽管传统上文件描述符在内核中足以标识一个文件，但现在情况不再如此。由于多个命名空间和容器的引入，具有相同数值的多个文件描述符可以共存于内核中。对文件的唯一表示由一个特殊的数据结构（`struct file`）提供。

调用 `close()` 关闭与文件的“连接”（释放文件描述符，以便在后续打开其他文件时使用）

在一个打开文件中的当前位置保存在文件位置指针（*file pointer*）中，这是一个整数，指定了当前位置与文件起始点的偏移量。对随机存取文件而言，该指针可以设置为任何值，只要不超出文件存储容量范围即可。这用于支持对文件数据的随机访问。其他文件类型，如命名管道或字符设备的设备文件，不支持这种做法。它们只能从头至尾顺序读取。

在文件打开时，可以指定各种标志（如 `O_RDONLY`），用来规定文件的存取模式。

#### 将文件作为通用接口
万物皆文件

该规则有少数例外（例如，网络设备），但大多数内核导出、用户程序使用的函数都可以通过 *VFS* 定义的文件接口访问。

### VFS 的结构
#### 结构概览
VFS由两个部分组成：文件和文件系统，这些都需要管理和抽象。

1. 文件的表示
   - [ ] 各个 VFS 组件的相互关系
   - 在抽象对底层文件系统的访问时，并未使用固定的函数，而是使用了函数指针。这些函数指针保存在两个结构中：*inode* 操作、文件操作
   - 文件与进程的关联：*task_struct* 包含一个成员，其中保存了所有打开的文件（通过一种迂回方式）。该成员是一个数组，访问时使用文件描述符作为索引。各个数组项包含的对象不仅关联到对应文件的 *inode*，还包含一个指针，指向用于加速查找操作的目录项缓存的一个成员。
   - 各个文件系统的实现也能在 *VFS inode* 中存储自身的数据

2. 文件系统和超级块信息
   - *VFS* 支持的文件系统类型通过一种特殊的内核对象连接进来，该对象提供了一种读取超级块的方法。除了文件系统的关键信息（块长度、最大文件长度，等等）之外，超级块还包含了读、写、操作 *inode* 的函数指针。
   - 内核还建立了一个链表，包含所有活动文件系统的超级块实例。之所以使用活动（*active*）这个术语替代已装载（*mounted*），是因为在某些环境中，有可能使用一个超级块对应几个装载点。
     - 尽管每个文件系统在 *file_system_type* 中只出现一次，但所有超级块实例的链表中，可能有几个同一文件系统类型的超级块实例，因为在各个块设备/分区上可能存储了同一类型的几个文件系统。
   - 超级块结构的一个重要成员是一个列表，包括相关文件系统中所有修改过的 *inode*（内核称之为脏 *inode*）。根据该列表很容易标识已经修改过的文件和目录，以便将其写回到存储介质。回写必须经过协调，保证在一定程度上最小化开销，因为这是一个非常费时的操作。另一方面，如果写回修改数据的间隔太长也可能有严重后果，因为系统崩溃会导致不能恢复的数据丢失。内核会周期性扫描脏块的列表，并将修改传输到底层硬件。

#### inode
```c
struct inode {
  ...
}
```
- 该结构包括几个链表元素，用于分类管理各个 *inode*。
- 这里考察的inode结构是用于在内存中进行处理，因而包含了一些实际介质上存储的 *inode* 所没有的成员。这些是由内核自身在从底层文件系统读入信息时生成或动态建立。还有一些文件系统，如 *FAT* 和 *Reiserfs* 没有使用经典意义上的 *inode*，因此必须从其包含的数据中提取信息并生成这里给出的形式。
- `i_atime` - 最后访问的时间; `i_mtime` - 最后修改的时间（*inode* 相关的数据段内容）; `t_ctime` - 最后修改 *inode* 的时间（*inode* 结构自身）
- `i_size` - 文件长度（单位：字节）; `i_blocks` - 文件按块计算的长度（文件系统的特征，不属于文件自身）
- `i_ino` - *inode* 的编号标识; `i_count` - 使用计数器，指定访问该 *inode* 结构的进程数目; `i_nlink` - 记录使用该 *inode* 的硬件链接总数
- `i_mode` - 文件类型和访问权限; `i_uid` 和 `i_gid` - 与该文件相关的 *UID* 和 *GID*
- `i_rdev` - 在 inode 表示设备文件时需要此字段，它表示与哪个设备进行通信；类型：数字，通过这个数字包含的信息，即足以找到有关目标设备、我们感兴趣的所有信息（对于块设备会找到一个 `struct block_device` 的实例）；如果 *inode* 表示设备特殊文件，那么 `i_rdev` 之后的匿名联合就包含了指向设备专用数据结构的指针。
- `i_bdev` - 用于块设备; `i_pipe` - 包含了用于实现管道的 *inode* 的相关信息; `i_cdev` - 用于字符设备（一个 *inode* 一次只能表示一种类型的设备）
- `i_devices` - 与设备文件的处理有关联：利用该成员作为链表元素，使得块设备或字符设备可以维护一个 *inode* 的链表，每个 *inode* 表示一个设备文件，通过设备文件可以访问对应的设备。尽管在很多情况下每个设备一个设备文件就足够了，但还有很多种可能性。例如 `chroot` 造成的环境，其中一个给定的块设备或字符设备可以通过多个设备文件，因而需要多个 *inode*。
- `i_op` - *inode* 操作; `i_fop` - 文件操作

1. *inode* 操作
   - *file_operations* 用于操作文件中包含的数据，而 *inode_operations* 负责管理结构性的操作（例如删除一个文件）和文件相关的元数据（例如，属性）。
   - *inode* 操作数据结构
    ```c
    struct inode_operations {
      ...
    }
    ```
    - `struct dentry` 是一种标准化的数据结构，可以表示文件名或目录。它还建立了文件名及其 *inode* 之间的关联。

2. *inode* 链表
   - 每个 *inode* 都有一个 `i_list` 成员，可以将 *inode* 存储在一个链表中。根据 *inode* 的状态，它可能有 3 种主要的情况：
     1. *inode* 存在于内存中，未关联到任何文件，也不处于活动使用状态。
     2. *inode* 结构在内存中，正在由一个或多个进程使用，通常表示一个文件。两个计数器的值都必须大于 0。文件内容和 *inode* 元数据都与底层块设备上的信息相同。也就是说，从上一次与存储介质同步以来，该 *inode* 没有改变过。
     3.  *inode* 处于活动使用状态。其数据内容已经改变，与存储介质上的内容不同。这种状态的 *inode* 被称作脏的。
   - 在 `fs/inode.c` 中内核定义了两个全局变量用作表头，`inode_unused` 用于有效但非活动的 *inode*（上述第1类），`inode_in_use` 用于所有使用但未改变的 *inode*（第2类）。脏的 *inode* （第3类）保存在一个特定于超级块的链表中。
   - 第4种可能性出现得不那么频繁，一般是与一个超级块相关的所有 *inode* 都无效时。在所有情况下，代码都结束于 `invalidate_inodes()` 函数中，无效 *inode* 保存在一个本地链表中，与 *VFS* 代码再没有关系了
   - 每个 *inode* 不仅出现在特定于状态的链表中，还在一个散列表中出现，以支持根据 *inode* 编号和超级块快速访问 *inode*，这两项的组合在系统范围内是唯一的。该散列表是一个数组，可以借助于全局变量 `inode_hashtable` 来访问。该表启动期间在 `fs/inode.c` 中的 `inode_init()` 函数中初始化。
     - `fs/inode.c` 中的 `hash()` 函数用于计算散列和。它将 *inode* 编号和超级块对象的地址合并为一个唯一的编号，保证位于散列表已经分配的下标范围内。碰撞照例通过溢出链表解决。*inode* 的成员 `i_hash` 用于管理溢出链表。
   - 除了散列表之外，*inode* 还通过一个特定于超级块的链表维护，表头是 `super_block->s_inodes`。`i_sb_list` 用作链表元素。
     - 但超级块管理了更多的 *inode* 链表，与 `i_sb_list` 所在的链表是独立的。如果一个 *inode* 是脏的，即其内容已经被修改，则列入脏链表，表头为 `super_block->s_dirty`，链表元素是`i_list`。这样做有下列好处：在写回数据时（数据回写通常也称之为同步）不需要扫描系统所有的 *inode*，考虑脏链表上所有的 *inode* 就足够了。另外两个链表（表头为 `super_block->s_io` 和 `super_block->s_more_io`）使用同样的链表元素 `i_list`。这两个链表包含的是已经选中向磁盘回写的 *inode*，但正在等待回写进行。

#### 特定于进程的信息
- 文件描述符用于在一个进程内唯一地标识打开的文件。这假定了内核能够在用户进程中的描述符和内核内部使用的结构之间，建立一种关联。每个进程的 `task_struct` 中包含了用于完成该工作的成员。
  ```c
  struct task_struct {
    ...
    // 文件系统信息
    int link_count, total_link_count;
    ...
    struct fs_struct *fs;
    // 打开文件信息
    struct files_struct *files;
    // 命名空间
    struct nsproxy *nsproxy;
    ...
  }
  ```
  - 整数成员 `link_count` 和 `total_link_count` 用于在查找环形链表时防止无限循环
  - 进程的文件系统相关数据保存在 `fs` 中
  - 由于内核允许同时运行多个模仿独立系统的容器，从容器角度看似“全局”的每个资源，都由内核包装起来，分别根据每个容器进行管理。虚拟文件系统也受到影响，因为各个容器可能因装载点的不同导致不同的目录层次结构。对应的信息包含在 `ns_proxy->mnt_namespace`
  - `files` 包含当前进程的各个文件描述符
    ```c
    struct files_struct {
      atomic_t count;
      struct fdtable *fdt;
      struct fdtable *fdtab;

      int next_fd;
      struct embedded_fd_set close_on_exec_init;
      struct embedded_fd_set open_fds_init;
      struct file *fd_array[NR_OPEN_DEFAULT];
    };
    ```
    - `next_fd`: 表示下一次打开新文件时使用的文件描述符
    - `close_on_exec_init` 和 `open_fds_init` 是位图：对执行 *exec* 时将关闭的所有文件描述符，在 `close_on_exec` 中对应的比特位都将置位。`open_fds_init` 是最初的文件描述符集合。
      ```c
      struct embedded_fd_set {
        unsigned long fds_bits[1];
      };
      ```
    - `fd_array` 的每个数组项都是一个指针，指向每个打开文件的 `struct file` 实例，默认情况下，内核允许每个进程打开  `NR_OPEN_DEFAULT` 个文件。该值定义在 `include/linux/sched.h` 中，默认值为 `BITS_PER_LONG`。因此在 32 位系统上，允许打开文件的初始数目是 32。如果一个进程试图同时打开更多的文件，内核必须对 `files_struct` 中用于管理与进程相关的所有文件信息的各个成员，分配更多的内存空间。
    - 最重要的信息包含在 `fdtab` 中
      ```c
      struct fdtable {
        unsigned int max_fds;
        struct file **fd;   // 当前 fd_array
        fd_set *close_on_exec;
        fd_set *open_fds;
        struct rcu_head rcu;
        struct files_struct *free_files;
        struct fdtable *next;
      };
      ```
      - 这里使用了 *RCU* 机制以便在无需锁定的情况下读取这些数据结构，这可以加速处理
      - `max_fds` 指定了进程当前可以处理的文件对象和文件描述符的最大数目，这里没有固有的上限，因为这两个值都可以在必要时增加（只要没有超出由 `Rlimit` 指定的值），尽管内核使用的文件对象和文件描述符的数目总是相同的，但必须定义不同的最大数目。
      - `fd` 是一个指针数组，每个数组项指向一个file结构的实例，管理一个打开文件的所有信息。用户空间进程的文件描述符充当数组索引。该数组当前的长度由 `max_fds` 定义
      - `open_fds` 是一个指向位域的指针，该位域管理着当前所有打开文件的描述符。每个可能的文件描述符都对应着一个比特位。如果该比特位置位，则对应的文件描述符处于使用中；否则该描述符未使用。当前比特位置的最大数目由 `max_fdset` 指定。
      - `close_on_exec` 也是一个指向位域的指针，该位域保存了所有在 *exec* 系统调用时将要关闭的文件描述符的信息。
    - 初看起来，`struct fdtable` 和 `struct files_struct` 之间某些信息似乎是重复的： *exec* 时关闭文件描述符和打开文件描述符两个位图，以及 *file* 指针的数组。事实上并非如此，因为 `files_struct` 中的成员是数据结构真正的实例（两个位图），而 *fdtable* 的成员则是指针。实际上，后者的成员 `fd`、`open_fds` 和 `close_on_exec` 都初始化为指向前者对应的 3 个成员。因此，`fd` 数组包含了 `NR_OPEN_DEFAULT` 项。`close_on_exec` 和 `open_fds` 位图最初包括 `BITS_PER_LONG` 个比特位。 由于 `NR_OPEN_DEFAULT` 设置为 `BITS_PER_LONG`，所有这些长度都是相同的。如果需要打开更多文件，内核会分配一个 `fd_set` 的实例，替换最初的 `embedded_fd_set`。要注意，`struct embedded_fd_set` 可以转换为 `struct fd_set`。在这种意义上讲，*embedded_fd_set* 是 `fd_set` 的缩小版，可以同样使用，但占用的空间较小。如果两个位图或 `fd` 数组的初始长度限制太低，内核可以将对应的指针指向更大的结构，以扩展空间。数组扩展的“步长”是不同的，这也说明了为什么该结构中描述符和文件数量需要两个不同的最大值。
      ```c
      #define __NFDBITS   (8 * sizeof(unsigned long))
      #define __FD_SETSIZE  1024
      #define __FDSET_LONGS (__FD_SETSIZE/__NFDBITS)

      typedef struct {
        unsigned long fds_bits[__FDSET_LONGS];
      } __kernel_fd_set;

      typedef __kernel_fd_set fd_set;
      ```
    - `struct file` 该结构保存了内核所看到的文件的特征信息：
      ```c
      struct file {
        struct list_head f_list;
        struct path f_path;
      #define f_dentry f_path.dentry
      #define f_vfsmnt f_path.mnt
        const struct file_operations *f_ops;
        atomic_t f_count;
        unsigned int f_flags;
        mode_t f_mode;
        loff_t f_ops;
        struct fown_struct f_owner;
        unsigned int f_uid, f_gid;
        struct file_ra_state f_ra;

        unsigned long f_version;
        ...
        struct address_space *f_mapping;
        ...
      };
      ```
      - `f_uid` 和 `f_gid` 指定了用户的 *UID* 和 *GID*
      - `f_owner` 包含了处理该文件的进程有关的信息（因而也确定了 `SIGIO` 信号发送的目标 *PID*，以实现异步输入输出）
      - 预读特征保存在 `f_ra`。这些值指定了在实际请求文件数据之前，是否预读文件数据、如何预读（预读可以提高系统性能）
      - 打开文件时传递的模式参数（通常指定读、写或读写访问模式）保存在 `f_mode` 字段中
      - `f_flags` 指定了在 `open` 系统调用时传递的额外的标志
      - 文件位置指针的当前值（对于顺序读取操作或读取文件特定部分的操作，都很重要）保存在 `f_pos` 变量中，表示与文件起始处的字节偏移
      - `f_path` 封装了下面两部分信息
        - 文件名和 *inode* 之间的关联
        - 文件所在文件系统的有关信息
          ```c
          struct path {
            struct vfsmount *mnt;
            struct dentry *dentry;
          };
          ```
          - `struct dentry` 提供了文件名和 *inode* 之间的关联
          - 有关所在文件系统的信息包含在 `struct vfs_mount` 中
        - 由于以前的一些内核版本没有使用 `struct path`，而是将 `dentry` 和 `vfsmount` 成员显式嵌入到 `struct file` 中，因此需要使用相应的辅助宏，以确保尚未更新到新接口的代码仍然能够工作。
      - `f_op` 指定了文件操作调用的各个函数
      - `f_version` 由文件系统使用，以检查一个 *file* 实例是否仍然与相关的 *inode* 内容兼容，这对于确保已缓存对象的一致性很重要
      - `f_mapping` 指向属于文件相关的 *inode* 实例的地址空间映射。通常它设置为 `inode->i_mapping`，但文件系统或其他内核子系统可能会修改它
      - 每个超级块都提供了一个 `s_list` 成员用作表头，以建立 *file* 对象的链表，链表元素是 `file->f_list`。该链表包含该超级块表示的文件系统的所有打开文件。例如，在以读/写模式装载的文件系统以只读模式重新装载时，会扫描该链表。当然，如果仍然有按写模式打开的文件，是无法重新装载的，因而内核需要检查该链表来确认
      - *file* 实例可以用 `get_empty_filp()` 分配，该函数利用了自身的缓存并将实例用基本数据预先初始化
    - 提高初始限制：每当内核打开一个文件或做其他的操作时，如果需要 `files_struct` 提供比初始值更多的项，则调用 `expand_files()`。该函数检查是否有必要增大数组，如果是这样则调用`expand_fdtable()`。`alloc_fdtable()` 分配一个文件描述符表，可以容纳最大可能数目的项，并为增大的位图分配内存。此后，该函数将文件描述符表先前的内容复制到新的、增大的实例中。将 `files_struct->fdt` 指针切换到新实例的过程由 *RCU* 函数 `rcu_assign_pointer()` 处理，然后释放旧的文件描述符表。

#### 文件操作
`struct file_operations`
- 仅当文件系统以模块形式装载并未编译到内核时，才使用 `owner` 项（`NULL` 指针表示文件系统已经持久编译到内核中）。该项指向在内存中表示模块的数据结构。

1. 目录信息
  ```c
  struct fs_struct {
    atomic_t count;
    int umask;
    struct dentry *root, *pwd, *altroot;
    struct vfsmount *rootmnt, *pwdmnt, *altrootmnt;
  };
  ```

2. VFS 命名空间
  ```c
  struct nsproxy {
    ...
    struct mnt_namespace *mnt_ns;
    ...
  };

  struct mnt_namespace {
    atomic_t count;
    struct vfsmount *root;
    struct list_head list;
    ...
  };

#### 目录项缓存
1. *dentry* 结构
  ```c
  struct dentry {
    ...
  };
  ```
2. 缓存的组织
3. *dentry* 操作
  ```c
  struct dentry_operations {
    ...
  };
  ```
4. 标准函数

### 处理 VFS 对象
#### 文件系统操作
尽管文件操作对所有应用程序来说都属于标准功能，但对文件系统的操作只限于少量几个系统程序，即用于装载和卸载文件系统的 *mount* 和 *umount* 程序。

文件系统在内核中是以模块化形式实现的。这意味着可以将文件系统编译到内核中，而内核自身在编译时也完全可以限制不支持某个特定的文件系统。事实上大约有 50 个文件系统，把这些代码都编译到内核中几乎没有意义。

因此，每个文件系统在使用前必须注册到内核，这样内核能够了解可用的文件系统，并按需调用装载功能

1. 注册文件系统
   - 在文件系统注册到内核时，文件系统是编译为模块，或者持久编译到内核中，都没有差别。如果不考虑注册的时间（持久编译到内核的文件系统在启动时注册，模块化文件系统在相关模块载入内核时注册），在两种情况下所用的技术方法是相同的。
   - 注册接口：`register_filesystem(), fs/super.c`，将未在全局链表中的文件系统添加到链表末尾
   - 描述文件系统的数据结构
    ```c
    struct file_system_type {
      const char *name;
      int fs_flags;
      struct super_block *(*get_sb)(struct file_system_type *, int,
        const char *, void *, struct vfsmount *);
      void (*kill_sb)(struct super_block *);
      struct module *owner;
      struct file_system_type *next;
      struct list_head fs_supers;
    };
    ```
    - `fs_supers` 和 `get_sb()`: 对于每个已经装载的文件系统，在内存中都创建了一个超级块结构。该结构保存了文件系统它本身和装载点的有关信息。由于可以装载几个同一类型的文件系统（最好的例子是 *home* 和 *root* 分区，二者的文件系统类型通常相同），同一文件系统类型可能对应了多个超级块结构，这些超级块聚集在一个链表中，`fs_supers` 是对应的表头。另外，用于从底层存储介质读取超级块的函数 `get_sb()` 对装载过程也很重要，逻辑上，该函数依赖具体的文件系统，不能实现为抽象。而且该函数也不能保存在上述 *super_operations* 结构中，因为超级块对象和指向该结构的指针都是在调用 `get_sb()` 之后创建的。
    - `kill_super()` 在不再需要某个文件系统类型时执行清理工作。

2. 装载和卸载
   - 目录树的装载和卸载比仅仅注册文件系统复杂得多
   - 在现存目录树中装载新的文件系统必须执行的任务、用于描述装载点的数据结构
     - 使用 *mount* 命令可以查询目录树中各种文件系统的装载情况
     - 装载点是附接（装载）文件系统的位置；每个装载的文件系统都有一个本地根目录；在将文件系统装载到一个目录时，装载点的内容被替换为即将装载的文件系统的相对根目录的内容，前一个目录数据消失，直至新文件系统卸载才重新出现（在此期间旧文件系统的数据不会被改变，但是无法访问）
     - 装载可以嵌套
     - 每个装载的文件系统都对应于一个 *vfsmount* 结构的实例
      ```c
      struct vfsmount {
        struct list_head mnt_hash;
        struct vfsmount *mnt_parent;    // 装载点所在的父文件系统
        struct dentry *mnt_mountpoint;  // 装载点在父文件系统中的 dentry
        struct dentry *mnt_root;        // 当前文件系统根目录的 dentry
        struct super_block *mnt_sb;     // 指向超级块的指针
        struct list_head mnt_mounts;    // 子文件系统链表
        struct list_head mnt_child;     // 链表元素，用于父文件系统中的 mnt_mounts 链表
        
        char *mnt_devname;  // 设备名称，例如 /dev/dsk/hda1
        struct list_head mnt_list;
        struct list_head mnt_expire;  // 链表元素，用于特定于文件系统的到期链表中
        struct list_head mnt_share;   // 链表元素，用于共享装载的循环链表
        struct list_head mnt_slave_list;  // 从属装载的链表
        struct list_head mnt_slave;       // 链表元素，用于从属装载的链表
        struct vfsmount *mnt_master;      // 指向主装载，从属装载位于 master->mnt_slave_list 链表上

        struct mnt_namespace *mnt_ns;     // 所属的命名空间


        /* 
         * 我们把 mnt_count 和 mnt_expiry_mark 放置在 struct vfsmount 的末尾
         * 以便让这些频繁修改的字段与结构的主体处于两个不同的缓存行中
         * （这样在 SMP 机器上读取 mnt_flags 不会造成高速缓存的颠簸）
         */
        atomic_t mnt_count;
        int mnt_expiry_mark;  // 如果标记为到期，则其值为 true
      };
      ```
     - 在装载新的文件系统时，*vfsmount* 并不是唯一需要在内存中创建的结构。装载操作开始于超级块的读取。*file_system_type* 对象中保存的 `read_super()` 函数指针返回一个类型为 *super_block* 的对象，用于在内存中表示一个超级块。它是借助于底层实现产生的：
      ```c
      struct super_block {
        ...
      }
      ```
     - `mount` 系统调用，仅适用于在现存的 *root* 文件系统中装载一个新文件系统。函数解析略
      ```c
      // 入口点是 sys_mount
      sys_mount()
        // 复制装载选项
        do_mount()
          path_lookup()
          // 根据标志调用相应的装载函数
      ```
     - 装载 *root* 文件系统本身，其代码可以参考 `init/do_mounts.c` 中的 `mount_root()`
     - 共享子树：略
     - `umount` 系统调用：
      ```c
      // 入口点是 sys_umount
      sys_umount()
        __user_walk()
        do_umount()
          sb->s_op->umount_begin()
          // 设置了 MNT_DETACH 或所述装载项不再被使用 ?
            // 是 -> umount_tree()
            //       release_mounts()
            // 否 -> 返回 -EBUSY
      ```
     - 自动过期：内核提供了一些基础设施，允许装载自动过期。在任何进程或内核本身都未使用某个装载时，如果使用了自动过期机制，那么该装载将自动从 *vfsmount* 树中移除。当前 *NFS* 和 *AFS* 网络文件系统使用了该机制。所有子装载的 *vfsmount* 实例，如果被认为将自动到期，都需要使用 `vfsmount->mnt_expire` 链表元素，将其添加到链表中。
     - 伪文件系统：文件系统未必需要底层块设备支持。它们可以使用内存作为后备存储器（如 *ramfs* 和 *tmpfs*），或根本不需要后备存储器（如 *procfs* 和 *sysfs*），其内容是从内核数据结构包含的信息生成的。伪文件系统的例子包括：负责管理表示块设备的 *inode* 的 `bdev`，处理管道的 *pipefs*，处理套接字的 *sockfs*。所有这些都出现在 `/proc/filesystems` 中，但不能装载。

#### 文件操作
1. 查找 *inode*
   - `struct nameidata` 结构用来向查找函数传递参数，并保存查找结果
    ```c
    struct nameidata {
      struct dentry *dentry;
      struct vfsmount *mnt;
      struct qstr last;
      unsigned int flags;
      ...
    };
    ```
   - 内核使用 `path_lookup()` 函数查找路径或文件名
   - `do_lookup()` 的实现，略
   - `do_follow_link()` 的实现，略

2. 打开文件
   - `sys_open()`，略

3. 读取和写入
   - 略

### 标准函数
*VFS* 层提供的有用资源是用于读写数据的标准函数。这些操作对所有文件系统来说，在一定程序上是相同的。如果数据所在的块是已知的，则首先查询页缓存。如果数据并未保存在其中，则向对应的块设备发出读请求。如果对每个文件系统都需要实现这些操作，则会导致代码大量复制。

大多数文件系统在其 *file_operations* 实例中，都将 `read()` 和 `write()` 分别指向 `do_sync_read()` 和 `do_sync_write()` 标准例程（在此前的内核版本中，标准的读写操作分别是 `generic_file_read()` 和 `generic_file_write()`，但已经被现在的版本替代了）。

#### 通用读取例程
几乎所有的文件系统都使用库程序 `generic_file_read()` 来读取数据。它同步地读取数据，保证在函数返回到调用者时，所需数据已经在内存中。实现中，实际的读取操作委托给一个异步例程，然后等待该例程结束：
```c
// 简化后的：
ssize_t do_sync_read(struct file *filp, char __user *buf, size_t len, loff_t *ppos) {
  struct iovec iov = { .iov_base = buf, .iov_len = len };
  struct kiocb kiocb;
  ssize_t ret;

  init_sync_kiocb(&kiocb, filp);
  kiocb.ki_pos = *ppos;
  kiocb.ki_left = len;

  ret = filp->f_op->aio_read(&kiocb, &iov, 1, kiocb.ki_pos);

  if (-EIOCBQUEUED == ret) 
    ret = wait_on_sync_kiocb(&kiocb);
  *ppos = kiocb.ki_pos;
  return ret;
}
```

1. 异步读取
   - `generic_file_aio_read()`

2. 从映射读取
   - `do_generic_mapping_read()`

#### 失效机制
内存映射通常调用由 *VFS* 层提供的 `filemap_fault()` 标准例程来读取未保存在缓存中的页。
- 代码流程：略

#### 权限检查
`vfs_permission()` 是 *VFS* 层的标准函数，用于检查是否允许以指定的某种权限访问给定的 *inode*，该权限可以是 `MAY_READ`、`MAY_WRITE` 或 `MAY_EXEC`。

大多数系统依赖 `generic_permission()` （`vfs_permission()` -> `permission()` -> `generic_permission()`）。

### 小结
几乎每个资源都可以表示为一个文件。

文件是内核世界中非常重要的成员，文件的表示涉及了相当多的工作量。

虚拟文件系统位于内核的底层和用户层之间，它提供了各种抽象数据结构来表示文件和 *inode*，而真实文件系统的实现必须填充这些结构，使得应用程序无需考虑底层文件系统，总是可以使用同样的接口访问和操作文件。

文件系统如何装载到用户层应用程序可见的文件系统树中。

打开文件需要穿过文件系统树。

## 第十章-无持久存储的文件系统
### proc 文件系统
#### /proc 的内容
- 信息分类：
  - 内存管理
  - 系统进程的特征数据
  - 文件系统
  - 设备驱动程序
  - 系统总线
  - 电源管理
  - 终端
  - 系统控制参数

1. 特定于进程的数据
2. 一般性系统信息
3. 网络信息
4. 系统控制参数

#### 数据结构
1. *proc* 数据项表示
   - *proc* 文件系统中的每个数据项都由 *proc_dir_entry* 的一个实例描述，简要信息如下：
    ```c
    struct proc_dir_entry {
      unsigned int low_ino; // inode 编号
      unsigned short namelen; // 文件名长度
      const char *name; // 文件名
      mode_t mode;      // 类型 + 访问权限信息
      nlink_t nlink;
      uid_t uid;        // 文件所有者的用户 ID
      gid_t gid;        // 文件所有者的组 ID
      loff_t size;      // 按字节计算的文件长度
      struct inode_operations *proc_iops;
      const struct file_operations *proc_fops;
      get_info_t *get_info;
      struct module *owner;
      struct proc_dir_entry *next, *parent, *subdir;
      void *data;
      read_proc_t *read_proc;
      write_proc_t *write_proc;
      ...
    }
    ```
    - `uid` 和 `gid` 都设置为 0，这意味着 *root* 用户是几乎所有 *proc* 文件的所有者
    - `size`-由于 proc 数据项是动态生成的，所有文件的长度通常无法预先知道，在这种情况下，该值为 0

2. *proc inode*
   - 内核提供了一个数据结构，称之为 *proc_inode*，支持以面向 *inode* 的方式来查看 *proc* 文件系统的数据项：
    ```c
    struct proc_op {
      int (*proc_get_link)(...)
      int (*proc_read)(...)
    };

    struct proc_inode {
      struct pid *pid;
      int fd;
      union proc_op op;
      struct proc_dir_entry *pde;
      struct inode vfs_inode;
    };
    ```

#### 初始化
在使用 *proc* 文件系统之前，必须用 *mount* 装载它，而内核必须建立并初始化几个数据结构，以便在内核内存中描述该文件系统的结构。遗憾的是，`/proc` 的外观和内容随着平台和体系结构的变化很大。

初始化的差别主要在于创建 `/proc` 的子目录

入口函数：`proc_root_init(), fs/proc/root.c`

#### 装载 proc 文件系统
在内核内部用于描述 *proc* 文件系统结构和内容的数据已经初始化之后，下一步是将该文件系统装载到目录树中。

从用户空间系统管理员的角度来看，`/proc` 的装载几乎与非虚拟文件系统是等同的。唯一的区别在于，将一个适宜的关键字（通常是 *proc* 或 *none*）指定为数据源，而不是设备文件：
```bash
mount -t proc proc /proc
```

文件系统的 *file_system_type* 实例提供了如何读取对应文件系统超级块的一些信息。对于 proc 文件系统，该结构定义为：
```c
static struct file_system_type proc_fs_type = {
  .name = "proc",
  .get_sb = proc_get_sb,
  .kill_sb = kill_anon_super,
};
```

将特定于文件系统的超级块数据填充到一个 *vfsmount* 结构的实例中，使得新的文件系统能够集成到 *VFS* 树种。借助 `proc_fill_super()` 来填充一个 *super_block* 的新实例。

#### 管理 /proc 数据项
1. 数据项的创建和注册
   - 新数据项分两个步骤添加到 *proc* 文件系统。首先，创建 *proc_dir_entry* 的一个新实例，填充描述该数据项的所有需要的信息。然后，将该实例注册到 *proc* 的数据结构，使得外部能看到该数据选项。因为这两个步骤从来都不独立执行，所以内核提供辅助函数合并了这两个操作，使得可以快捷地创建新的 *proc* 数据项：
    ```c
    // 创建 proc_dir_entry 实例
    extern struct proc_dir_entry *create_proc_entry(const char *name, mode_t mode, struct proc_dir_entry *parent);

    // ... 初始设置

    // 注册到 proc 文件系统
    static int proc_register(struct proc_dir_entry *dir, struct proc_dir_entry *dp);
    ```
    - 注册任务分为三个步骤：
      - 生成一个唯一的 *proc* 内部编号，向数据项赋予身份。`get_inode_number()` 返回一个未使用的编号，用于为动态生成的数据项
      - 必须适当地设置 *proc_dir_entry* 实例的 `next` 和 `parent` 成员，将新数据项集成到 *proc* 文件系统的层次结构中
      - 如果此前 *proc_dir_entry* 的成员 `proc_iops` 或 `proc_fops` 为 `NULL` 指针，那么需要根据文件类型，适当地设置指向 *file_operations* 和 *inode_operations* 结构实例的指针。否则，使用原值即可。
   - 除了 `create_proc_entry()`，内核还提供了两个辅助函数创建新的 *proc* 数据项。这 2 个实际上都是 `create_proc_entry()` 的包装器例程：
    ```c
    static inline proc_dir_entry *create_proc_read_entry(const char *name, mode_t mode, struct proc_dir_entry *base,
      read_proc_t *read_proc, void *data);
    
    static inline proc_dir_entry *create_proc_info_entry(const char *name, mode_t mode, struct proc_dir_entry *base,
      get_inof_t *get_info);
    ```
   - 其他辅助函数，用于管理 *proc* 数据项：
     - `proc_mkdir()`-创建一个新目录
     - `proc_mkdir_mode()`-创建一个新目录，目录的访问权限可以显式指定
     - `proc_symlink()`-创建一个符号连接
     - `remove_proc_entry()`-从 *proc* 目录中删除一个动态生成的数据项
   - 内核源码包含了一个示例文件，在  `Documentation/DocBook/procfs_example.c` 中

2. 查找 *proc* 数据项
   - 按第八章的讨论，查找过程将在一定的时间到达 `real_lookup()`，该函数将调用 *inode_operations* 的 `lookup` 函数指针，根据文件名的各个路径分量，来确定文件名对应的 *inode*。
   - 对 *proc* 数据项的搜索从 *proc* 文件系统的装载点开始，通常是 `/proc`。在 *proc* 文件系统根目录的 *inode_operations* 实例中，其 `lookup` 指针指向了 `proc_root_lookup()` 函数。

#### 读取和写入信息
- 内核使用保存在 `proc_file_operations` 中的操作来读写常规 *proc* 数据项的内容：
  ```c
  static struct file_operations proc_file_operations = {
    .llseek = proc_file_lseek,
    .read = proc_file_read,
    .write = proc_file_write,
  };
  ```

#### 进程相关的信息
`proc_pid_lookup()` 负责打开 `/proc/<pid>` 中特定于 *PID* 的文件。该例程的目的在于，创建一个 *inode* 作为第一个对象，用于后续的特定于 *PID* 的操作。该 *inode* 表示了 `/proc/pid` 目录，其中包含了所有能够提供特定于进程的信息的文件。

1. *self* 目录
   - 可以明确地根据进程的 *PID* 来选择进程，但当前运行进程的数据只需要选择 `/proc/self` 目录即可获取，无需提供 *PID*，内核会自动判断当前运行的进程。
2. 根据 *PID* 进行选择

#### 系统控制机制
操作内核行为的传统方法是 *sysctl* 系统调用。

*Linux* 借助于 *proc* 文件系统，内核重排了所有的 *sysctl*，建立起一个层次结构，并导出到 `/proc/sys` 目录下。可以使用简单的用户空间工具来读取或操作这些参数。要修改内核的运行时行为，`cat` 和 `echo` 就足够了。

讲解 *sysctl* 机制的 *proc* 接口、讨论在内核中如何注册管理 *sysctl*

1. 使用 *sysctl*
2. 数据结构-`struct ctl_table`
3. 静态的 *sysctl* 表
4. 注册 *sysctl*
5. `/proc/sys` 文件操作

### 简单的文件系统
一个小的文件系统库 *libfs*，包含了实现文件系统所需的几乎所有要素，开发者只需要提供到其数据的一个接口，文件系统就完成了。

此外，还有一些以 *seq_file* 机制提供的标准例程可用，使得顺序文件的处理毫不费力。

如果开发者只是想向用户空间导出一两个值，而不想和现存的文件系统（如 *proc*）打交道，内核对此也提供了一种方案：*debugfs* 文件系统允许只用几个函数调用，就实现一个双向的调试接口。

#### 顺序文件
通过 *kprobe* 机制的实现，来说明顺序文件的思想。

1. 编写顺序文件处理程序
2. 与虚拟文件系统的管理

#### 用 libfs 编写文件系统

#### 调试文件系统
使用了 *libfs* 函数的一个特别的文件系统是调试文件系统 *debugfs*。

它向内核开发者提供了一种向用户层提供信息的可能方法。这些信息并不会编译到产品内核中。它只是开发新特性时的一种辅助手段。仅当内核编译时启用了 `DEBUG_FS` 配置选项，才会激活对 *debugfs* 的支持。因而向 *debugfs* 注册文件的代码，都会被 C 预处理条件语句包围，来检查 `CONFIG_DEBUG_FS`。

1. 示例
2. 编程接口

#### 伪文件系统
第八章中提到，内核支持伪文件系统，其中收集了一些相关的 *inode*，但不能装载，因而对用户层是不可见的。*libfs* 也提供了一个辅助函数，来实现这种特殊类型的文件系统。

内核使用了一个伪文件系统来跟踪表示块设备的所有 *inode*：
```c
static int bd_get_sb(...) {
  return get_sb_pseudo(...);
}

static struct file_system_type bd_type = {
  .name = "bdev",
  .get_sb = bd_get_sb,
  .kill_sb = kill_anon_super,
};
```
- *libfs* 提供的 `get_sb_pseudo()` 方法可以确保不能从用户空间装载该文件系统。

为使用伪文件系统，内核需要使用 `kern_mount()` 或 `kern_mount_data()` 装载它。

### sysfs
sysfs 是一个向用户空间导出内核对象的文件系统，它不仅提供了查看内核内部数据结构的能力，还可以修改这些数据结构。特别重要的是，该文件系统高度层次化的组织：*sysfs* 的数据项源自内核对象（*kobject*），而内核对象的层次化组织直接反映到了 *sysfs* 的目录布局中。由于系统的所有设备和总线都是通过 *kobject* 组织的，所以 *sysfs* 提供了系统的硬件拓扑的一种表示。

在许多情况下，使用了简短、易读的文本串来导出对象的属性，但通过 *sysfs* 与内核传递二进制数据的方法也被频繁采用。*sysfs* 已经称为老式的 *IOCTL* 机制的一种替代品。向内核发送神秘的 *ioctl* 通常需要一个 *C* 程序。与之相比，从/向 *sysfs* 文件读/写一个值要简单得多，一个简单的 *shell* 命令就足够了。另一个优点在于，一个简单的列表目录操作，就能获得可设置选项的一个概观。

类似于许多虚拟文件系统，*sysfs* 最初基于 *ramfs*。因而其实现使用了许多与内核其他部分不同的技巧。请注意，只要配置时启用了该特性，*sysfs* 总是编译到内核中，不可能将其生成为模块。*sysfs* 的标准装载点是 `/sys`。

内核源码中包含了一些有关 *sysfs* 的文档，包括 *sysfs* 与驱动程序模型的关系、*sysfs* 与 *kobject* 框架的关系，等等。该文档可以在 `Documentation/filesystem/sysfs.txt` 和 `Documentation/filesystems/sysfs-pic.txt` 中找到。

请注意 *kobject* 与 *sysfs* 之间的关联不是自动建立的。独立的 *kobject* 实例默认情况下并不集成到 *sysfs*。要使一个对象在 *sysfs* 文件系统中可见，需要调用 `kobject_add()`。但如果 *kobject* 是某个子系统的成员，那么向 *sysfs* 的注册是自动进行的。

#### 概述
略（与 *kobject*、总线、设备、驱动程序和类相关）

#### 数据结构
1. 目录项-`struct sysfs_dirent`
2. 属性-`struct attribute`

#### 装载文件系统
*mount* 系统调用最终将填充超级块的工作委托给 `sysfs_fill_super()`。

#### 文件和目录操作
- `sysfs_file_operations`：
  ```c
  const struct file_operations sysfs_file_operations = {
    .read = sysfs_read_file,
    .write = sysfs_write_file,
    .llseek = generic_file_llseek,
    .open = sysfs_open_file,
    .release = sysfs_release,
    .poll = sysfs_poll,
  };
  ```

1. 打开文件
   - 数据结构：为便于在用户层和 sysfs 之间实现交换数据，内核需要提供一些缓冲区。缓冲区表示为 `struct sysfs_buffer`。
   - 实现：`sysfs_open_file()`

2. 读写文件内容
   - 读取数据-`sysfs_read_file()`
   - 写入数据-`sysfs_write_file()`

3. 目录遍历
   - `sysfs_dir_node_operations->sysfs_lookup()`

#### 向 sysfs 添加内容
- 注册子系统

## 第十二章-网络
网络功能的实现是内核最复杂、牵涉最广的一部分。

与网络相关的头文件的数目巨大，使得内核开发者将这些头文件存储到一个专门的目录 `include/net` 中，而不是存储到标准位置 `include/linux`。

### 互联的计算机
计算机之间通信的相关问题：
- 如何建立物理连接？使用何种线缆？通信介质有哪些限制和特殊要求？
- 如何处理传输错误？
- 如何识别网络中的每一台计算机？
- 如果两台计算机通过其他计算机连接，那么二者之间的数据交换如何进行？如何查找最佳的路由？
- 如何打包数据，使之不依赖于特定计算机的特性？
- 如果一台计算机提供了几个网络服务，如何识别这些服务？
- ...

最“合理”的系统是：将问题分类，创建各种层来解决明确定义的问题，层间借助固定的机制进行通信。这种方法大大简化了实现、维护，尤其是调试。

### ISO/OSI 和 TCP/IP 参考模型
*ISO* (*International Organization for Standardization*，国际标准化组织)设计了一种参考模型，定义了组成网络的各个层。该模型由 7 层组成，称为 *OSI*（*Open Systems Interconnection*，开放系统互联）模型。

但对某些问题来说，划分为 7 层过于详细了。因此，实际上通常使用另一种参考模型，其中将 *ISO/OSI* 模型的一些层合并为锌层。该模型只有 4 层，因此其结构更为简单。这种模型称为 *TCP/IP* 参考模型，*IP* 表示 *Internet Protocol*（网际协议），而 TCP 表示 *Transmission Control Protocol*（传输控制协议）。当今因特网上的大部分通信都是基于该模型的。

- [ ] 图，TCP/IP 参考模型和 ISO/OSI 参考模型

各层的功能划分：
略

### 通过套接字通信

## 第十三章-系统调用
应用程序看到的内核是负责执行多种系统功能的一个大的例程集合。标准库是一个中间层，用于在不同的体系结构和系统之间，标准化并简化内核例程的管理。

从内核的视角看，情况当然有一点复杂，因为用户态和核心态之间有几个重大的不同。要特别注意两种模式下不同的虚拟地址空间和利用各种处理器特性的不同方式。另外，控制权如何在应用程序和内核之间来回传递，参数和返回值如何传递。本章将讨论这些问题。

系统调用用于从用户应用程序调用内核例程，以利用内核的一些专门的功能。

从用户空间切换到内核空间的机制、实现系统调用的基础设施、特别的实现方面的特性。

### 系统程序设计基础
大体上，系统程序设计主要是利用标准库进行工作，标准库提供了各种基本函数，用于开发应用程序。

通常会使用 *C* 语言编写系统程序，因为它与 *UNIX* 概念融合得做好，所有 *UNIX* 内核都是用 *C* 语言编写的，*Linux* 也不例外。

标准库不仅是实现内核系统调用的接口集合，其中也提供了许多其他完全在用户空间实现的函数。

通过考察操作系统的内部结构，如何来极大地提高程序的性能。根据内存管理子系统管理内存的方式，可以避免不必要的调页操作，并且能够最有效地使用系统的缓存和缓冲区。

#### 追踪系统调用
- 使用 *strace* 工具，将 *shead*（应用程序）发出的所有系统调用的列表写到 *log.txt* 中：
  ```bash
  strace -o log.txt ./shead
  ```
  - 能够在上述应用程序的 *C* 源码中找到对应系统调用的行，都以斜体显示。其他行对应的系统调用，是由编译时自动添加的一些代码生成的。
  - 其他的系统调用是由启动和运行应用程序所需的框架代码生成的。例如，C 标准库是动态映射到进程内存区的，其他调用，如 `old_mmap()` 和 `unmap()`，负责管理应用程序使用的动态内存区域。
  - 标准库的另外两个例程在内部使用了不同名的系统调用，以达到预期的效果：
    - `malloc()` 是用于在进程堆区域分配内存的标准函数。GNU 库的 `malloc()` 变体提供了一个额外的内存管理设施，以便有效使用由内核分配的内存空间。`malloc()` 在内部执行了 `brk()` 系统调用。
    - `printf()` 首先处理传递的参数，并用 `write()` 系统调用显示结果。

- 使用 *strace* 工具无需接触所跟踪应用程序的源码即可了解其内部结构和运作方式。

#### 支持的标准
*POSIX, Portable Operating System Interface for UNIX* 标准，*Linux* 和 *C* 标准库尽力遵循 *POSIX* 标准。

除了 *POSIX* 之外，还有其他标准，这些不是由某个委员会指定的，而是源于 *UNIX* 和类 *UNIX* 操作系统的开发。在 *UNIX* 的历史中，两条开发主线产生了两个独立系统：一个是 *System V*，另一个是 *BSD*。

只实现 *POSIX* 标准并不能建立一个完整的 *UNIX* 系统，除了名称之外，这种做法一文不值。*POSIX* 无非是一组接口的集合，其具体实现并不是强制性的，也不一定要归入到内核来实现。

#### 重启系统调用
考虑系统调用与信号的冲突：在一个进程执行系统调用时，向该进程发送一个信号，那么在处理时，二者的优先级如何分配。

可重启系统调用（*restartable system call*）：重新启动被中断的系统调用，直至该调用不再被信号中断。该技术称为重启（*restrating*）。

倘若系统调用被中断，*Linux*（和其他 *System V* 变体）返回 `-EINTR` 常数。用户空间应用程序必须明确检查所有系统调用的返回值，并在返回值为 `-EINTR` 时重新启动被中断的系统调用，直至该调用不再被中断。

*BSD* 内核将中断系统调用的执行并切换到用户态执行信号处理程序。在发生这种情况时，该系统调用不会有返回值，内核在信号处理程序结束后将自动重启该调用。因为该行为对用户应用程序是透明的，也不再需要重复实现对 `-EINTR` 返回值的检查和调用的重启。

*Linux* 通过 `SA_RESTART` 标志支持 *BSD* 方案，可以在安装信号处理例程时按需对具体信号指定该标志。*System V* 提议的机制用作默认方案，因为 *BSD* 机制偶尔会导致一些困难。

### 可用的系统调用
每个系统调用都通过一个符号常数标识，符号常数的定义是平台相关的，在 `<asm-arch/unistd.h>` 中指定。

按功能分类的系统调用：
1. 进程管理
   - `fork()`、`vfork()`、`clone()`：`clone()` 支持创建线程
   - `exit()`
   - 查询和设置进程的属性，如 *PID、UID* 等等：`setgid()`、`setuid()` 和 `getuid()` 等等
   - `personality()`：定义了应用程序的执行环境
   - `ptrace()`：使得能够跟踪系统，它是 *strace* 工具的基础
   - `nice()`：设置普通进程的优先级，它给进程分配的优先级在 -20 和 19 之间，随数值的升高优先级递减。只有 *root* 进程（或有 `CAP_SYS_NICE` 权限的进程）才能指定负的优先级值。
   - `setrlimit()`：用于设置一定的资源限制

2. 时间操作
   - `adjtimex()`：读取和设置基于时间的内核变量，以控制内核在时间方面的行为
   - `alarm()` 和 `setitimer()`：建立报警器和间隔定时器，将操作延迟到一个稍后的时间执行。`getitimer()` 读取设置。
   - `gettimeofday()` 和 `settimeofday()`
   - `sleep()` 和 `nanosleep()`
   - `time()` 和 `stime()`

3. 信号处理
  信号是在进程之间交换有限信息以及促进进程间通信的最简单（也最古老）的方法。*Linux* 不仅支持所有类 *UNIX* 系统所共有的经典信号，还支持实时信号，这与 *POSIX* 标准是一致的。
   - `signal()` 和 `sigaction()`：设置信号处理函数
   - `sigpending()`：检查进程当前是否有待决信号被阻塞
   - `sigsuspend()`：将进程置于等待队列上，直至某个特定（一组信号中的一个）的信号到达
   - `setmask()`：启用信号的阻塞机制，而 `getmask()` 返回所有当前阻塞信号的列表
   - `kill()`：用于向一个进程发送任何信号
   - 还有一组处理实时小信号的系统调用，但其对应的函数名带有前缀 *rt_*。例如，`rt_sigaction()` 设置一个实时信号处理程序。
   - 与传统信号机制相比，所有体系结构（即使是 32 位 *CPU*）都可以处理 64 个不同的实时信号。实时信号可以关联附加信息，这使得（应用）程序员的工作稍微容易些。

4. 调度
   - `setpriority()` 和 `getpriority()`
   - `sched_setscheduler()` 和 `sched_getscheduler()`：分别设置和查询调度类。
   - `sched_setparam()` 和 `sched_getparam()` 分别设置和查询进程的附加调度参数（当前，只使用了实时优先级的参数）
   - `sched_yield()`：自愿放弃 *CPU* 的控制权，即使进程当前仍然有 CPU 时间可用。

5. 模块
   - `init_module()`：添加一个新模块
   - `delete_module()`：从内核移除一个模块

6. 文件系统
  所有关于文件系统的系统调用都应用到 *VFS* 层的例程。从 *VFS* 层开始，各个调用转发到具体文件系统的实现，后者通常会访问块层。从资源和执行时间来衡量，此类系统调用的代价都很高。
   - 创建和修改目录结构：`chdir()`、`mkdir()`、`rmdir()`、`rename()`、`symlink()`、`getcwd()`、`chroot()`、`umask()` 和 `mknod()`
   - 文件和目录属性修改：`chown()` 和 `chmod()`
   - 处理程序内容：`open()`、`close()`、`read()`、`readv()`、`write()` 和 `writev()`、`truncate()` 和 `llseek()`
   - 读取目录结构：`readdir()` 和 `getdents()`
   - 创建和删除连接：`link()`、`symlink()` 和 `unlink()`；读取连接内容：`readlink()`
   - 文件系统的装载和卸载：`mount()` 和 `umount()`
   - 等待某些事件：`poll()` 和 `select()`
   - 装载一个新进程，替换旧的进程：`execve()`，在与 `fork()` 联合使用时，它会启动一个新的程序。

7. 内存管理：
   - 就动态内存管理而言，最重要的调用是 `brk()`，它修改进程数据段的长度。
   - `mmap()`、`mmap2()`、`mremap()`：执行内存映射、解除映射和重新映射操作。`mprotect()` 控制对虚拟内存中特定区域的访问；`madvice()` 提出对特定虚拟区域的使用建议。根据 `malloc()` 的实现，它在内部可以使用 `mmap()` 或 `mmap2()`。这是可行的，因为匿名映射允许建立没有文件作为后备存储的映射。与使用 `brk()` 相比，该方法更加灵活。
   - `swapon()` 和 `swapoff()` 分别启用和禁用外存储器设备上（附加）的交换区。

8. 进程间通信和网络功能
  因为“进程间通信（*IPC*）和网络”是比较复杂的问题，很容易臆断有大量相关的系统调用。但事实刚好相反，只有两个系统调用来处理所有可能的任务，但其中涉及了非常多的参数。
   - `socketcall()`：处理网络方面的问题，用于实现套接字抽象。它管理各种类型的连接和协议，总共实现了 17 种功能，通过 `SYS_ACCEPT`、`SYS_SENDTO` 等常数来区分。参数必须以指针形式传递，指向一个与函数类型相关的用户空间结构，其中保存了所需的数据
   - `ipc()` 用于处理计算机本地的连接，而不是通过网络建立的连接。该系统调用实现了 11 种功能，它使用固定数目的参数来从用户空间向内核空间传递数据。

9. 系统信息和设置
   - `syslog()`：向系统日志写入消息
   - `sysinfo()`：返回有关系统状态的信息，特别有关内存使用的统计量（物理内存、缓冲区、交换区）
   - `sysctl()`：用于“微调”内核参数。内核现在支持大量的动态可配置选项，可以使用 *proc* 文件系统读取和修改。

10. 系统安全和能力
    - `capset()` 和 `capget()`：负责设置和查询进程的能力
    - `security()`：是一个系统调用的多路分解器，用于实现 *LSM (Linux Security modules, Linux 安全模块)*

### 系统调用的实现

#### 系统调用的结构
用于实现系统调用的内核代码分为两个颇为不同的部分。系统调用执行的实际任务实现位一个 *C* 例程，与其余内核代码几乎没有差别。用于调用该例程的机制则充满了平台相关的特性，必须考虑大量细节，因为最终实现使用汇编语言代码是必然的。

1. 处理程序函数实现
   - 这些函数散布在内核中各处，因为这些函数都嵌入到了与其目的关系最为密切的代码中
   - 用于实现系统调用的处理程序函数，在形式上有如下几个共同的特性：
     - 每个函数的名称前缀都是 *sys_*，将该函数唯一地标识位一个系统调用，更精确地说，标识为一个系统调用的处理程序函数
     - 所有的处理程序函数都最多接受 5 个参数
     - 所有的系统调用（处理函数）都在核心态执行，不允许直接访问用户态的内存，都必须确保在进行实际读写操作之前目标内存区对内核必须是可用的
   - 核心态和用户态之间的切换，由特定于平台的内核代码执行，这与中断处理程序是无关的
   - 处理程序函数的实现总是简短而紧凑的，它通常会尽快将控制权传递给一个更通用的内核辅助函数。例如 `read()` 中大部分工作是由 `vfs_read()` 完成的。另外一种是系统调用（处理函数）充当多路分解器，将系统调用委派给执行不同任务的函数，一个典型的例子是 `socketcall()`。
   - 每个处理程序函数都包括了额外的限定符：`asmlinkage`，这不是 *C* 语言的标准成分。`asmlinkage` 是一个汇编语言宏，定义在 `<linkage.h>` 中，对大多数平台来说，它根本什么都不做。
   - [ ] 系统调用中各操作的时间顺序
2. 调用分派和参数传递
   - 系统调用由内核分配的一个编号唯一标识。所有的系统调用都由一处中枢代码处理，根据调用编号和一个静态表，将调用分派到具体的函数。传递的参数也由中枢代码处理，这样参数的传递独立于实际的系统调用。
   - 从用户态切换到核心态，以及调用分派和参数传递，都是由汇编语言代码实现的，这其中考虑了许多平台相关的特性。
   - 为容许用户态和核心态之间的切换，用户进程必须通过一条专用的机器指令，引起处理器/内核对该进程的关注，这需要 *C* 标准库的协助。内核也必须提供一个例程，来满足切换请求并关注技术细节。该例程不能在用户空间中实现，因为其中需要执行普通应用程序不允许执行的命令。
   - 参数传递：
     - 不同的平台使用不同的汇编语言方法来执行系统调用（细节可在 *GNU* 标准库的源码中找到，参考 `sysdeps/unix/sysv/linux/arch/syscall.S` 文件。特定平台所需的汇编语言代码可以在 *syscall* 标号下找到，这些代码为库其余部分提供了一个通用接口，可用于调用系统调用）。在所有平台上，系统调用都是通过寄存器直接传递的，对具体的处理程序函数而言，参数与寄存器之间的映射是精确定义的。还需要一个寄存器来定义系统调用的标号，将系统调用分派给匹配的处理程序函数。
     - 例如，*PowerPC* 处理器提供了一条汇编语言指令，称作 `sc` (*system call*)，该指令专门用于实现系统调用。寄存器 *r3* 保存系统调用编号，而参数保存在寄存器 *r4* 到 *r8* 中。
     - 在应用程序借助标准库切换到核心态后，内核面临的任务是查找与该系统调用匹配的处理程序函数，并向该处理函数提供传递的参数。`sys_call_table` 表中保存了一组指向处理程序例程的函数指针，可用于查找处理程序（在所有平台上）。因为该表是用汇编语言指令在内核的数据段中产生的，其内容因平台而不同。但原理总是同样的：内核根据系统调用编号找到表中适当的位置，由此获得所指向目标处理程序函数的指针。
   - 系统调用表
     - `sys_call_table`，通常可以在与处理器类型对应的目录下的 *entry.S* 文件中找到
     - 找到处理程序函数后，将保存在寄存器中的参数值复制到栈上，内核调用处理程序例程，并切换到系统调用处理中独立于处理器的部分。
   - 因为核心态和用户态使用两个不同的栈（第三章），系统调用参数不能像普通那样在栈上传递。在两个栈之间的切换，或者由进入核心态时调用的体系结构相关的汇编语言代码进行，或者在特权级别从用户态切换到核心态时由处理器自动进行。
3. 返回用户态
   - 返回值的语义：通常，系统调用的返回值有如下约定：负值表示错误，而正值（和 0）表示成功结束，这里使用了借助于预处理器在 `include/asm-generic/errno-base.h` 和 `include/asm-generic/errno.h` 中定义的符号常数。`<errno.h>` 文件中包含了几个额外的错误码，但这些是特定于内核的，用户应用程序从来不会看到；使用 *UNIX* 系统调用时可能出现的典型错误都列出在 `<errno-base.h>` 中。
   - 511 之前（含）的错误码用于一般性错误，内核相关的错误码使用 512 以上的值
   - 尽管错误码总是以负数形式返回，但这里给出的所有错误码都是正值。这是一个内核惯例，错误码定义为正值，但返回时增加负号。
   - *Linux* 使用 `long` 数据类型从内核空间向用户空间传输结果。根据使用的处理器类型不同，这可能是 32 或 64 位。如果返回比较大的数字，可能占据 `unsigned long` 的整个范围。如果分配的内存地址位于虚拟内存空间的顶部，`malloc()` 和 `long` 的情形就是如此。内核会将返回的指针解释为负数，因为它超出了 `unsigned long` 的正值范围，尽管系统调用成功结束，仍然会报告错误。因此，内核将小于 -511 的返回值都排除在错误码之外，可以正确地解释为成功的系统调用的（很大的）返回值。（能够返回到用户空间的错误码符号常数不会大于 511，即返回的错误码从 -1 到 -511）
   - 成功地结束系统调用还需要完成的工作，就是从核心态切换回用户态。返回值的传递方式，与调用时参数的传递方式类似。实现系统调用处理程序的 C 函数使用 `return` 将返回值放置在内核栈上。该值被复制到一个特定的处理器寄存器，标准库会处理该寄存器并将返回值传递给应用程序。

#### 访问用户空间
尽管内核尽可能保持内核空间和用户空间的独立，有些情况下，内核代码必须访问用户应用程序的虚拟内存。当然，这只在内核执行由用户应用程序发起的同步操作时才有意义，而不适用于任意进程进行的读或写访问。

因为如下两种原因，内核必须访问应用程序的地址空间：
1. 如果一个系统调用需要超过 6 个不同的参数，它们只能借助进程内存空间中的 *C* 结构实例来传递。系统调用将借助寄存器，将指向该结构实例的一个指针传递给内核。
2. 由系统调用的副效应产生的大量数据，不能通过返回值机制传递给用户进程。相反，必须通过指定的内存区交换该数据。当然，该内存区必须在用户空间中，使得用户应用程序能够访问。

在内核访问自身的内存区时，虚拟地址和物理内存页之间的映射总是存在的。但用户空间中的情况有所不同，这里页可能被换出，甚至可能尚未分配物理内存页。因而，内核不能简单地反引用用户空间的指针，而必须采用特定的函数，确保目标内存区已经在物理内存中。为确保内核遵守了这种约定，用户空间指针通过 `__user` 属性标记，以支持 *C check tools* 对源码的自动化检查。

#### 追踪系统调用
*strace* 工具用来追踪进程的系统调用，它使用了 `ptrace()` 系统调用。

`sys_ptrace()` 处理程序例程的实现是体系结构相关的，定义在 `arch/<arch>/kernel/ptrace.c` 中。

`ptrace()` 本质上是一个用于读取和修改进程地址空间中的值的工具，不能用于直接跟踪系统调用。只有从正确的位置提取所需的信息，才能跟踪进程并就进行的系统调用得出结论。

即使调试器 *gdb* 的实现也完全依赖于 `ptrace()`，`ptrace()` 不仅能用于跟踪系统调用，还提供了更多的选项。

```c
asmlinkage long sys_ptrace(long request, long pid, long addr, long data);
```
- `pid` 标识了目标进程，进程标识符根据调用者的命名空间来解释。跟踪者程序必须通过 `ptrace()` 将自身连接到目标进程，而且这可以在进程已经运行后进行（不仅能在进程开始时进行），*strace* 负责连接到进程，通常是使用 `fork()` 和 `exec()` 启动目标程序后立即进行。
- `addr` 和 `data` 向内核传递一个内存地址和附加信息。其语义因选择的操作而不同。
- 借助于符号常数，`request` 用于选择一个操作，由 `ptrace()` 执行。可用的选项如下（`man 2 ptrace` 查看具体信息）：`PTRACE_ATTACH`、`PTRACE_DETACH`...

1. 系统调用追踪
   - 使用 `ptrace()` 模拟 *strace* 的示例程序，略
2. 内核端实现
   - 入口函数 `sys_ptrace()`
   - 函数解析，略
     - 开始跟踪
     - `PTRACE_CONT` 和 `PTRACE_SYSCALL` 的实现
     - 停止跟踪
     - 读取和修改目标进程的数据

### 小结
从某种角度看，可以将内核视作一个综合性的库，它包含了各种可向用户层应用程序提供的功能，系统调用是应用程序与该库之间的接口。

系统调用是从用户态切换到核心态的同步机制，中断是两种状态之间的异步切换。

## 第十四章-内核活动
- 两种类型的中断：
  1. 硬件中断（hardware interrupt）：由系统自身和与之连接的外设自动产生。它们用于支持更高效地实现设备驱动程序，也用于引起处理器自身对异常或错误的关注，这些是需要与内核代码进行交互的。
  2. 软件中断（SoftIRQ）：用于有效实现内核中的延期操作。

- 与内核的其他部分相比，用于处理中断和系统调用相关部分的代码中，汇编和 C 代码交织在一起，以解决 C 语言无法独立处理的一些微妙问题。

- 内核经常需要一些机制，将某些活动延迟到未来的某个时间执行，或将活动置于某个队列上，在时间充裕时进行后续处理。

### 中断
- 在内核版本 2.4 及之前，中断实现的大量代码散布到各个特定于体系结构的组件中。在内核版本 2.6 时，引入了一个用于中断和 IRQ 的通用框架。各个平台只负责在最低层次上与硬件交互，所有其他功能都由通用代码提供。
- 内容：系统中断类型、中断运作方式、完成的任务、造成的问题。

#### 中断类型
- 类型
  1. 同步中断和异常：这些由 CPU 自身产生，针对当前执行的程序。
  2. 异步中断：由外部设备产生，可能发生在任意时间。不同于同步中断，异步中断并不与特定进程关联。为避免损害正在执行的进程，内核必须确保中断能够尽快处理完毕（通过缓冲数据），使得 CPU 时间能够返还给当前进程。这也是内核需要延期操作机制的原因。

- 两种类型的共性：
  如果 CPU 当前不处于核心态，则发起从用户态到核心态的切换。接下来，在内核中执行一个专门的例程，称为中断服务例程（interrupt service routine, 简称 ISR）或中断处理程序（interrupt handler）。

- 内核是否禁用带来的问题：否-死锁；是-ISR 处理时间一长，丢失一些对系统正确运作必不可少的中断。

- 内核为解决上述问题，将中断处理程序划分为两个部分：性能关键的前一部分在禁用中断时执行，而不那么重要的后一部分延期执行，进行所有次要的操作。

- 中断编号
  - 每个中断都有一个编号
  - 中断共享：必须硬件和内核同时支持才能使用该技术，因为必须要识别出中断来源于哪个设备。

#### 硬件中断 IRQ
- 以前，中断这个名词用得很不谨慎，用来表示由 CPU 和外部硬件发出的中断。明白的读者当然会注意到这陈述不大准确。中断不能由处理器外部的外设直接产生，而必须借助于一个称为中断控制器（interrupt controller）的标准组件来请求，该组件存在于每个系统中。外部设备（或其槽位），会有电路连接到用于向中断控制器发送中断请求的组件。控制器再将中断请求转发到 CPU 的中断输入。因为外部设备不能直接发出中断，而必须通过上述组件请求中断，所以这种请求更正确的叫法是 IRQ，或中断请求（interrupt request）。因为就软件而言，IRQ 和中断之间的差别不是那么大，这两个术语通常可以替换使用。在所指的语义清楚的情况下，这不成问题。
- 这里的一个要点涉及 IRQ 和中断的数目，它会影响到软件。对大多数 CPU 来说，都只是从可用于处理硬件中断的整个中断号范围抽取一小部分使用。抽取出的范围通常位于所有中断号序列的中部。中断控制器除了负责 IRQ 信号的电工处理之外，还会对 IRQ 编号和中断号进行一个“转换”。

#### 处理中断
- 在 CPU 得知发生中断后，它将进一步的处理委托给一个软件例程，该例程可能会修复故障、提供专门的处理或将外部事件通知用户进程。由于每个中断和异常都有唯一的编号，内核使用一个数组，数组项是指向处理程序函数的指针。相关的中断号根据数组项在数组中的位置判断。
- 中断的处理：
  - [ ] 图，中断的处理
  - 进入和退出任务：建立环境、调用处理程序、系统复原到中断之前的状态。调用中断处理程序前后的两部分，分别称为进入路径（entry path）和退出路径（exit path）。
  - 进入路径的一个关键任务是从用户态栈切换到核心态栈、必须保存用户应用程序当前的寄存器状态（这与调度器间用于上下文切换的机制是相同的）。
  - 在进入核心态时，只保存整个寄存器集的一部分。内核并不使用全部寄存器。例如，内核不使用浮点操作，因而并不保存浮点寄存器。平台相关的数据结构 pt_regs 列出了核心态可能修改的所有寄存器，它的定义考虑到了不同的 CPU 之间的差别。在汇编语言编写的底层例程负责填充该结构。
  - 在退出路径中，内核会检查下列事项：调度器是否应该选择一个新进程代替旧的进程；是否有信号必须投递到原进程。
  - 从中断返回之后，只有确认了这两个问题，内核才能完成其常规任务，即还原寄存器集合、切换到用户态栈、切换到适用于用户应用程序的适当的处理器状态（或切换到一个不同的保护环）。
  - C 语言代码和汇编语言代码之间的交互，对应的代码位于 arch/arch/kernel/entry.S 中。
- 术语中断处理程序（interrupt handler）的使用是可能引起歧义的。它用于指代 CPU 对 ISR（中断服务程序）的调用，包括了进入/退出路径和 ISR 本身。当然，如果只指代在进入路径和退出路径之间执行、由 C 语言实现的例程，将更为准确。
- 屏蔽（Masking，这个术语用于表示选择性地禁用一个或多个中断）只能短时间使用。ISR 必须满足如下两个要求：
  - 实现（特别是在禁用其他中断时）必须包含尽可能少的代码，以支持快速处理；
  - 可以在其他 ISR 执行期间调用的中断处理程序例程，不能彼此干扰。
- 通常，每个处理程序例程都可以划分为 3 个部分，具有不同的意义：
  - 关键操作必须在中断发生后立即执行，否则无法维持系统的稳定性或计算机的正确运作。在执行此类操作期间，必须禁用其他中断。
  - 非关键操作也应该尽快执行，但允许启用中断（因而可能被其他系统事件中断）。
  - 可延期操作不是特别重要，不必在中断处理程序中实现。内核可以延迟这些操作，在事件充裕时进行。
- 内核提供了 tasklet，用于在稍后执行可延期操作。

#### 数据结构
- 中断技术上的实现有两方面：汇编语言代码，与处理器高度相关，用于处理特定平台上相关的底层细节；抽象接口，是设备驱动程序及其他内核代码安装和管理 IRQ 处理程序所需的。这里主要关注第二方面。
- 为响应外部设备的 IRQ，内核必须为每个潜在的 IRQ 提供一个函数。该函数必须能够动态注册和注销。静态表组织方式是不够的，因为可能为设备编写模块，而且设备可能与系统的其他部分通过中断进行交互。
- IRQ 相关信息管理的关键点是一个全局数组，每个数组项对应一个 IRQ 编号。数组位置和中断号是相同的，很容易定位与特定的 IRQ 相关的数组项。IRQ 最终映射到哪个处理器中断，在这里是不相关的。数组定义：
  ```c
  // kernel/irq/handle.c
  struct irq_desc irq_desc[NR_IRQS] __cacheline_aligned_in_smp = {
    [0 ... NR_IRQS - 1] = {
      .status = IRQ_DISABLED,
      .chip = &no_irq_chip,
      .handle_irq = handle_bad_irq,
      .depth = 1,
    }
  };
  ```
  - IRQ 的最大可能数目是通过一个平台相关的常数 NR_IRQS 指定。大多数体系结构下，该常数定义在处理器相关的头文件 include/asm-arch/irq.h 中。
  - 不同处理器间及同一处理器家族内，该常数的值变化都很大，主要取决于辅助 CPU 管理 IRQ 的辅助芯片。
  - 最初，所有中断槽位都使用 handle_bad_irq 作为处理程序，该函数只对没有安装处理程序的中断进行确认。
  - 与上述简单例子相反，数组元素类型并不仅仅是函数指针。
- 通用的 IRQ 子系统，以统一的方式处理不同的中断控制器和不同类型的中断。基本上，它由 3 个抽象层组成：
  - [ ] 中断处理子系统的各部分及交互方式
  - high-level interrupt service routines，高层中断服务例程：针对设备驱动程序端（或其他内核组件）的中断，执行由此引起的所有必要的工作。例如，如果设备使用中断通知一些数据已经达到，那么高层 ISR 的工作应该是将数据复制到适当位置。
  - interrupt flow handling，中断流处理：处理不同的中断流类型之间的各种差别，如边沿触发（edge-triggering）和电平触发（level-triggering）。从内核的角度来看，电平触发更为复杂，因为在每个中断后，都需要将线路明确设备为一个特定的电势，表示“没有中断”。
  - chip-level hardware encapsulation，芯片级硬件封装：需要与产生中断的底层硬件直接通信。该抽象层可视为中断控制器的某种“设备驱动程序”。
- IRQ Desc 数据结构描述-全局数组中元素类型：
  ```c
  <irq.h>
  struct irq_desc {
    irq_flow_handler_t handle_irq;
    struct irq_chip *chip;
    void *handler_data;
    void *chip_data;
    struct irqaction *action;   /* IRQ 操作列表 */
    unsigned int status;        /* IRQ 状态 */

    unsigned int depth;         /* 嵌套停用 irq */
    unsigned int irq_count;     /* 用于检测错误的中断 */
    unsigned int irqs_unhandled;
    ...
    const char *name;
  } ____cacheline_internodealigned_in_smp;
  ```
  - 从内核中高层代码的角度来看，每个 IRQ 都可以由该结构完全描述。上述 3 个抽象层在该结构中表示如下：
  - 中断流处理由 handle_irq 提供。handle_data 可以指向任意数据。每当中断发生时，特定于体系结构的代码都会调用 handle_irq。该函数负责使用 chip 中提供的特定于控制器的方法，进行处理中断所必须的一些底层操作。用于不同中断类型的默认函数由内核提供。
  - action 提供了一个操作链，需要在中断发生时执行。由中断通知的设备驱动程序，可以将与之相关的处理程序函数放置在此处。
  - 流处理和芯片相关操作被封装在 chip 中。chip_data 指向可能与 chip 相关的任意数据。
  - name 指定了流处理程序的名称，将显式在 /proc/interrupts 中。对边沿触发中断，通常是“edge”，对电平触发中断，通常是“level”。
  - depth 有两个任务：可用于确定 IRQ 电路是启用的还是禁用的，正值表示禁用，而 0 表示启动。这个值相当于一个计数器，内核其余部分的代码每次禁用某个中断，则将对应的计数器加 1；每次中断被再次启用，则将计数器减 1。在 depth 归 0 时，硬件才能再次使用对应的 IRQ。这种方法能够支持对嵌套禁用中断的正确处理。
  - status 描述了 IRQ 的当前状态。<irq.h> 文件定义了各种常数，可用于描述 IRQ 电路当前的状态。每个常数表示位串中一个置位的标志位，只要不相互冲突，几个标志可以同时设置。IRQ 不仅可以在处理程序安装期间改变其状态，而且可以在运行时改变。
    - IRQ_DISABLED：
    - IRQ_INPROGRESS：
    - IRQ_PENDING
    - IRQ_MASKED
    - IRQ_PER_CPU
    - IRQ_LEVEL
    - IRQ_REPLAY
    - IRQ_AUTODETECT
    - IRQ_WAITING
    - IRQ_NOREQUEST
  - irq_count 和 irq_unhandled 提供了一些统计量，用于检测停顿和未处理，但持续发生的中断。后者通常称作假中断（spurious interrupt）。
- IRQ Chip 数据结构描述-IRQ 控制器抽象，它提供的函数用于改变 IRQ 的状态，这也是它们还负责设置 flag 的原因：
  ```c
  <irq.h>
  struct irq_chip {
    const char *name;
    unsigned int (*startup)(unsigned int irq);
    void (*shutdown)(unsigned int irq);
    void (*enable)(unsigned int irq);
    void (*disable)(unsigned int irq);

    void (*ack)(unsigned int irq);
    void (*mask)(unsigned int irq);
    void (*mask_ack)(unsigned int irq);
    void (*unmask)(unsigned int irq);
    void (*eoi)(unsigned int irq);
    void (*end)(unsigned int irq);
    void (*set_affinity)(unsigned int irq, cpumask_t dest);
    ...
    int (*set_type)(unsigned int irq, unsigned int flow_type);
    ...
  }
  ```
  - 该结构需要考虑内核中出现的各个 IRQ 实现的所有特性，因而，一个该结构的特定实例，通常只定义所有可能方法的一个子集。
  - startup：用于第一次初始化一个 IRQ。大多数情况下，初始化工作仅限于启用该 IRQ。因而，startup 函数实际上就是将工作转给 enable。
  - enable：激活一个 IRQ。换句话说，它执行 IRQ 由禁用状态到启用状态的转换。为此，必须向 I/O 内存或 I/O 端口中硬件相关的位置写入特定于硬件的数值。
  - disable 与 enable 相对应，用于禁用 IRQ。
  - shutdown：完全关闭一个中断源。如果不支持该特定，那么这个函数实际上是 disable 的别名。
  - ack：与中断控制器的硬件密切相关。在某些模型中，IRQ 请求的到达（以及在处理器的对应中断）必须显式确认，后续的请求才能进行处理。如果芯片组没有这样的要求，该指针可以指向一个空函数，或 NULL 指针。ack_and_mask 确认一个中断，并在接下来屏蔽该中断。
  - end：标记中断处理在流层次的结束。如果一个中断在中断处理期间被禁用，那么该函数负责重新启用此类中断。
  - eoi：现代的中断控制器不需要内核进行太多的流控制，控制器几乎可以管理所有事务。在处理中断时需要一个到硬件的回调，由 eoi 提供，eoi 表示 end of interrupt，即中断结束。
  - set_affinity：在多处理器系统中，可以使用该函数指定 CPU 来处理特定的 IRQ。这使得可以将 IRQ 分配给某些 CPU（通常，SMP 系统上的 IRQ 是平均发布到所有处理器的）。该方法在单处理器系统上没用，可以设置为 NULL 指针。
  - set_type 设置 IRQ 的流类型。该方法主要使用在 ARM、PowerPC 和 SuperH 机器上，其他系统不需要该方法，可以将 set_type 设置为 NULL。辅助函数 set_irq_type(irq, type) 是一个便捷函数，用于设置 irq 的 IRQ 类型。包括：IRQ_TYPE_RISING 和 IRQ_TYPE_FALLING 以及 IRQ_TYPE_EDGE_BOTH、IRQ_TYPE_LEVEL_HIGH 和 IRQ_TYPE_LEVEL_LOW。IRQ_TYPE_NONE 设置了一种未指定的类型。
  - 注意，内核为 irq_chip 定义了别名 hw_interrupt_type，这是为兼容 IRQ 子系统的前一版本。
- IRQ Action 数据结构描述-（注册的 irq 的表示）
  ```c
  <interrupt.h>
  struct irqaction {
    irq_handler_t handler;
    unsigned long flags;
    const char *name;
    void *dev_id;
    struct irqaction *next;
  }
  ```
  - handler：处理程序函数。在设备请求一个系统中断，而中断控制器通过引发中断将该请求转发到处理器的时候，内核将调用该处理程序函数。在考虑如何注册处理程序函数时，再仔细考察其参数的语义。但注意，处理程序的类型为 irq_handler_t，与流处理程序的类型 irq_flow_handler_t 显然是不同的。
  - name 和 dev_id 唯一地标识一个中断处理程序。name 是一个短字符串，用于标识设备，而 dev_id 是一个指针，指向在所有内核数据结构中唯一标识了该设备的数据结构实例。如果几个设备共享一个 IRQ，那么 IRQ 编号自身不能标识该设备，此时，在删除处理程序时，将需要上述信息。
  - flags：通过位图描述了 IRQ（和相关的中断）的一些特性，位图中的各个标志位可通过预定义的常数访问。<interrupt.h> 中定义了下列常数：
    - IRQF_SHARED：共享的 IRQ
    - IRQF_SAMPLE_RANDOM：如果 IRQ 对内核熵池（entropy pool）有共享，将设置该位
    - IRQF_DISABLED：表示 IRQ 的处理程序必须在禁用中断的情况下执行。
    - IRQF_TIMER：表示时钟中断
  - next：用于实现共享的 IRQ 处理程序。几个 irqaction 实例聚集到一个链表中。链表的所有元素都必须处理同一个 IRQ 编号（处理不同编号的实例，位于 irq_desc 数组中不同的位置）。在发生一个共享中断时，内核扫描该链表找出中断实际上的来源设备。
  - [ ] IRQ 管理涉及的各数据结构

#### 中断流处理
1. 设置控制器硬件：内核提供的一些标准函数，用于注册 irq_chip 和设置流处理程序：
  ```c
  <irq.h>
  int set_irq_chip(unsigned int irq, struct irq_chip *chip);
  void set_irq_handler(unsigned int irq, irq_flow_handler_t handle);
  void set_irq_chained_handler(unsigned int irq, irq_flow_handler_t handle);
  void set_irq_chip_and_handler(unsigned int irq, struct irq_chip *chip, irq_flow_handler_t handle);
  void set_irq_chip_and_handler_name(unsigned int irq, struct irq_chip *chip, irq_flow_handler_t handle, const char *name);
  ```
  - set_irq_chip 将一个 IRQ 芯片以 irq_chip 实例的形式关联到某个特定的中断。
  - set_irq_handler 和 set_irq_chained_handler 为某个给定的 IRQ 编号设置流处理程序。第二种变体表示，处理程序必须处理共享的中断。这会置位 irq_desc[irq]->status 中的标志位 IRQ_NOREQUEST 和 IRQ_NOPROBE：设置第一个标志，是因为共享中断是不能独占使用的；设置第二个标志，是因为在有多个设备的 IRQ 电路上，使用中断探测显然是个坏主意。两个函数在内部都使用了 __set_irq_handler，该函数执行一些合理性检查，然后设置 irq_desc[irq]->handle_irq。
  - set_chip_and_handler 是一个快捷方式，它相当于连续调用上述的各函数。_name 变体的工作方式相同，但可以为流处理程序指定一个名称，保存在 irq_desc[irq]->name 中。
2. 流处理
   - 类型：
    ```c
    <irq.h>
    typedef void fastcall (*irq_flow_handler_t)(unsigned int irq, 
      struct irq_desc *desc);
    ```
   - 不同的硬件需要不同的流处理方式，例如边沿触发和电平触发就需要不同的处理。内核对各种类型提供了几个默认的流处理程序。它们有一个共同点：每个流处理程序在其工作结束后，都要负责调用高层 ISR。handle_IRQ_event 负责激活高层的处理程序。
   - 边沿触发
     - 现在的硬件大部分采用的是边沿触发中断，默认处理程序实现在 handle_edge_irq 中。
     - 在处理边沿触发的 IRQ 时无须屏蔽，这与电平触发 IRQ 是相反的。
     - 通过设置 IRQ_INPROGRESS 标志，避免 handle_edge_irq 同时在不同的 CPU 上执行。
     - 通过设置 IRQ_PENDING 标志，内核能够记录还有另一个 IRQ 需要在稍后处理。
     - 在屏蔽该 IRQ（确保在 hanle_IRQ_event 执行期间只能发生一个中断）并通过 mask_ack_irq 向控制器发送一个确认后，处理过程可以放弃。因而第二个 CPU 可以恢复正常的工作，而第一个 CPU 将在稍后处理该 IRQ。
     - 如果 IRQ 被禁用，或没有可用的 ISR 处理程序，都会放弃处理。
     - [ ] handle_edge_irq 的代码流程图
   - 电平触发中断
     - 电平触发中断在处理时必须屏蔽
     - [ ] handle_level_irq 的代码流程图
   - 其他中断类型
     - 现代 IRQ 硬件只需要极少的流处理工作。只需在 IRQ 处理结束后调用一个芯片相关的函数 chip->eoi。此类型的默认处理程序是 handle_fasteoi_irq。它基本上等同于 handle_level_irq，除了只需在最后与控制器芯片交互。
     - 非常简单，根本不需要流控制的中断由 handle_simple_irq 管理。如果调用者想要自行处理流，也可以使用该函数。
     - 各 CPU IRQ，即 IRQ 只能发送到多处理器系统的一个特定的 CPU，由 handle_percpu_irq 处理。该函数在接收之后确认 IRQ，在处理之后调用 eoi 例程。其实现非常简单，因为不需要锁，根据定义代码只能在一个 CPU 上运行。

#### 初始化和分配 IRQ
讲述如何注册和初始化 IRQ。
1. 注册 IRQ
   - 由设备驱动程序动态注册 ISR 的工作，可以使所述的数据结构非常简单地进行。该函数原型在所有体系结构上都是相同的，因为对编写平台无关的驱动程序来说，这是一个绝对的先决条件。
    ```c
    // kernel/irq/manage.c
    int request_irq(unsigned int irq, irqreturn_t handler, 
      unsigned long irqflags, const char *devname, void *dev_id);
    ```
   - [ ] 图，request_irq 的代码流程
     - 内核首先生成一个新的 irqaction 实例，然后用函数参数填充其内容。所有进一步的工作都委托给 setup_irq 函数，它将执行下列步骤。
     - 如果设置了 IRQF_SAMPLE_RANDOM，则该中断将对内核熵池有所贡献，熵池用于随机数生成器 /dev/random。rand_initialize_irq 将该 IRQ 添加到对应的数据结构。
     - 由 request_irq 生成的 irqction 实例被添加到所属 IRQ 编号对应的例程链表尾部，该链表表头为 irq_desc[NUM]->action。在处理共享中断时，内核就通过这种方式来确保中断发生时调用处理程序的顺序与其注册顺序相同。
     - 如果安装的处理程序是该 IRQ 编号对应链表中的第一个，则调用 handler->startup 初始化函数（如果没有提供显式的 startup 函数，则只调用 chip->enable 来启用该 IRQ）。如果该 IRQ 此前已经安装了处理程序，则没有必要再调用该函数。
     - register_irq_proc 在 proc 文件系统中建立目录 /proc/irq/NUM。而 register_handler_proc 生成 proc/irq/NUM/name。接下来，系统中就可以看到对应的 IRQ 通道在使用。

2. 释放 IRQ
   - 与上述过程刚好相反。
   - 首先，通过硬件相关的函数 chip->shutdown 通知中断控制器该 IRQ 已经删除（如果没有提供显式的 shutdown 函数，则只调用 chip->disable 禁用该中断）。接下来将相关数据项从内核的一般数据结构中删除。辅助函数 free_irq 承担这些任务。可以在 kernel/irq/manage.c 中找到该函数。
   - 在 IRQ 处理程序需要删除一个共享中断时，IRQ 编号本身不足以标志该 IRQ。在这种情况下，为提供唯一标识，还必须使用前面讲述的 dev_id。内核扫描所有注册的处理程序的链表，直至找到一个匹配的处理程序（dev_id 匹配）。这时才能移除该项。

3. 注册中断
   - 前面所述的机制只适用于由系统外设的中断请求所引发的中断。但内核还必须考虑由处理器本身或用户进程中的软件机制所引发的中断。与 IRQ 相比，内核无需提供接口供此类中断动态注册处理程序。这是因为，所使用的编号在初始化时就是已知的，此后不会改变。中断和异常的注册在内核初始化时进行，其分配在运行时并不改变。
   - 平台相关的内核源码基本上没有共同点，各个平台之间最大的相似性就是文件名。arch/arch/kernel/traps.c 包含了用于中断处理程序注册的系统相关的实现。
   - 所有实现的结果都是这样：在中断发生时自动调用对应的处理程序函数。因为系统中断不支持中断共享，只需要建立中断号和函数指针之间的关联。
   - 通常，内核以下述两种方式之一来响应中断：
     - 向当前用户进程发送一个信号，通知有错误发生。
     - 内核自动修复错误，这对用户进程不可见。

#### 处理 IRQ
1. 切换到核心态
   - 基于每个中断之后由处理器自动执行的汇编语言代码
   - 实现位置：arch/arch/kernel/entry.S，其中通常定义了各个入口点，在中断发生时处理器可以将控制流转到这些入口点。
   - 只有那些最为必要的操作直接在汇编语言代码中执行。内核试图尽快返回到常规的 C 代码，因为 C 代码更容易处理。在 C 语言中调用函数时，需要将所需的数据（返回地址和参数）按一定的顺序放到栈上。在用户态和核心态之间切换时，还需要将最重要的寄存器保存到栈上，以便以后恢复。这两个操作由平台相关的汇编语言代码执行。
   - 在大多数平台上，控制流接下来传递到 C 函数 do_IRQ，其实现也是平台相关的。根据平台不同，该函数的参数或者是处理器寄存器集合，或者是中断号和指向处理器寄存器集合的指针：
    ```c
    // arch/arch/kernel/irq.c
    fastcall unsigned int do_IRQ(struct pt_regs regs)
    // arch/arch/kernel/irq.c
    unsigned int do_IRQ(int irq, struct pt_regs *regs)
    ```
    - pt_regs 用于保存内核使用的寄存器集合。各个寄存器的值被依次压栈（通过汇编语言代码），在 C 函数调用之前，一直保存在栈上。这些值并不是仅仅保存用于后续的使用，C 代码也可以读取这些值。
    - struct pt_regs 的定义是平台相关的，因为不同的处理器提供了不同的寄存器集合。pt_regs 中包含了内核使用的寄存器。其中不包括的寄存器，可能只能由用户态应用程序使用。

2. IRQ 栈
   - 只有在内核使用内核栈来处理 IRQ 的情况下，上面描述的情形才是正确的。但不一定总是如此。
   - 对常规的内核工作以及 IRQ 处理例程所需的空间来说，4KiB （8KiB 缩减到 4KiB）并不总是够用，因而引入了另外两个栈：
     - 用于硬件 IRQ 处理栈
     - 用于软件 IRQ 处理栈
   - 常规的内核栈对每个进程都会分配，而这两个额外的栈是针对各 CPU 分别分配的。在硬件中断发生时（或处理软中断时），内核需要切换到适当的栈。
    ```c
    // arch/x86/kernel/irq_32.c
    static union irq_ctx *hardirq_ctx[NR_CPUS] __read_mostly;
    static union irq_ctx *softirq_ctx[NR_CPUS] __read_mostly;

    // arch/x86/kernel/irq_32.c
    union irq_ctx {
      struct thread_info tinfo;
      u32 stack[THREAD_SIZE/sizeof(u32)];
    }
    ```
    - 如果启动了 4KiB 栈，则 THREAD_SIZE 定义为 4096，这确保了所要求的栈长度。该数据结构刚好能够放在一个页帧中，这也意味着，tinfo 中包含的线程信息，在栈上总是可用的。

3. 调用流处理程序例程
  do_IRQ (generic_handle_irq) 代码流程

4. 调用高层 ISR
   - 采用 handle_IRQ_event 来激活与特定 IRQ 相关的高层 ISR。
    ```c
    // kernel/irq/handle.c
    irqreturn_t handle_IRQ_event(unsigned int irq, struct irqaction *action);
    ```
   - handle_IRQ_event 可能执行下述操作
     - 如果第一个处理程序函数中没有设置 IRQF_DISABLED，则用 local_irq_enable_in_hardirq 启动（当前 CPU 的）中断。换句话说，该处理程序可以被其他 IRQ 中断。但根据流类型，也可能一直屏蔽刚处理的 IRQ。
     - 逐一调用所注册的 IRQ 处理程序的 action 函数。
     - ...
   - ...

5. 实现处理程序例程
   - in_interrupt 函数用于指明当前是否在处理中断；
   - 中断上下文与普通上下文的不同之处
     - 中断是异步执行的。因而从用户空间来看，处理程序例程并不是在一个明确定义的环境中执行。这种环境下，禁止访问用户空间，特别是与用户空间地址之间来回赋值内存数据的行为。
     - 中断上下文中不能调用调度器，因而不能自愿地放弃控制权。
     - 处理程序例程不能进入睡眠状态（其中调用的所有过程和函数，依此类推，都不能进入睡眠状态）。只有在外部事件导致状态改变并唤醒进程时，才能解除睡眠状态。但中断上下文中不允许中断，进程睡眠后，内核只能永远等待下去。因为也不能调用调度器，不能选择进程来执行。
   - ...

### 软中断
- 软中断使得内核可以延期执行任务。因为它们的运作方式与上文描述的中断类似，但完全是用软件实现的，所以称为软中断（software interrupt）或 softIRQ 是完全符合逻辑的。
- 内核借助于软中断来获知异常情况的发生，而该情况将在稍后由专门的处理程序例程解决。如上所述，内核在 do_IRQ 末尾处理所有待决软中断，因而可以确保软中断能够定期得到处理。
- 从一个更抽象的角度来看，可以将软中断描述为一种延迟到稍后时刻执行的内核活动。但尽管硬件和软件中断之间有明显的相似性，它们并不总是可比较的。
- 软中断机制的核心部分是一个表，包含 32 个 softirq_action 类型的数据项：
  ```c
  <interrupt.h>
  struct softirq_action {
    void (*action)(struct softirq_action *);
    void *data;
  };
  ```
  - 该数据结构的定义是体系结构无关的，而软中断机制的整个实现也是如此。除了处理的激活之外，没有利用处理器相关的功能或特性，这与普通的中断是完全相反的。
- 软中断必须先注册，然后内核才能执行软中断。open_softirq 函数即用于该目的。它在 softirq_vec 表中指定的位置写入新的软中断：
  ```c
  // kernel/softirq.c
  void open_softirq(int nr, void (*action)(struct softirq_action*), void *data) {
    softirq_vec[nr].data = data;
    softirq_vec[nr].action = action;
  }
  ```
- 各个软中断都有一个唯一的编号，这表明软中断是相对稀缺的资源，使用其必须谨慎。默认情况下，系统上只能使用 32 个软中断。但这个限制不会有太大的局限性，因为软中断充当实现其他延期执行机制的基础，而且也很适合设备驱动程序的需要。
- 只有中枢的内核代码才使用软中断。软中断只用于少数场合，这些都是相对重要的情况：
  ```c
  <interrupt.h>
  enum {
    HI_SOFTIRQ = 0,
    TIMER_SOFTIRQ,
    NET_TX_SOFTIRQ,
    NET_RX_SOFTIRQ,
    BLOCK_SOFTIRQ,
    TASKLET_SOFTIRQ,
    SCHED_SOFTIRQ,
  #ifdef CONFIG_HIGH_RES_TIEMRS
    HRTIMER_SOFTIRQ,
  #endif
  };
  ```
  - 两个用来实现 tasklet：HI_SOFTIRQ、TASKLET_SOFTIRQ
  - 两个用于网络的发送和接收：NET_TX_SOFTIRQ、NET_RX_SOFTIRQ
  - 一个用于块层，实现异步请求完成：BLOCK_SOFTIRQ
  - 一个用于调度器，以实现 SMP 系统上周期性的负载均衡：SCHED_SOFTIRQ
  - 定时器：TIMER_SOFTIRQ
  - 高分辨率定时器：HRTIMER_SOFTIRQ
- 软中断的编号形成了一个优先顺序
- raise_softirq(int nr) 用于引发一个软中断（类似普通中断）。该函数设置各 CPU 变量 irq_stat[smp_processor_id].__softirq_pending 中的对应比特位。该函数将相应的软中断标记为执行，但这个执行是延期执行。通过使用特定于处理器的位图，内核确保几个软中断（甚至是相同的）可以同时在不同的 CPU 上执行。
- 如果不在中断上下文调用 raise_softirq，则调用 wakeup_softirqd 来唤醒软中断守护进程，这是开启软中断处理的两个可选方法之一。

#### 开启软中断处理
有几种方法可以开启软中断处理，但这些都归结为调用 do_softirq 函数。

- [ ] do_softirq 的代码流程图
- __do_softirq:
  - 将原来的位图重置为 0。换句话说，清除所有软中断。这两个操作都是在（当前处理器上）禁用中断的情况下执行，以防其他进程对位图的修改造成干扰。
  - 后续代码是在允许中断的情况下执行。这使得在软中断处理程序执行期间的任何时刻，都可以修改原来的位图。
  - softirq_vec 中的 action 函数在一个 while 循环中针对各个待决的软中断被调用。
  - 在处理了所有标记出的软中断后，内核检查在此期间是否有新的软中断标记到位图中。要求在前一轮循环中至少有一个没有处理的软中断，而重启的次数没有超过 MAX_SOFTIRQ_RESTART（通常设置为 10）。如果是这样，则再次按序处理标记的软中断。这操作会一直重复下去，直至在执行所有处理程序之后没有新的未处理软中断为止。
  - 如果在 MAX_SOFTIRQ_RESTART 次重启处理过程之后，仍然有未处理的软中断，内核将调用 wakeup_softirqd 唤醒软中断守护进程。

#### 软中断守护进程
- 软中断守护进程的任务是，与其余内核代码异步执行软中断。为此，系统中的每个处理器都分配了自身的守护进程，名为 ksoftirqd。
- 内核中有两处调用 wakeup_softirqd 唤醒了该守护进程（唤醒之后将其放置到就绪进程的列表末尾，并不会马上执行）
  - 在 do_softirq 中
  - 在 raise_softirq_irqoff 末尾。该函数由 raise_softirq 在内部调用，如果内核当前停用了中断，也可以直接使用。
- 该守护进程的优先级为 19
- 在系统启动时用 initcall 机制调用 init 不久，即创建了系统中的软中断守护进程。在初始化之后，各个守护进程都执行以下无限循环：
  ```c
  // kernel/softirq.c
  static int ksoftirqd(void *__bind_cpu) {
  ...
    while (!kthread_should_stop()) {
      if (!local_softirq_pending()) {
        schedule();
      }
      __set_current_state(TASK_RUNNING);
      while (local_softirq_pend()) {
        do_softirq();
        cond_resched();
      }
      set_current_state(TASK_INTERRUPTIBLE);
    }
  ...
  }
  ```
  - cond_resched 确保在对当前进程设置了 TIF_NEED_RESCHED 标志的情况下调用调度器。这是可能的，因为所有这些函数执行时都启动了硬件中断。

### tasklet
软中断是将操作推迟到未来时刻执行的最有效的方法。但该延期机制处理起来非常复杂。因为多个处理器可以同时且独立地处理软中断，同一个软中断的处理程序例程可以在几个 CPU 上同时运行。对软中断的效率来说，这是一个关键，多处理器系统上的网络实现显然受惠于此。但处理程序例程的设计必须是完全可重入且线程安全的。另外，临界区必须使用自旋锁保护（或其他 IPC 机制），而这需要大量审慎的考虑。

tasklet 和工作队列是延期执行工作的机制，其实现基于软中断，但它们更易于使用，因而更适合于设备驱动程序（以及其他一般性的内核代码）。

tasklet 是“小进程”，执行一些迷你任务，对这些任务使用全功能进程可能比较浪费。

1. 创建 tasklet
   - 数据结构描述-tasklet_struct
    ```c
    <interrupt.h>
    struct tasklet_struct {
      struct tasklet_struct *next;
      unsigned long state;
      atomic_t count;
      void (*func)(unsigned long);
      unsigned long data;
    };
    ```
    - func 和 data: 
    - next
    - count: 用于禁用已经调度的 tasklet，如果其值不等于 0，在接下来执行所有待决的 tasklet 时，将忽略对应的 tasklet。
    - state：表示任务的当前状态，类似于真正的进程。但只有两个选项，分别由 state 中的一个比特位表示，这也是二者可以独立设置/清除的原因。
      - TASKLET_STATE_SCHED-在 tasklet 注册到内核，等待调度执行时，将设置 TASKLET_STATE_SCHED
      - TASKLET_STATE_RUN-表示 tasklet 当前正在执行。只在 SMP 系统上有用，用于保护 tasklet 在多个处理器上并行执行。

2. 注册 tasklet
   - tasklet_schedule 将一个 tasklet 注册到系统中：
    ```c
    <interrupt.h>
    static inline void tasklet_schedule(struct tasklet_struct *t);
    ```
   - 如果设置了 TASKLET_STATE_SCHED 标志位，则结束注册过程，因为该 tasklet 此前已经注册过了。否则，将该 tasklet 置于一个链表的起始，其表头是特定于 CPU 的变量 tasklet_vec。该链表包含了所有注册的 tasklet，使用 next 成员作为链表元素。
   - 在注册了一个 tasklet 之后，tasklet 链表即标记为即将进行处理。

3. 执行 tasklet
   - 因为 tasklet 基于软中断实现，它们总是在处理软中断时执行。
   - tasklet 关联到 TASKLET_SOFTIRQ 软中断。因而，调用 raise_softirq(TASKLET_SOFTIRQ)，即可在下一个适当的时机执行当前处理器的 tasklet。
   - 内核使用 tasklet_action 作为该软中断的 action 函数：
    ```c
    // kernel/softirq.c
    static void tasklet_action(struct softirq_action *a) {
    ...
      while (list) {
        struct tasklet_struct *t = list;
        list = list->next;
        if (tasklet_trylock(t)) {
          if (!atomic_read(&t->count)) {
            if (!test_and_clear_bit(TASKLET_STATE_SCHED, &t->state))
              BUG();
            t->func(t->data);
            tasklet_unlock(t);
            continue;
          }
          tasklet_unlock(t);
        }
      ...
      }
    }

    // interrupt.h
    static inline int tasklet_trylock(struct tasklet_struct *t) {
      return !test_and_set_bit(TASKLET_STATE_RUN, &(t)->state);
    }
    ```
    - 该函数首先确认特定于 CPU 的链表，其中保存了标记为将要执行的各个 tasklet。它接下来将表头重定向到函数局部的一个数据项，相当于从外部公开的链表删除了所有表项。
    - 因为一个 tasklet 只能在一个处理器上执行一次，但其他的 tasklet 可以并行运行，所以需要特定于 tasklet 的锁。在执行一个 tasklet 的处理程序函数之前，内核使用 tasklet_trylock 检查 tasklet 的状态是否为 TASKLET_STATE_RUN。换句话说，它是否已经在系统的另一个处理器上运行。如果对应比特位尚未设置，则设置该比特位。
    - 如果 count 成员不为 0，则该 tasklet 已经停用。在这种情况下，不执行相关的代码。
    - 执行完 tasklet 的处理程序函数，最后，使用 tasklet_unlock(t) 清除 tasklet 的 TASKLET_SCHED_RUN 标志位。
   - 如果在 tasklet 执行期间，有新的 tasklet 进入当前处理器的 tasklet 队列，则会尽快引发 TASKLET_SOFTIRQ 软中断来执行新的 tasklet。
   - 除了普通的 tasklet 之外，内核还是用了另一种 tasklet，它具有“较高”的优先级。除以下修改之外，其实现与普通的 tasklet 完全相同。较高优先级是指该软中断的处理程序 HI_SOFTIRQ 在所有其他处理程序之前执行，尤其是在构成了软中断活动主体的网络处理程序之前执行。
     - 使用 HI_SOFTIRQ 作为软中断，而不是 TASKLET_SOFTIRQ，相关的 action 函数是 tasklet_hi_action。
     - 注册的 tasklet 在 CPU 相关的变量 tasklet_hi_vec 中排队。这是使用 tasklet_hi_schedule 完成的。
   - 当前大部分声卡驱动程序都利用了这一选项，因为操作延迟时间太长可能损塞音频输出的音质。而用于高速传输的网卡也可以得益于该机制。

### 等待队列和完成量
等待队列（wait queue）用于使进程等待某一个特定事件发生，而无须频繁轮询。进程在等待期间睡眠，在事件发生时由内核自动唤醒。完成量（completion）机制基于等待队列，内核利用该机制等待某一操作结束。这两种机制使用得都比较频繁，主要用于设备驱动程序。

1. 等待队列
  - 每个等待队列都需要有一个队列头：
    ```c
    <wait.h>
    struct __wait_queue_head {
      spinlock_t lock;
      struct list_head task_list;
    };
    typedef struct __wait_queue_head wait_queue_head_t;
    ```
    - 等待队列可以在中断时修改，在操作队列之前必须获得一个自旋锁 lock。
    - task_list 是一个双链表，用于实现双链表最擅长表示的结构，即队列。
  - 队列中的成员是以下数据结构的实例
    ```c
    <wait.h>
    struct __wait_queue {
      unsigned int flags;
      void *private;
      wait_queue_func_t func;
      struct list_head task_list;
    };
    typedef struct __wait_queue wait_queue_t;
    ```

### 工作队列
- 工作队列是将操作延期执行的另一种手段。因为它们是通过守护进程在用户上下文执行，函数可以睡眠任意长的时间，这与内核是无关的。在内核版本 2.5 开发期间，设计了工作队列，用以替换此前使用的 keventd 机制。
- 每个工作队列都有一个数组，数组项的数目与系统中处理器的数目相同。每个数组项都列出了将延期执行的任务。
- 对每个工作队列来说，内核都会创建一个新的内核守护进程，延期任务使用上文描述的等待队列机制，在该守护进程上下文中执行。
  - 新的工作队列通过调用 create_workqueue 或 create_workqueue_singlethread 函数来创建。前一个函数在所有 CPU 上都创建一个工作线程，而后者只在系统的第一个 CPU 上创建一个线程。两个函数在内部都使用了 __create_workqueue_key（另一种变体是 create_freezable_workqueue，用于创建能够与系统休眠进行良好协作的工作队列）：
    ```c
    // kernel/workqueue.c
    /* 
     * name - 表示创建的守护进程在进程列表中显示的名称
     * singlethread - 如果为 0，则在系统的每个 CPU 上都创建一个线程，
     *                否则只在第一个 CPU 上创建线程
     */
    struct workqueue_struct *__create_workqueue(const char *name, int singlethread)
    ```
- 所有推送到工作队列上的任务，都必须打包为 work_struct 结构的实例，从工作队列用户的角度来看，该结构的下述成员是比较重要的：
  ```c
  <workqueue.h>
  struct work_struct;
  typedef void (*work_func_t)(struct work_struct *work);

  struct work_struct {
    atomic_long_t data;
    struct list_head entry;
    work_func_t func;
  }
  ```
  - entry: 链表元素
  - func: 指向将延期执行的函数。函数有一个参数，是一个指针，指向用于提交该工作的 work_struct 实例。这使得工作函数可以获得 work_struct 的 data 成员，该成员可以指向与 work_struct 相关的任意数据。
  - data: 之前内核版本使用的数据类型为 void*，使用 atomic_long_t，以便将更多信息放入该结构，因为指针在所有支持的体系结构上都对齐到 4 字节边界，而前两个比特位保证为 0。通过 atomic_long_t，可以将前两个比特位作为标志位。
    ```c
    <workqueue.h>
    #define WORK_STRUCT_FLAG_MASK     (3UL)
    #define WORK_STRUCT_WQ_DATA_MASK  (~WORK_STRUCT_FLAG_MASK)
    ```
  - 当前只定义了一个标志：WORK_STRUCT_PENDING 用来查到当前是否有待决（该标志置位）的可延迟工作项。辅助宏 work_pending(work) 用来检查该标志。
  - 将 data 设置为原子数据类型，确保对该比特位的修改不会带来并发问题。
- 为简化声明和填充该结构的静态实例所需的工作，内核提供了 INIT_WORK(work, func) 宏，它向一个现存的 work_struct 实例提供一个延期执行函数。如果需要 data 成员，则需要稍后设置。
- 有两种方法可以向一个工作队列添加 work_struct 实例，分别是 queue_work 和 queue_work_delayed。
  - queue_work:
    ```c
    // kernel/workqueue.c
    int fastcall queue_work(struct workqueue_struct *wq, struct work_struct *work);
    ```
    - 将 work 添加到工作队列 wq，work 本身所指定的工作，其执行时间待定（在调度器选择该守护进程时执行）
  - queue_work_delayed：为确保排队的工作项将在提交后指定的一段时间内执行，需扩展 work_struct，添加一个定时器。queue_delayed_work 用于向工作队列提交 delayed_work 实例。它确保在延期工作执行之前，至少会经过由 delay 指定的一段时间（以 jiffies 为单位）
    ```c
    <workqueue.h>
    struct delayed_work {
      struct work_struct work;
      struct timer_list time;
    };

    // kernel/workqueue.c
    int fastcall queue_delayed_work(struct workqueue_struct *wq,
      struct delay_work *dwork, unsigned long delay);
    ```
    - 该函数首先创建一个内核定时器，它将在 delayed jiffies 之内超时。相关的处理程序接下来使用 queue_work，按通常的方式将工作添加到工作队列。
- 内核创建了一个标准的工作队列，称为 events。内核的各个部分中，凡是没有必要创建独立的工作队列者，均可以使用该队列。内核提供了以下两个函数，可用于将新的工作添加该标准队列：
  ```c
  // kernel/workqueue.c
  int schedule_work(struct work_struct *wk);
  int schedule_delayed_work(struct delay_work *dwork, unsigned long delay);
  ```

### 小结
内核可以用同步或异步方式激活。
- 同步：
- 异步：从硬件使用中断触发来异步激活内核

内核用于管理中断的通用数据结构，对不同 IRQ 类型实现流处理。

内核必须为 IRQ 提供服务例程，而服务例程即 ISR 的实现要颇为谨慎。最重要的一点是，必须使这些处理程序的执行尽可能快速，因而通常将工作划分为两部分：快速的上半部和低速的下半部（通常在中断上下文之外执行）。

内核提供了一些方法来将操作延迟到未来的时刻执行：软中断是硬件 IRQ 的软件等价物，tasklet 基于该机制。虽然它们能够使内核将工作推迟到稍后执行，但不允许休眠。然后对等待队列及其衍生机制而言，睡眠是允许的。

## 第十五章-时间管理
### 概述
#### 定时器的类型
在最初的发布版本中，定时器子系统只包括现在所谓的低分辨率定时器。在本质上，低分辨率定时器是围绕时钟周期展开的，该周期是每隔一定时间定期发生的，可以预定在某个周期激活指定的事件。

内核目前支持两类差别很大的定时器：
- 经典定时器（*classical timer*），也称低分辨率（*low resolution*）定时器或定时器轮（*timer wheel*）定时器
- 高分辨率定时器（*high-resolution timer*）

高分辨率定时器的一部分代码总是被编译到内核，但只有设置了编译选项 `HIG_RES_TIMERS`，才能提供比低分辨率定时器更高的精度。高分辨率定时器引入的框架，由低分辨率定时器重用（实际上，低分辨率定时器是基于高分辨率定时器的机制而实现的）

经典定时器是由固定的栅栏所框定的，而高分辨率时钟事件在本质上可以发生在任意时刻

高分辨率定时器内核文档：`Documentation/hrtimers.txt`

除分辨率之外，内核在术语上还会区分下面两种定时器：
- 超时
- 定时器

提供定时功能的几种硬件设备（时钟芯片）：
*PIT, Programmable Interrupt Timer*，即可编程中断计时器 - 经典时钟源
*APIT, Advance Programmable Interrupt Timer*, 高级可编程中断计时器 - 高分辨率时钟源

构成时间子系统的各个组件的概览
- [ ] 图
一些定时设备可以提供发生在任意的非规则时刻的事件，与周期性事件设备相对，它们称作单触发设备（*one-shot device*）

配置选项
- 内核中不仅有两个不同（但有关）的定时子系统，而且动态时钟特性会使情况更为复杂。通常，周期时钟在内核的整个生命周期内都是活动的
- 为节省能耗，内核允许配置动态时钟，它不需要周期信号
- 4 中配置方式：（高分辨率，低分辨率）x（动态时钟，周期时钟）

### 低分辨率定时器的实现
该部分的前提假设：内核使用周期时钟

#### 定时器激活与进程统计
定时器的时间基线，内核会使用处理器的时钟中断或其他周期性时钟源：*PIT* 和 *HPET, High Precision Event Time*（高精度事件定时器）

中断定期发生，每秒 *HZ* 次。*HZ* 由体系结构相关的预处理器符号定义，位于 `asm-arch/param.h`，通过 `CONFIG_HZ` 配置

在启动动态时钟时，也定义（并使用）了 *HZ*，因为它是许多计时任务的基本量

动态和周期时钟在表面上没有什么区别，只有在近乎于无事可做而且可以跳过一些时钟中断时，我们才能看到二者的差别

通常较高的 *HZ* 值使得系统具有更好的交互性和响应速度，特别是，每个时钟中断都会调用调度器。缺点是：因为定时器例程调用得更频繁，有更多的系统工作需要完成。在 *HZ* 增高时，内核的一般性开销也会随之增高。这样，较大的 *HZ* 值比较适合于桌面系统和多媒体系统；而较低的 *HZ* 值更适合于服务器和批处理器，这种场合下交互性属于次要因素

内核 2.6 系列的早期版本直接挂钩到中断，来开始定时器的激活和进程的统计，但随着通用时钟框架的引入，在一定程度上又增加了复杂性。
特定的体系结构上进行处理的方式，通常是在 `time_init()` 中设置，该函数在系统启动时调用以初始化基本的低分辨率计时设施。周期时钟设置为每秒运行 *HZ* 个周期。
*ISR*（*IA-32*/*AMD64* 上注册的中断处理程序）： `timer_interrupt()/timer_event_interrupt()`。它们都通过调用所谓的全局时钟的事件处理程序，来通知内核中通用的、体系结构无关的时间处理层。根据使用的计时模型不同，会采用不同的处理程序函数。无论如何，该处理程序都通过调用 `do_timer()` 和 `update_process_times()`，使得周期性低分辨率计时设施开始运作。
- `do_timer()`: 负责全系统范围的、全局性的任务：更新 `jiffies` 值，处理进程统计。在多处理器系统上，会选择一个特定的 *CPU* 来执行这个两个任务，而不涉及其他 *CPU*。
- `update_process_times()`: 需要由 *SMP* 系统上的每个 *CPU* 执行。除了进程统计之外，它还激活了所有注册的经典低分辨率定时器并使之到期，并向调度器提供时间感知。其中，定时器激活和到期通过调用 `run_local_timers()` 触发，该函数又引发了软中断 `TIMER_SOFTIRQ`，而其处理程序函数负责运行低分辨率定时器。

全局变量 `jiffies_64`（一个整型变量，在所有体系结构上都是 64 位）加 1。 
- `jiffies_64` 确定了系统启动以来时钟中断的准确数目
- 在停用动态时钟时，其值会定期增加。如果启用了动态时钟，那么自上次更新以来，可能已经过了多个时钟周期
- `jiffies` 是一个 `unsigned long` 类型变量。在 32 位处理器上为 4 字节，需要考虑回绕问题。在 64 位机器上，这两个变量是同义词。
- 内核使用一种技巧，来防止在两个不同的事件基准之间转换时引入精度损失。`jiffies` 和 `jiffies_64` 的低 32 位是重合的，指向同一块内存或同一个寄存器。根据底层体系结构的字节序不同，可能是 `jiffies_64` 的前 4 个或后 4 个字节序。

每个时钟中断都必须执行的其余操作：
- `update_wall_time()`: 更新 *wall time*，它指定了系统已经启动并运行了多长时间。该信息也是由 `jiffies` 机制提供，*wall clock* 从当前时间源读取时间，并据此更新 *wall clock*。与 `jiffies` 机制相反，*wall clock* 使用了人类可读格式（纳秒）来表示当前时间。
- `cacl_load()` 更新系统负载统计，确定在前 1 分钟、5 分钟、15 分钟内，平均有多少个就绪状态的进程在就绪队列上等待。该状态可以使用 `w` 命令输出。

#### 处理 jiffies
`jiffies` 提供了内核中一种简单形式的低分辨率时间管理方式。

在读取该变量的值，或比较按 `jiffies` 指定的时间时，需要注意：
- 由于 `jiffies_64` 在 32 位系统上是一个复合变量，它不能直接读取，而只能用辅助函数 `get_jiffies_64()` 访问。这确保在所有系统上都能返回正确的值。
- 比较时间（使用这些函数替代自行编写的比较函数，可防止所谓的 *off-by-one* 错误、使用这些函数可以确保正确处理 `jiffies` 计数器回绕问题）：
  ```c
  time_after(a, b) // 如果 a 在 b 之后，返回 true
  time_before(a, b) // 如果 a 在 b 之前，返回 true
  time_after_eq(a, b)
  time_before_eq(a, b)
  time_in_range(a, b, c) // 检查时间 a 是否包含在 [b, c] 时间间隔内，包含边界
  ```
  补充：*An off-by-one error (OBOE) is a common programming mistake where a loop, array, or string operation miscalculates the number of iterations or the index range, causing it to execute one extra or one fewer time than intended. This typically happens when developers misjudge the start or end of a sequence, especially in zero-based indexing languages like C, C++, or Java*
  - 尽管比较由 `jiffies_64` 给出的 64 位时间问题较少，但内核针对 64 位时间提供了上述函数。除了 `time_in_range()` 以外，只要向其他函数名增加 *_64* 后缀，即可得到处理 64 位时间值的函数变体。
- 时间换算
  ```c
  <jiffies.h>
  unsigned int jiffies_to_msecs(const unsigned long j);
  unsigned int jiffies_to_usecs(const unsigned long j);
  unsigned long msecs_to_jiffies(const unsigned int m);
  unsigned long usesc_to_jiffies(const unsigned int u);
  ```
  - 后续还有 `jiffies` 和 `struct timeval` 和 `struct timespec` 之间的转换函数

#### 数据结构
低分辨率定时器的实现，处理过程是由 `run_local_timers()` 发起的

定时器按链表组织，以下数据结构表示链表上的一个定时器：
```c
struct timer_list {
  struct list_head entry;
  unsigned long expires;

  void (*function)(unsigned long);
  unsigned long data;

  struct tvec_t_base_s *base;
};
```
- `entry`: 使用了一个双链表将注册的各个定时器彼此连接起来，`entry` 是链表元素
- `function`: 该函数在超时时调用
- `data`: 传递给回调函数 `function()` 的一个参数
- `expires`: 确定定时器到期的时间，单位是 `jiffies`
- `base`: 系统中的每个处理器对应于一个基元素，通过 `base` 指针确定定时器在哪个 *CPU* 上运行

宏 `DEFINE_TIMER(_name, _function, _expires, _data)` 用于声明一个静态的 *timer_list* 实例

在安装一个新定时器时使用了偏移量，而所有内核数据结构都使用的是绝对值（基于 `jiffies`），`timer_list->expires` 也使用绝对时间，而非偏移量

因为在定义时间间隔时程序员习惯于按秒而不是 *HZ* 单位来思考，内核提供了一个匹配的数据结构，还可以将其转换为 `jiffies` (当然，还可以反向转换)：
```c
struct timeval {
  time_t tv_sec;  // 秒
  suseconds_t tv_usec;  // 微妙
};
```
- `timeval_to_jiffies()` 和 `jiffies_to_timeval()` 函数用于在这种表示和 `jiffies` 值之间转换

另一种指定时间的可能方法是，使用纳秒而不是微妙：
```c
struct timespec {
  time_t tv_sec;
  long tv_nsec;
};
```
- `timespec_to_jiffies()` 和 `jiffies_to_timespec()` 函数可以在 `jiffies` 和 *timespec* 之间来回转换。

#### 动态定时器
内核需要数据结构来管理系统中注册的所有定时器（这些定时器可能分配给某个进程或内核本身），该结构必须容许快速而高效地检查到期的定时器，以免消耗太多 *CPU* 时间。毕竟，每个时钟中断都必须进行这样的检查。

1. 操作方式
   - 数据结构中不仅必须包含管理定时器所需的全部信息，而且它必须能够很容易地进行周期性的扫描，以便执行到期的定时器并删除
   - [ ] 图，用于管理定时器的数据结构
   - 内核创建了不同的组，根据定时器的到期时间进行分类。
   - 分类的基础是一个主数组，有 5 个数组项（称为组（*group*），有时又称为桶（*bucket*）），它们也都是数组。主数组的 5 个位置根据到期时间对定时器进行粗略的分类。
     - 第一组到期时间在 0 到 255（或 `2^8 - 1`）个时钟周期之间的所有定时器
     - 第二组包含了到期时间在 256 和 `2^(8 + 6) - 1` 个时钟周期之间的所有定时器
     - 第三组中定时器的到期时间范围从 `2^14` 到 `2^(8 + 2 * 6) - 1` 个时钟周期
     - 依次类推
     - 注意：上述以普通系统上桶的大小作为计算的基础，在内存较少的小型系统上，时间间隔有所不同
   - 每个组本身由一个数组组成，定时器在其中再次排序。
     - 第一个组有 256 个数组项，每个位置表示 0 到 255 个时钟周期之间一个可能的到期时间。如果系统中有几个定时器的到期时间相同，它们通过一个标准的双链表连接起来（链表元素为 *timer_list* 的 `entry` 成员）
     - 其余数组项是 64 个，但每个数组项包含的 *time_list* 的 `expires` 值不再只有一个，而是一个时间间隔，间隔的长度与组是相关的。如第二组是 `2^8=256`，第三组是 `2^14`，第四组是 `2^20`，依次类推。
   - 操作方式：
     - 内核主要负责关注第一组的定时器。为简单起见，假定每个数组都有一个计数器，存储了某个数组位置的编号。
     - 第一组中的索引项指向的数组元素，保存了稍后即将执行的各定时器的 *timer_list* 实例。每当遇到一个时钟中断时，内核都扫描该链表（具有相同的 `expires`），执行所有定时器函数，并将索引位置加 1。刚执行过的定时器则从数据结构移除。下一次发生中断时，将执行新的数组位置上的定时器，依次类推。在所有项都处理之后，因为这里的加法是模 256 的，因而索引将恢复到初始位置（位置 0）
     - 在第一组的索引位置恢复到初始位置 0 之后，会将第二组中一个数组项的所有定时器补充到第一组（正好 256 个，后续组别也可依此类推）
     - 后续各组的数组位置（上述提到的数组计数器）不再是每个时钟周期加 1，而是每 `256^(i - 1)` 个时钟周期加 1，其中 *i* 是组的编号
     - 示例：在第一组索引重置时，`jiffies` 系统计时器的值为 10000。在第二组的第一个数组项中，有一个定时器链表，各定时器分别在时钟周期 10001、10015、10015、10254 到期。这些定时器分别会定位到第一组的 1、15 核 254，在位置 15 会创建一个链表，包括两个指针，因为这两个定时器同时到期。在复制完成后，将第二组的索引位置加 1。
     - 为确定哪些定时器已经到期，内核无须扫描一个巨大的定时器链表，处理范围仅限于第一组中的一个数组项。因为该位置通常是空的或仅包含一个定时器，检查可以很快进行。偶尔从后续的各组向前复制数据甚至也不需要多少 *CPU* 时间，因为复制可以通过指针操作高效进行（内核无须复制内存块，而只需将指针设置为新值）。
2. 数据结构
   - 上述各组的内容是通过两个简单的数据结构生成的：
    ```c
    kernel/timer.c
    typedef struct tvec_s {
      struct list_head vec[TVN_SIZE];
    } tvec_t;

    typedef struct tvec_root_s {
      struct list_head vec[TVR_SIZE];
    } tvec_root_t
    ```
    - *tvec_root_t* 对应第一组，而 *tvec_t* 表示后续各组，它们的不同之处只只在于数组项的个数
    - `TVR_SIZE` 定义为 256；所有其他组的数组长度为 `TVN_SIZE`，默认值 64。缺乏内存的系统可设置配置选项 `BASE_SMALL`，在这种情况下，第一组有 64 个数组项，而其他各组的数组项为 16 个
   - 系统中的每个处理器都有自身的数据结构，来管理运行于其上的定时器。下列数据结构的一个各 *CPU* 实例，用作根数据项：
    ```c
    struct tvec_t_base_s {
      ...
      unsigned long timer_jiffies;
      tvec_root_t tv1;
      tvec_t tv2;
      tvec_t tv3;
      tvec_t tv4;
      tvec_t tv5;
    } ____cacheline_aligned_in_smp;
    ```
    - `timer_jiffies`: 记录一个时间点（单位为 `jiffies`），表示该结构中此前到期的定时器都已经执行。例如，如果该变量的值为 10500，那么内核就知道，`jiffies` 值为 10499 及之前到期的定时器都已经执行过了。通常 `timer_jiffies` 等于 `jiffies` 或比 `jiffies` 小 1。如果内核有一段时间无法执行定时器（系统负荷非常高），二者的差值可能会稍大一点。
3. 实现定时器处理
   - 对所有定时器的处理都由 `update_process_timesby()` 发起，它会调用 `run_local_timers()` 函数。该函数将使用 `raise_softirq(TIMER_SOFTIRQ)` 来激活定时器管理软中断，在下一个可能的时机执行。（因为软中断不能直接处理，所以也可能经过若干 `jiffies`，此期间内核没有处理器任何定时器。因而，有时候定时器可能激活较迟，但绝不可能过早激活）
   - `run_timer_softirq()` 用作上述软中断的处理函数，它会选择特定于 *CPU* 的 `struct tvec_t_base_s` 实例，并调用 `__run_timers()`。
   - `__run_timers()` 实现了上面描述的算法。其中索引位置根据 *base* 的 `timer_jiffies` 获取，为此定义了下列宏：
    ```c
    kernel/timer.c
    #define TVN_BITS  (CONFIG_BASE_SMALL ? 4 : 6)
    #define TVR_BITS  (CONFIG_BASE_SMALL ? 6 : 8)
    #define TVN_SIZE  (1 << TVN_BITS)
    #define TVR_SIZE  (1 << TVR_BITS)
    #define TVN_MASK  (TVN_SIZE - 1)
    #define TVR_MASK  (TVR_SIZE - 1)

    #define INDEX(N)  ((base->timer_jiffies >> (TVR_BITS + (N) * TVN_BITS)) & TVN_MASK)
    ```
    - 在小型系统上（通常是嵌入式系统）可以定义配置选项：*BASE_SMALL*，为各组分配较少的数组项，从而节省一些空间。定时器实现的其他方面不受该选项的影响
    - 第一组的索引位置可通过 `base->timer_jiffies & TVR_MASK` 计算，其他（第 `N` 组）的索引值通过上述宏定义 `INDEX(N)` 来计算（注，第二组的 `N = 0`）
  - 如果内核在过去错过了若干定时器，现在将处理上一个时间点（`base->timer_jiffies`）到当前时间（`jiffies`）之间到期的所有定时器
  - `cascade()` 函数用于从指定组取得定时器补充前一组

4. 激活定时器
   - 在安装新定时器时，必须区分这是来自内核自身的需求，还是用户空间应用程序的需求
   - 内核定时器的机制（用户定时器基于该机制建立）：
     - `add_timer()` 用于将一个完全设置好的 *timer_list* 实例插入到上述数据结构中：
      ```c
      <timer.h>
      static inline void add_timer(struct timer_list *timer);
      ```
     - `add_timer()` 将主要工作交由 `internal_add_timer()` 完成
     - 内核必须首先计算从现在开始，还有多少个时钟周期才能触发新定时器的超时例程，使用 `expires - base->timer_jiffies` 计算该值。定时器所在的组和组内位置可根据该值确定。现在只需要将新定时器添加到链表，它将置于链表的尾部，而 `run_timer_list()` 是从头开始处理的，因而实现了一个先进先出的机制。

### 通用时间子系统
虽然低分辨率定时器使用 `jiffies` 作为时间的基本单位，但高分辨率定时器使用人类的时间单位，即纳秒。

1 纳秒是一个精确定义的时间间隔，而一个 `jiffies` 或时钟周期的长度是根据内核配置而定的。

与经典定时器相比，高分辨率定时器对各体系结构的体系结构相关代码提出了更多要求。通用时间框架提供了高分辨率定时器的基础。

内核的第二个定时器子系统的核心实现可以在 `kernel/time/hrtimer.c` 中找到。作为高分辨率定时器集成的通用计时代码，位于 `kernel/time` 下的几个文件中。

#### 概述
- [ ] 图，通用时间子系统概述
- 对图的理解：
  - 每个 *CPU* 都有自身的时钟设备，它提供了 *CPU* 的局部时钟
  - 在所有的 *CPU* 中存在一个负责更新全局时钟，即 `jiffies`

子系统的各个组件和数据结构，这里涉及 3 种机制，形成了内核中任何与时间相关的任务的基础：
- 时钟源：由 `struct clocksource` 定义。时间管理的支柱，本质上每个时钟源都提供了一个单调增加的计数器，通用的内核代码只能进行只读访问。不同时钟源的精度取决于底层硬件的能力
- 时钟事件设备：由 `struct clock_event_device` 定义。向时钟增加了事件功能，在未来的某个时刻发生。由于历史原因，这种设备通常也称为时钟事件源（*clock event source*）
- 时钟设备：由 `struct tick_device` 定义。扩展了时钟事件源的功能，提供一个时钟事件的连续流，各个时钟事件定期触发。但可以使用动态时钟机制，在一定时间间隔内停止周期时钟。

两种时钟类型
- 全局时钟（*global clock*）：
  - 负责提供周期时钟，主要用于更新 `jiffies` 值
  - 在此前的内核版本中，此类型时钟在 *IA-32* 系统上是由 *PIT* 实现的，在其他体系结构上由类似芯片实现
  - 全局时钟的角色，由一个明确选择的局部时钟承担
- 每个 *CPU* 一个局部时钟（*local clock*）：
  - 用来进行进程统计、性能剖析和实现高分辨率定时器

高分辨率定时器只能工作于提供了各 *CPU* 时钟源的系统上。否则，处理器之间的大量通信将大大降低系统性能，这是高分辨率定时器的作用所不能弥补的。

在两种广泛使用的平台 *AMD64* 和 *IA-32*（*MIPS* 平台也受到影响）上发生的令人遗憾的问题，使得整体的概念复杂化。*SMP* 系统上的局部时钟基于 *APIC* 芯片。遗憾的是，这种时钟能否正确工作，取决于系统所处的电源模式。对于低功耗模式（确切地说，*ACPI* 模式 *C3*），将停用局部 *APIC* 定时器，因而无法作为时钟源使用。在这种电源管理模式下，系统全局时钟仍然处于工作状态，将用于周期性地激活信号，使之看起来仍然来自于原来的时钟源。这种规避方案称为广播机制。（由于广播需要 *CPU* 之间的通信，与专门的局部时钟源相比，这种解决方案要慢，且不那么精确。内核会自动将定时器由高分辨率模式切换回低分辨率模式。）

#### 配置选项
定时器实现受到几个配置选项的影响，在编译时间，有如下两种可能的选择：
1. 内核在联编时，可以选择支持或不支持动态时钟。如果启用了动态时钟，将设置预处理器常数 `CONFIG_NO_HZ`
2. 可以启用或禁用高分辨率定时器支持。如果要提供支持，将启用预处理器符号 `CONFIG_HIGH_RES_TIMERS`
由于这两个配置彼此独立，这导致时间和定时器子系统有 4 种不同配置。

另外，每个体系结构都需要进行一些配置选择。这些不受用户影响：
- `GENERIC_TIME` 表示体系结构支持通用时间框架
- `GENERIC_CLOCKEVENTS` 表示体系结构支持通用时钟时间
上述两个是支持动态时钟和高分辨率定时器的必要前提

- `CONFIG_TICK_ONESHOT` 用于支持时钟事件设备的单触发模式。如果启用了高分辨率定时器或动态时钟，会自动选中该选项。
- `GENERIC_CLOCKEVENTS_BROADCAST`-如果体系结构受困于省电模式的问题而需要广播，那么必须定义此项。当前该问题只影响 *IA-32*、*AMD64* 和 *MIPS*

#### 时间表示
通用时间框架使用数据类型 `ktime_t` 来表示时间值。无论在何种底层体系结构下，该类型都是一个 64 位量。
```c
<ktime.h>
typedef union {
  s64 tv64;
#if BITS_PER_LONG != 64 && !defined(CONFIG_RTIME_SCALAR)
  struct {
#ifdef __BIT_ENDIAN
    s32 sec, nsec;
#else
    s32 nsec, sec;
#endif
  } tv;
#endif
} ktime_t;
```
- 为减少 32 位机器上的工作量，该结构的定义确保两个 32 位值的排序能够被立即直接解释为一个 64 位值，显然这需要根据处理器的字节序不同而对字段进行不同的排序
- 在 64 位体系结构上只需简单的整数操作

内核定义了几个辅助函数来处理 `ktime_t` 对象：
```c
<ktime.h>
ktime_t timespec_to_ktime(const struct timespec ts)
ktime_t timeval_to_ktime(const struct timeval tv)
struct timespec ktime_to_timespec(const ktime_t kt)
struct timeval ktime_to_timeval(const ktime_t kt)
s64 ktime_to_ns(const ktime_t kt)
s64 ktime_to_us(const ktime_t kt)
```

#### 用于时间管理的对象
前面提到：内核中有 3 个对象管理着计时功能，即时钟源、时钟事件设备、时钟设备

1. 时钟源
   - 如何从机器中提供的各种时钟源获得时间值，内核为此定义了时钟源的抽象：
    ```c
    struct clocksource {
      char *name;
      struct list_head list;
      int rating;
      cycle_t (*read)(void);
      cycle_t mask;
      u32 mult;
      u32 shift;
      unsigned long flags;
      ...
    };
    ```
    - `list`: 用于将所有的时钟源连接到一个标准的内核链表上
    - `rating`: 衡量时钟的质量，每个时钟源都必须（诚实地）在 `rating` 中指定其质量。可能范围：（时间戳计数器, *time stamp counter, TSC*）
      - *1~99*: 质量非常差的时钟源，万不得已或启动期间
      - *100~199*: 可用于实际应用
      - *300~399*: 相当快且准确
      - *400~499*: 完美的时钟源
    - `read()`: 读取时钟周期的当前计数值
    - `mult` 和 `shift`: 并非所有时钟源的 `read()` 返回值都使用了统一的计时单位，因而需要分别转换为纳秒值：
      ```c
      <clocksource.h>
      static inline s64 cyc2ns(struct clocksource *cs, cycle_t cycles) {
        u64 ret = (u64)cycles;
        ret = (ret * cs->mult) >> cs->shift;
        return ret;
      }
      ```
    - 数据类型 `cycle_t` 定义为一个 64 位无符号整数，它不依赖于底层平台
    - `mask`: 如果时钟不提供 64 位时间值，那么 `mask` 指定了一个位掩码，用于选择适当的比特位。`CLOCKSOURCE_MASK(bits)` 宏用于针对给定的比特位数构建适当的掩码
    - `flags`: 若干标志。
      - `CLOCK_SOURCE_IS_CONTINUOUS` 表示一个连续时钟。置位-表示该时钟是自有振荡的，不能跳跃；未置位-可以丢失一些周期。如果时钟源要用于高分辨率定时器，该标志必须置位
   - 在启动期间，如果计算机确实没有提供更好的选择（在启动后，绝不会如此），内核提供了一个基于 `jiffies` 的时钟：
    ```c
    kenel/time/jiffies.c
    #define NSEC_PER_JIFFY ((u32)((((u64)NSEC_PER_SEC) << 8)/ACTHZ))

    struct clocksource clocksource_jiffies = {
      .name = "jiffies",
      .rating = 1,  /* 最低的 rating 值 */
      .read = jiffies_read,
      .mask = 0xffffffff, /* 32 位 */
      .mult = NSEC_PER_JIFFY << JIFFIES_SHIFT,
      .shift = JIFFIES_SHIFT
    };
    ```
    - `jiffies_read`: 无需与硬件进行交互，返回当前 `jiffies` 值
  - 在 IA-32 和 AMD64 机器上，时间戳计数器通常提供了最佳时钟
    ```c
    arch/x86/kernel/tsc_64.c
    static struct clocksource clocksource_tsc = {
      .name = "tsc",
      .rating = read_ts,
      .mask = CLOCKSOURCE_MASK(64),
      .shift = 22,
      .flags = CLOCK_SOURCE_IS_CONTINUOUS | CLOCK_SOURCE_MUST|VERIFY,
    }
    ```
    - `read_tsc()` 使用一些汇编程序代码从硬件读出当前计数器值
2. 使用时钟源
   - 首先，时钟源必须注册到内核： `clocksource_register()`，时钟源被添加到全局的 `clocksource_list, kernel/time/clocksource.c`，其中根据 `rating` 对所有可用的时钟源进行排序。
   - `select_clocksource()` 选择最佳时钟源，通常选择 `rating` 值最大的时钟源，但也可以从用户层通过 `/sys/devices/system/clocksource/clocksource0/current_clocksource` 指定优先选择的时钟源，内核将优先使用
   - `current_clocksource`，全局变量，指向当前最佳时钟源
   - `next_clocksource`，全局变量，指向一个 `struct clocksource` 实例，它比当前使用的时钟源更好。在注册一个新的最佳时钟源时，内核将自动切换到最佳时钟源
   - 为读取时钟计时，内核提供下列函数：
     - `__get_realtime_clock_ts()`：以一个指向 `struct timespec` 实例的指针为参数，读取当前时钟，转换结果，并保存到 *timespec* 实例
     - `getnstimeofday()`：`__get_realtime_clock_ts()` 的一个前端
3. 时钟事件设备
   - 数据结构：
    ```c
    <clockchip.h>
    struct clock_event_device {
      const char *name;
      unsigned int features;
      unsigned long max_delta_ns;
      unsigned long min_delta_ns;
      unsigned long mult;
      int shift;
      int rating;
      int irq;
      cpumask_t cpumask;
      int (*set_next_event)(unsigned long evt, 
        struct clock_event_device *);
      void (*set_mode)(enum clock_event_mode,
        struct clock_event_device *);
      void (*event_handler)(struct clock_event_device *);
      void (*broadcast)(cpumask_t mask);
      struct list_head list;
      enum clock_event_mode mode;
      ktime_t next_event;
    };
    ```
    - `set_next_event()` 和 `event_handler()`：时钟事件设备允许注册一个事件，在未来一个指定的时间点上发生。当与完备的定时器实现相比，它只能存储一个事件。`set_next_event()` 用于设置事件将要发生的时间，`event_handler()` 在事件实际发生时调用。
    - `name`: 将显示在 `/proc/timerlist` 中
    - `max_delta_ns` 和 `min_delta_ns`: 指定了当前时间和下一次事件的触发时间之间的差值，分别是最大和最小值（下一次事件允许发生在 `[t + min_delta_ns, t + max_delta_ns]` 内，`t` 表示当前时间）。时钟按各自的频率运作，而通用时间子系统则需要时间发生时刻的纳秒表示，辅助函数 `clockevent_delta2ns()` 用于将一个表示转换为另一个。
    - `mult` 和 `shift`: 乘数和位移数，用于在时钟周期和纳秒值之间进行转换
    - `event_handler()`: 指向的函数由硬件接口代码（通常是特定于体系结构的）调用，将时钟事件传递到通用的时间子系统层
    - `irq`: 指定了该事件设备使用的 *IRQ* 编号。只有全局设备才需要该编号，各 *CPU* 的局部时钟使用不同的硬件机制来发送信号，将 `irq` 设置为 1 即可
    - `cpumask` 指定了该事件设备所服务的 *CPU*。局部设备通常只负责一个 *CPU*
    - `broadcast()`: 广播实现所需的成员，它可以规避 *IA-32* 和 *AMD64* 系统上在省电模式下不工作的局部 *APIC* 设备
    - `rating` 的作用类似于时钟设备中相应的机制，时钟事件设备可以通过标称其精度来进行比较
    - `list`: 所有 `struct clock_event_device` 的实例都保存在全局链表 `clockevent_devices` 上。`clockevents_register_device()` 用于注册一个新的时钟事件设备，该函数将指定的设备置于上述全局链表上
    - `next_event` 存储了下一个事件的绝对时间
    - `features`: 每个事件设备的几个特性都可以根据 `features` 中存储的一个位串来判定。`<clockchips.h>` 中提供了若干常数：
      - `CLOCK_EVT_FEAT_PERIODIC`: 支持周期性事件（的时钟事件设备），即事件周期不断重复，无须通过对设备重新编程来显式激活事件
      - `CLOCK_EVT_FEAT_ONESHOT`: 表示时钟能够发出单触发事件，只发生一次。基本上，这刚好与周期性事件相反
    - `set_mode()`: 用来切换所需要的运行方式，即在周期模式和单触发模式之间切换
    - `set_next_event()`: 通用代码并不需要直接调用 `set_next_event()`，因为内核为此提供了辅助函数：`clockevents_program_event(struct clock_event_device *dev, ktime_t expires, ktime_t now)`，其中当前时间 `now` 可以通过 `ktime_get()` 获取
  - 在 *IA-32* 和 *AMD64* 系统上，全局时钟事件设备的角色最初由 *PIT* 承担，在 *HPET* 初始化之后，将接管该职责。为在 *x86* 系统上跟踪用于处理全局时钟事件的设备，采用了全局变量 `global_clock_event`，它指向当前使用的全局时钟设备的 *clock_event_device* 实例
  - 内核通常会对每个时钟硬件设备注册一个时钟设备和一个时钟事件设备。例如，考虑 *IA-32* 和 *AMD64* 系统上的 *HPET* 设备。该设备作为时钟源的功能汇集到 `clocksource_hpet`，而 `hpet_clockevent` 则是 *clock_event_device* 的一个实例
4. 时钟设备
   - 时钟设备是时钟事件设备的一个扩展：
    ```c
    <tick.h>
    struct tick_device {
      struct clock_event_device *evtdev;
      enum tick_device_mode mode;
    };

    enum tick_device_mode {
      TICKDEV_MODE_PERIODIC,
      TICKDEV_MODE_ONESHOT,
    };
    ```
    - *tick_device* 只是 `struct clock_event_device` 的一个包装器，增加了一个额外的字段，用于指定设备的运行模式
  - 内核仍然会区分全局和局部（各 *CPU*）时钟设备。局部设备汇集在 `tick_cpu_device` 中
  - 在注册一个新的时钟事件设备时，内核会自动创建一个时钟设备
  - 在 `include/time/tick-internal.h` 中还定义了如下几个全局变量
    - `tick_cpu_device` 是一个各 *CPU* 链表，包含了系统中每个 *CPU* 对应的 `struct tick_device` 实例
    - `tick_next_period` 指定了下一个全局时钟事件发生的时间（单位为纳秒）
    - `tick_do_timer_cpu` 包含了一个 *CPU* 编号，该 *CPU* 的时钟设备将承担全局时钟设备的角色
    - `tick_period` 存储了时钟周期的长度，单位为纳秒。它与 *HZ* 相对，后者存储了时钟的频率
  - 为设置一个时钟设备，内核提供了 `tick_setup_device()` 函数：
    ```c
    static void tick_setup_device(struct tick_device *td, 
      struct clock_event_device *newdev, int cpu, cpumask_t cpumask)
    ```
    - `td` 指定了将要设置的 *tick_device* 实例。它将绑定到时钟事件设备 `newdev`
    - `cpu` 表示该设备关联的处理器，用于限制只有特定的 *CPU* 才能使用该时钟设备
    - 在该设备第一次设置时（即，如果该时钟设备没有相关的时钟事件设备），内核执行如下两个操作
      - 如果没有选定时钟设备来承担全局时钟设备的角色，那么将选择当前设备来承担此职责，而 `tick_do_timer_cpu` 设置为当前设备所属的处理器编号。`tick_period` 是时钟周期，单位为纳秒，它是根据 *HZ* 值计算的
      - 该时钟设备设置为按周期模式工作
    - 如果当前没有启用了广播模式，则内核需要建立一个周期时钟。这取决于时钟设备是运行于周期模式还是单触发模式。这两种模式下，剩余的工作分别委托给 `tick_setup_periodic()` 或 `tick_setup_oneshot()`
    - 如果注册了一个新的时钟事件设备，能够利用该设备创建一个比当前更好的时钟设备，那么会自动调用该函数（`tick_setup_device()`）。虽然将优先选择更高质量的设备，但在新的、更精确的设备不支持单触发模式，而旧设备支持该模式的情况下，不会选择新设备。
    - 注意：即使时钟设备处于单触发模式，也并不意味着一定启用了动态时钟！例如，在高分辨率模式下，时钟总是基于单触发定时器实现的

根据所选择的配置，内核所需要处理的情形：
- 没有动态时钟的低分辨率系统，总是使用周期时钟。该内核不包含任何对单触发操作的支持
- 启用了动态时钟特性的低分辨率系统，以单触发模式使用时钟设备
- 高分辨率系统总是使用单触发模式，无论是否启用了动态时钟特性

所有系统最初都工作于低分辨率模式，未启用动态时钟，只有在必要的硬件初始化之后，系统才能切换到不同的特性组合

主要考虑低分辨率、周期时钟的情况：
- [ ] 图，高/低两种分辨率、是/否启用动态时钟特性的4种特性组合下，所使用的时钟事件和广播处理程序函数
`tick_setup_periodic()` 的解析：
- 如果时钟事件设备支持周期性事件，`tick_set_periodic_handler()` 将`tick_handle_periodic()` 安装为处理程序函数，而 `clockevents_set_mode()` 确保时钟事件设备以周期模式运行
- 如果时钟事件设备不支持周期事件，那么内核必须用单触发事件来设法应付过去。`clockevents_set_mode()` 将事件设备设置为该模式，此外，需要使用 `clockevents_program_event()` 来编程设置下一个事件
- 在两种情况下，时钟设备的下一事件发生时，都会调用处理程序函数 `tick_handle_periodic()`，它会继续调用 `tick_periodic()`。如果使用周期事件，则调用 `tick_periodic()` 之后直接返回；如果时钟事件设备以单触发模式运行，那么需要在这之后编程设置下一个时钟事件，此时会调用 `clockevents_program_event()`。如果该调用失败（是因为下一个时钟事件的时间已经过去），那么内核将循环调用 `tick_periodic()` 并尝试再次编程设置该事件（调用 `clockevents_program_event()`），直至成功为止

`tick_periodic()` 负责处理给定 *CPU* 上的周期时钟信号：
```c
kernel/time/tick_common.c
static void tick_periodic(int cpu);
```
- 如果当前时钟设备负责全局时钟，那么将调用 `do_timer()`（`do_timer()` 负责更新全局 `jiffies` 值，该值在内核的许多部分中，用作粗粒度的时间基准）
- 每个时钟处理程序都会调用 `update_process_times()` ，以及 `profile_tick()`（`profile_tick()` 用于程序性能剖析）

### 高分辨率定时器
高分辨率定时器与低分辨率定时器相比，有如下两个根本性的不同：
1. 高分辨率定时器按时间在一颗红黑树上排序
2. 它们独立于周期时钟。它们不使用基于 `jiffies` 的时间规格，而是采用了纳秒时间戳

由于低分辨率定时器的实现基于高分辨率机制，即使不启用高分辨率定时器，内核中也会联编（一部分）对高分辨率定时器的支持。当然，这种情况下系统只能够提供低分辨率定时功能。

高分辨率定时器框架中，并非普遍适用且实际上用于提供高分辨率功能支持的部分组件由预处理器符号 `CONFIG_HIGH_RES_TIMERS` 控制，只有在编译时通过该选项启用高分辨率支持的情况下，相关代码才会编译到内核中。而框架的通用部分总是会编译到内核中。（这意味着，即使只支持低分辨率定时器的内核也会包含高分辨率定时器框架的一部分）

#### 数据结构
高分辨率定时器可以基于两种时钟（称为时钟基础，*clock base*）
- 单调时钟（`CLOCK_MONOTONIC`），在系统启动时从 0 开始（始终单调运行）
- 实时时钟（`CLOCK_REALTIME`），表示系统的实际时间（可能发生跳跃，例如在系统时间改变时）

对系统中的每个 *CPU*，都提供两个时钟基础（单调时钟和实际时间）。每个时钟基础都有一个红黑树，来排序所有待决的高分辨率定时器（所有定时器都按过期时间在红黑树上排序）。如果定时器已经到期但其处理程序回调函数尚未执行，则从红黑树迁移到一个链表中

时钟基础的数据结构描述：
```c
<hrtimer.h>
struct hrtimer_clock_base {
  struct hrtimer_cpu_base *cpu_base;
  clockid_t index;
  struct rb_root active;
  struct rb_node *first;
  ktime_t resolution;
  ktime_t (*get_time)(void)
  ktime_t (*get_softirq_time)(void)
  ktime_t softirq_time;
#ifdef CONFIG_HIGH_RES_TIMERS
  ktime_t offset;
  int (*reprogram)(struct hrtimer *t, 
    struct hrtime_clock_base *b, ktime_t n);
#endif
};
```
- `hrtimer_cpu_base`: 指向该时钟基础所属的各 CPU 时钟基础结构
- `index`: 用于区分 `CLOCK_MONOTONIC` 和 `CLOCK_REALTIME`
- `rb_root`: 红黑树根节点，所有活动的定时器都在该树中排序
- `first`: 指向第一个到期的定时器
- `softirq_time` 和 `get_softirq_time()`: 对高分辨率定时器的处理，由相关的软中断 `HRTIMER_SOFTIRQ` 发起，`softirq_time` 存储了软中断发出的时间，而 `get_softirq_time()` 可用于获取该时间。如果未启用高分辨率模式，那么存储的时间将是粗粒度的。
- `get_time()`: 读取细粒度的时间。这对单调时钟是比较简单的（可直接使用由当前时钟源提供的值），但需要进行一些简单的算术操作，才能将该值转换为实际的系统时间。
- `resolution`: 表示该定时器的分辨率，单位为纳秒
- `offset`: 在调整实时时钟时，会造成存储在 `CLOCK_REALTIME` 时钟基础上的定时器的过期时间值与当前实际时间之间的偏差。`offset` 字段有助于修正这种情况，它表示定时器需要校正的偏移量。
- `reprogram()`: 用于对给定的定时器事件重新编程，即修改过期时间。

对每个 *CPU* 来说，都会使用以下数据结构建立两个时钟基础：
```c
<hrtimer.h>
struct hrtimer_cpu_base {
  struct hrtimer_clock_base clock_base[HRTIMER_MAX_CLOCK_BASES];
#ifdef CONFIG_HIGH_RES_TIMERS
  ktime_t expires_next;
  int hres_active;
  struct list_head cb_pending;
  unsigned long nr_events;
#endif
};
```
- `expires_next`: 将要到期的下一个事件的绝对时间
- `hres_active`: 表示高分辨率模式是否已经启用
- `cb_pending`: 在定时器到期时，将从红黑树迁移到一个链表中，表头为 `cb_pending`。在该链表上的定时器仍然需要进行处理，这由对应的软中断处理程序完成（要求对应的定时器允许在软中断的上下文中执行回调函数）
- `nr_events`: 用于跟踪记录时钟中断的总数

全局 *per-CPU* *hrtimer_cpu_base* 的定义：
```c
kernel/hrtimer.c
DEFINE_PER_CPU(struct hrtimer_cpu_base, hrtimer_bases) = {
  .clock_base = {
    {
      .index = CLOCK_REALTIME,
      .get_time = &ktime_get_read,
      .resolution = KTIME_LOW_RES,
    },
    {
      .index = CLOCK_MONOTONIC,
      .get_time = &ktime_get,
      .resolution = KTIME_LOW_RES,
    },
  }
};
```
- 系统初始化为低分辨率，数值为 `KTIME_LOW_RES`。`KTIME_LOW_RES` 表示在频率为 *HZ* 的周期时钟下，时钟信号间隔的长度，单位为纳秒
- `ktime_get()` 和 `ktime_get_read()` 都通过使用 `getnstimeofday()` 来获取当前时间。

高分辨率定时器数据结构描述：
```c
<hrtimer.h>
struct hrtimer {
  struct rb_node node;
  ktime_t expires;
  int (*function)(struct hrtimer *);
  struct hrtimer_base *base;
  unsigned long state;
#ifdef CONFIG_HIGH_RES_TIMERS
  enum hrtimer_cb_mode cb_mode;
  struct list_head cb_entry;
#endif
};
```
- `node`: 将定时器维持在上述红黑树中
- `base`: 指向定时器的基础
- `function()` 和 `expires`: 后者表示到期时间，前者是在定时器到期时调用的回调函数，有两个可能的返回值：
  - `HRTIMER_NORESTART`: 定时器无须重启（定时器将从系统消失）
  - `HRTIMER_RESTART`: 定时器必须重启。需要在回调函数中，设置定时器的到期时间为未来的某个时间点。辅助函数 `hrtimer_forward()` 可重置定时器，使之在 `now` 之后到期（`now` 通常设置为 `hrtimer_clock_base->get_time()` 的返回值）。确切的到期时间，需要在旧的到期时间上加上 `interval * n`，其中 `n` 作为返回值返回。
    ```c
    <hrtimer.h>
    unsigned long hrtimer_forward(struct hrtimer *timer, ktime_t new, 
      ktime_t now, ktime_t interval);
    ```
- `cb_entry`: 将定时器置于 `hrtimer_cpu_base->cb_pending` 链表上
- `cb_mode`: *hrtimer* 回调模式：
  ```c
  <hrtimer.h>
  /* 
   * HRTIMER_CB_SOFTIRQ - 回调函数必须在软中断上下文运行
   * HRTIMER_CB_IRQSAFE - 回调函数可能在硬件中断上下文运行
   * HRTIMER_CB_IRQSAFE_NO_RESTART - 回调函数可能在硬件中断上下文运行，不会重启定时器
   * HRTIMER_CB_IRQSAFE_NO_SOFTIRQ - 回调函数必须在硬件中断上下文运行，用于时钟仿真的特别模式
   */
  enum hrtimer_cb_mode {
    HRTIMER_CB_SOFTIRQ,
    HRTIMER_CB_IRQSAFE,
    HRTIMER_CB_IRQSAFE_NO_RESTART,
    HRTIMER_CB_IRQSAFE_NO_SOFTIRQ,
  };
  ```
- `state`: 定时器的当前状态
  - `HRTIMER_STATE_INACTIVE`: 表示不活动的定时器
  - `HRTIMER_STATE_ENQUEUE`: 在时钟基础上排队、等待到期的定时器
  - `HRTIMER_STATE_CALLBACK`: 表示当前正在执行定时器的回调函数
  - `HRTIMER_STATE_PENDING`: 定时器已经到期，正在回调链表上等待执行

高分辨率定时器的一个常见应用，就是使一个进程睡眠一段比较短的时间，时间的长度可以指定。内核为此提供了另一个数据结构：
```c
<hrtimer.h>
struct hrtimer_sleeper {
  struct hrtimer timer;
  struct task_struct *task;
};
```
- 内核使用 `hrtimer_wakeup()` 作为到期时调用的回调函数，以唤醒睡眠进程

#### 设置定时器
设置一个新的定时器
- `hrtimer_init()` 初始化一个 *hrtimer* 实例
  ```c
  <hrtimer.h>
  void hrtimer_init(struct hrtimer *timer, clockid_t which_clock,
    enum hrtimer_mode mode);
  ```
  - `which_clock` 是定时器绑定的目标时钟
  - `mode` 指定了是使用相对时间值（相对于当前时间），还是绝对时间值：
    ```c
    <hrtimer.h>
    enum hrtimer_mode {
      HRTIMER_MODE_ABS,
      HRTIMER_MODE_REL,
    };
    ```
- `hrtimer_start()` 设置定时器的到期时间，并启动定时器

取消一个设置好的定时器
- `hrtimer_cancel()`: 如果定时器当前正在执行因而无法停止，该函数一直等待执行完毕；如果定时器处于未激活状态，返回 0；如果定时器处于活动状态（`HRTIMER_STATE_ENQUEUE` 或 `HRTIMER_STATE_PENDING`），返回 1。
- `hrtimer_try_to_cancel()`: 如果定时器当前正在执行因而无法停止，返回 -1；如果定时器处于未激活状态，返回 0；如果定时器处于活动状态（`HRTIMER_STATE_ENQUEUE` 或 `HRTIMER_STATE_PENDING`），返回 1。

重启一个取消的定时器
`hrtimer_restart()`

#### 实现
未选择高分辨率定时器配置项时，高分辨率定时器的到期是由一个低分辨率时钟驱动的。这种情况下，仍然会采用高分辨率框架，但只以低分辨率运作。

即使高分辨率定时器支持已经编译到内核中，但在启动时只提供了低分辨率计时功能。因而，在考察高分辨率定时器的运行时，需要考虑两种可能性：基于具有高分辨率计时能力的适当时钟和具有低分辨率的时钟

1. 高分辨率模式下的高分辨率定时器
   - 假设：高分辨率时钟已经设置好且正在运行中，而向高分辨率模式的迁移工作已经完全完成
   - 在负责高分辨率定时器的时钟事件设备引发一个中断时，将调用 `hrtimer_interrupt()` 作为事件处理函数。
     - 该函数负责选中所有到期的定时器，或者将其转移到过期链表（如允许），或者直接调用定时器的处理函数。
     - 在对时钟事件设备重新编程（使得在一个待决定时器到期时可以引发一个中断）之后，将引发软中断 `HRTIMER_SOFTIRQ`
     - 在该软中断执行时，`run_hrtimer_softirq()` 负责执行到期链表上所有定时器的处理函数
     - `KTIME_MAX` 在该函数中可用于表示没有下一个定时器（因为它足够大）
     - 函数会遍历所有的时钟基础（单调时钟和实时时钟）
     - 在移除定时器时，如果需要重启（定时器回调函数返回 `HRTIMER_RESTART`），则通过 `enqueue_hrtimer()` 来完成该请求
     - 在已经选择了所有时钟基础的待决定时器之后，内核需要对时钟事件设备重新编程，以便在下一个定时器到期时引发中断。如果有定时器在回调链表上等待，则必须引发 `HRTIMER_SOFTIRQ`
2. 低分辨率模式下的高分辨率定时器
   - 对于没有提供高分辨率时钟的系统，高分辨率定时器的到期操作由 `hrtimer_run_queues()` 发起，该函数由高分辨率定时器软中断 `HRTIMER_SOFTIRQ` 调用（由于软中断处理在这种情况下是基于低分辨率定时器，因而该机制很自然不能提供任何高分辨率计时能力）。
   - 从低分辨率到高分辨率模式的切换也是由该函数开始的
   - 该函数通过 `hrtimer_get_softirq_time()` 将粗粒度的时间值保存到定时器基础之后，代码遍历所有时钟基础（单调时钟和实时时钟）并用 `run_hrtimer_queue()` 处理各个队列中的项

#### 周期时钟仿真
高分辨率模式下的时钟事件处理程序是 `hrtimer_interrupt()`，这意味着 `tick_handle_periodic()` 不再提供周期时钟信号，因而需要基于高分辨率定时器提供一个等效功能

`struct tick_sched` 是一个专门的数据结构，用于管理周期时钟相关的所有信息，由全局变量 `tick_cpu_sched` 为每个 *CPU* 分别提供了一个该结构的实例

在内核切换到高分辨率模式时，将调用 `tick_setup_sched_timer()` 来激活时钟仿真层。这将为每个 *CPU* 安装一个高分辨率定时器（`tick_cpu_sched->sched_timer`），该定时器的回调函数选择了 `tick_sched_timer()`。为避免所有 *CPU* 同时运行周期时钟处理程序的情况，内核分配了加速时间。
- [ ] 图，高分辨率模式下周期时钟处理程序的分配

`tick_sched_timer()` 解析：
- 如果当前执行该定时器的 *CPU* 负责提供全局时钟（在启动时，系统尚处于低分辨率模式，该职责已经分配到具体的 *CPU*），那么 `tick_do_update_jiffies64()` 将计算自上一次更新以来所经过的 `jiffies`数目（在这里该数目总是 1，因为现在不考虑动态时钟）。
- 在 `update_process_times()` 和 `profile_tick()` 中执行各 *CPU* 周期时钟任务时，需要计算下一个事件的时间，而 `hrtimer_forward()` 将据此对定时器进行设置。通过返回 `HRTIMER_RESTART`，定时器将自动重新进入队列，并在下一个时钟到期时激活。

#### 切换到高分辨率定时器
最初，高分辨率定时器并未启用，只有在已经初始化了适当的高分辨率时钟源并将其添加到通用时钟框架后，才能启用高分辨率定时器。但在最初，低分辨率时钟就已经提供了。

在低分辨率定时器活动时，高分辨率队列由 `hrtimer_run_queue()` 处理。在队列运行前，该函数将检查系统中是否存在适用于高分辨率定时器的时钟事件设备。如果有，则切换到高分辨率模式。
- 如果有一个支持单触发模式的时钟，而且其精度可以达到高分辨率定时器所要求的分辨率（即设置了 `CLOCK_SOURCE_VALID_FOR_HRES` 标志），那么 `tick_check_oneshot_change()` 将通知内核可以使用高分辨率定时器。
- 实际的切换由 `hrtimer_switch_to_hres()` 执行

### 动态时钟
周期时钟要求系统在一定的频率下，周期性地处于活动状态。因此，长时间的休眠是不可能的。

动态时钟改善了这种情况。只有在有些任务需要实际执行时，才激活周期时钟。否则，会临时禁用周期时钟。

对动态时钟技术的支持可以在编译时选择，启用此选项的系统也称为无时钟系统（*tickless system*），但这个名称是不完全准确的，因为即使在这种情况下，周期时钟运行的基础频率 *HZ* 仍然为时序提供了一个基本的度量工具。由于时钟可以根据当前的需要来激活或停用，因为“动态时钟”这个术语就很适用。

动态时钟切换的时机：
- 如果运行队列没有活动进程，内核将选择一个特别的 *idle* 进程来运行。此时，动态时钟机制将开始发挥作用。每当选中 *idle* 进程运行时，都将禁用周期时钟，直至下一个定时器即将到期为止。与此同时，*CPU* 可以进入不受打扰的睡眠状态。
- 在经过这样一段时间后，或者有中断发生时，将重新启用周期时钟

单触发时钟是实现动态时钟的先决条件。因为动态时钟的一个关键特性是可以根据需要来停止或重启时钟机制，纯粹周期性的定时器根本就不适用于该机制。

#### 数据结构
动态时钟需要根据使用的定时器分辨率高低来采用不同的实现。在两种情况下，其实现都是围绕以下数据结构进行：
```c
<tick.h>
struct tick_sched {
  struct hrtimer sched_timer;
  enum tick_nohz_mode nohz_mode;
  ktime_t idle_tick;
  int tick_stopped;
  unsigned long idle_jiffies;
  unsigned long idle_calls;
  unsigned long idle_sleeps;
  ktime_t idle_entrytime;
  ktime_t idle_sleeptime;
  ktime_t sleep_length;
  unsigned long last_jiffies;
  unsigned long next_jiffies;
  ktime_t idle_expires;
};
```
- `sched_timer` 定时器
- `nohz_mode` 当前运作模式
  ```c
  <tick.h>
  enum tick_nohz_mode {
    NOHZ_MODE_INACTIVE, // 周期时钟处于活动状态
    NOHZ_MODE_LOWRES,
    NOHZ_MODE_HIGHRES,
  };
  ```
- `idle_tick` 存储在禁用周期时钟之前，上一个时钟信号的到期时间
- `tick_stopped` 如果周期时钟已经停用，则 `tick_stopped` 为 1；否则，其值为 0
- `idle_jiffies` 存储周期时钟禁用时的 `jiffies` 值
- `idle_calls` 统计了内核试图停用周期时钟的次数
- `idle_sleeps` 统计了实际上成功停用周期时钟的次数
- `idle_sleeptime`(`idle_entrytime`?) 存储了周期时钟上一次禁用的准确时间（使用当前最佳的分辨率）
- `sleep_length` 存储了周期时钟将禁用的时间长度，即从时钟禁用起，到预定将发生的下一个时钟信号为止，这一段时间的长度
- `idle_sleeptime` 累计了时钟停用的总时间
- `next_jiffies` 存储了下一个定时器到期时间的 *jiffy* 值
- `idle_expires` 存储了下一个将到期的经典定时器的到期时间，单位不是 *jiffies*

*tick_sched* 中收集的统计信息通过 `/proc/timer_list` 导出到用户层

`tick_cpu_sched` 是一个全局 *per-CPU* 变量，提供了一个 `struct tick_sched` 实例。这是必须的，因为对时钟的禁用是按 *CPU* 指定的，而不是对整个系统指定。

#### 低分辨率系统下的动态时钟
考虑内核不使用高分辨率定时器，只提供低分辨率计时功能的情形。

定时器软中断调用 `hrtimer_run_queues()` 来处理高分辨率定时器队列，即使底层时钟事件设备只提供了低分辨率也是如此。再次强调，这并不会为定时器提供更好的分辨率。

1. 切换到动态时钟
   - `hrtimer_run_queues()` 调用 `tick_check_oneshot_change()` 来判断是否可以激活高分辨率定时器、检查是否可以在低分辨率系统上启用动态时钟，在如下两种情况都满足时，将调用 `tick_nohz_switch_to_nohz()` 来激活动态时钟（没有启用）
     - 提供了支持单触发模式的时钟事件设备
     - 未启用高分辨率模式
   - `tick_switch_to_oneshot()` 迁移动态时钟时将时钟事件设备设置为单触发模式，并安装一个适当的时钟定时器处理程序 `tick_nohz_handler()`
   - 动态时钟模式激活后，`struct tick_sched` 的 *per-CPU* 实例的 `nohz_mode` 字段将改变为 `NOHZ_MODE_LOWRES`。
   - 为使该机制进入运转，内核最后需要激活一个周期时钟定时器，使得该定时器在下一个时钟信号应该出现的时间到期
2. 动态时钟处理程序 `tick_nohz_handler()`
   - 任务
     - 执行时钟机制所需的所有操作
     - 对时钟设备重新编程，使得下一个时钟信号在适当的时候到期
   - 需要检查当前 *CPU* 是否负责全局时钟。
     - 调用 `tick_do_update_jiffies64()`
     - `update_process_times()` 和 `profile_tick()` 将接管局部时钟的职责
     - 如果一个 *CPU* 要进入比较长时间的休眠，不能继续负责全局时钟，需要撤销其职责。如果是这样，那么接下来如果有哪个 *CPU* 的时钟定时器处理程序被调用，该 *CPU* 必须承担该职责
   - 关键的是对时钟设备重新编程的部分。如果时钟机制在当前 *CPU* 已经停用，重新编程是没有必要的，*CPU* 可以进入完全的睡眠状态。
   - 注意，`next_event.tv64 = KTIME_MAX` 保证事件设备在最近的时间内不会到期，实际上可能永远都不会到期
   - 如果时钟定时器是活动的，那么 `tick_nohz_reprogram()` 会将该定时器设置为在下一个 *jiffy* 到期
3. 更新 `jiffies`
   - 全局时钟设备调用 `tick_do_update_jiffies64()` 来更新全局 `jiffies_64` 变量，它是低分辨率定时器处理的基础。在使用周期时钟时，这相对简单，因为每过一个 *jiffy*，都会调用该函数。
   - 在启用动态时钟时，可能出现这种情况：系统的所有 *CPU* 都处于 *idle* 状态，系统处于没有全局时钟的状态。`tick_do_update_jiffies64()` 需要考虑这种情况。

#### 高分辨率系统下的动态时钟
由于在内核使用高分辨率时，时钟事件设备以单触发模式运行，对动态时钟的支持比低分辨率情形更为容易实现

周期时钟是通过 `tick_sched_timer()` 仿真的，该函数也用于实现动态时钟。其中，包括两个要素：
1. 由于 *CPU* 可能放弃全局时钟的职责，该处理程序需要检查现在是否是这种情况
2. 在处理程序结束时，通常需要对时钟事件设备重新编程，使得下一个时钟信号在适当时候发生。如果时钟已经停止，则不必如此。

在高分辨率模式下初始化动态时钟模式，只需要在现存代码中修改一处。`tick_setup_sched_timer()` 在高分辨率下用于初始化时钟仿真层，如果在编译时启用动态时钟，会向该函数添加一小段代码。

#### 停止和启动周期时钟
动态时钟提供了暂时延迟周期时钟的框架，但内核仍然需要判断在何时停止和重启时钟。

很自然的一种做法是，在调度 *idle* 进程时停止时钟。动态时钟框架提供了 `tick_nohz_stop_sched_tick()`，用于停止时钟。

*idle* 进程实现（*idle* 运行函数）

1. 停止时钟-`tick_nohz_stop_sched_tick()`
   - 检查下一个定时器轮事件是否在一个时钟周期之后
   - 如果是这样，则重新编程时钟设备，忽略下一个时钟周期信号，直至有必要时才恢复。这将自动忽略所有不需要的时钟信号
   - 在 *tick_sched* 中更新统计信息
2. 重启时钟-`tick_nohz_restart_sched_tick()`

### 广播模式
在一些体系结构上，在某些省电模式启用时，时钟事件设备将进入睡眠。幸好，系统不是只有一个时钟事件设备，因此仍然可以用另一个可工作的设备来替换停止的设备。全局变量 `tick_broadcast_device` 是用于广播设备的 *tick_device* 实例

`tick_handle_periodic_broadcast()` 用作事件处理程序，它可以处理广播设备的周期模式和单触发模式，该处理程序在每个 `tick_periodic()` 之后都会激活

`tick_handle_periodic_broadcast()` 使用了 `tick_do_periodic_broadcast()`，该函数调用了当前 *CPU* 上不工作设备的 `event_handler()`。该处理程序不能区分对其调用是来自时钟中断还是广播设备，因而会像底层事件设备仍然正常工作一样去执行。

如果有更多不工作的时钟设备，那么 `tick_do_broadcast()` 将利用链表中第一个设备的 `broadcast()` 方法。对局部 *APIC* 来说，`broadcast()` 方法指向 `lapic_timer_broadcast()`。对所有与不工作时钟设备相关的 *CPU*，该方法都负责发送处理器间中断（*inter-processor interrupt, IPI*）`LOCAL_TIMER_VECTOR`。该中断的中断向量由内核设置为调用 `apic_timer_interrupt()`。其结果就是，时钟事件设备无法区分 *IPI* 和真正的中断，因而其效果与设备仍然处于工作状态是相同的。

处理器间中断是很慢的，因而不会提供高分辨率定时器所需的精度和分辨率。因而如果需要广播，内核总是切换到低分辨率模式。

### 定时器相关系统调用的实现
#### 时间基准
在使用定时器时，有 3 个选项可以区分如何计算经过的时间，或定时器所处的时间基准（也称时间域）。内核提供了以下形式的时间基准，在超时发生时，会触发各种信号：
- `ITIMER_REAL` 
  - 测量定时器激活以来实际流逝的时间
  - 定时器会继续运转，而不管系统是处于核心态还是用户态，后者使用该定时器的应用程序当前是否在运行
  - 在定时器到期时将发出 `SIGALRM` 类型的信号
- `ITIMER_VIRTUAL`
  - 只在定时器的拥有者进程在用户态消耗的时间内运行
  - 核心态，或处理器忙于另一个应用程序，消耗的时间将忽略
  - 定时器到期通过 `SIGVTALRM` 信号表示
- `ITIMER_PROF`
  - 计算进程在用户态和核心态消耗的时间
  - 系统其他进程消耗的时间将忽略
  - 定时器到期发送的信号是 `SIGPROF`
  - 主要用途是剖析应用程序的性能，以查找程序中计算最密集的片段，并据此进行优化

定时器的类型和时间间隔的长度，都必须在创建间隔定时器时指定

#### alarm 和 setitimer 系统调用
`alarm()` 安装 `ITIMER_REAL` 类型的定时器（实时定时器），而 `setitimer()` 不仅可用于安装实时定时器，还可以安装虚拟和剖析定时器。

这两个系统调用都结束于 `do_setitimer()`，两个系统调用的实现都依赖于 `kernel/itimer.c` 中定义的一种共同机制，实现围绕 `struct hrtimer` 展开

系统调用入口分别是 `sys_alarm()` 和 `sys_setitimer()`，它们都使用 `do_setitimer()` 来实际实现定时器

1. 对进程结构 *task_struct* 的扩展
   - 每个进程的 *task_struct* 实例都包含一个指针，指向一个 `struct signal_struct` 实例，其中包含了几个成员，用于容纳定时器所需的信息：
    ```c
    <sched.h>
    struct signal_struct {
      ...
      /* 进程的 ITIMER_REAL 定时器 */
      struct hrtimer read_timer;
      struct task_struct *tsk;
      ktime_t it_real_incr;

      /* 进程的 ITIMER_PROF 和 ITIMER_VIRTUAL 定时器 */
      cputime_t it_prof_expires, it_virt_expires;
      cputime_t it_prof_incr, it_virt_incr;
      ...
    }
    ```
2. 实时定时器
   - 在安装实时定时器（`ITIEMR_REAL`）时，首先必须保存可能存在的旧定时器的属性（在新定时器安装后，这些属性将返回到用户层），并用 `hrtimer_try_to_cancel()` 取消旧定时器。安装定时器时，会“覆盖”此前的定时器
   - 定时器的时间间隔保存在特定于进程的 `signal_struct->it_real_incr` 字段，如果该字段为零，那么该定时器不是周期性的，而只能激活一次
   - 在动态定时器到期时，不会有处理程序例程在用户空间执行。相反，系统会生成一个信号，导致调用信号处理程序，因而间接调用了一个回调函数。
   - 内核使用了回调处理程序 `it_real_fn()`，该函数向安装定时器的进程发送 `SIGALRM` 信号，但不会重新设备信号处理程序以使得信号具有周期性。相反，在进程上下文中投递信号时（确切地说，在 `dequeue_signal()` 中），会重新安装定时器，在用 `hrtimer_forward()` 前推到期时间之后，用 `hrtimer_start()` 重启定时器。

#### 获取当前时间
内核提供两种方法，可用于获取时间信息
1. 系统调用 `adjtimex()`，读取当前内核内部时间
2. 设备特殊文件 `/dev/rtc`

### 管理进程时间
*task_struct* 实例包含了两个与进程时间有关的成员：
```c
<sched.h>
struct task_struct {
  ...
  cputime_t utime, stime;
  ...
}
```
- `update_process_times()` 用于管理特定于进程的时间数据，从局部时钟调用
  - `account_process_tick()`: 使用 `account_user_time()` 和 `account_sys_time()` 来更新进程在用户态或核心态消耗的 *CPU* 时间，即 *task_struct* 的 `utime` 和 `stime` 成员。如果进程超出了 `Rlimit` 指定的 *CPU* 份额限制，那么还会每隔 1 秒发送 `SIGXCPU` 信号
  - `run_local_timers()`: 激活低分辨率定时器，或对其进行到期操作
  - `scheduler_tick()`: 用于进程调度
  - `run_posix_cpu_timers()`: 使当前注册的 *POSIX* 定时器开始运行

## 第十六章-页缓存和块缓存
性能和效率是内核开发中两个非常重要的因素。

内核不仅依赖于一个精巧的整体框架来规范各个部分之间交互，还需要一个功能广泛的缓冲和缓存框架来提高系统的速度。

缓冲和缓存利用一部分系统物理内存，确保最重要、最常使用的块设备数据在操作时可以直接从主内存获取，而无需从低速设备读取。物理内核还用于存储从块设备读取的数据，使得随后对该数据的访问可直接在物理内存进行，而无需从外部设备再次取用。

数据并非在每次修改后都立即写回，而是在一定的时间间隔后才进行回写，时间间隔的长度取决于多种因素，如空闲物理内存的容量、物理内存中数据的利用率，等等。单个的写请求会被收集起来，并打包进行，这在总体上花费的时间较少。因而，延迟写操作在总体上改进了系统性能。

缓存也有负面效应，内核必须审慎地采用（但缓存带来的好处，在很大程度上超出了带来的不利之处，因而缓存特性已经永久性地集成到了内核的结构中）：
- 通常，物理内存的容量比块设备小得多，因而只能缓存仔细挑选的部分数据
- 用于缓存的内存区不能分配给“普通”的应用程序数据。这减少了实际上可用的物理内存容量
- 如果系统崩溃，缓存包含的数据可能没有写回到底层的块设备。这造成了不可恢复的数据丢失

缓存是页交换或调页操作的逆操作，实现页交换时是利用低速的块设备来代替物理内存，而缓存牺牲物理内存使得不需要再块设备上进行低速操作。

此前各章讨论了内核提供的一些方法，用于缓存特定的结构。*slab* 缓存是一个内存到内存的缓存，其目的不是加速对低速设备的操作，而是对现存资源进行更简单、更高效的使用。*dentry* 缓存也用于减少对低速块设备的访问，但它无法推广到通用场合，因为它是专门用于处理单一数据类型的。

内核为块设备提供了两种通用的缓存方案：
1. 页缓存（*page cache*）
   - 针对以页为单位的所有操作
   - 一个主要的例子是内存映射技术。因为其他类型的文件访问也是基于内核中的这一技术实现的，所以页缓存实际上负责了块设备的大部分缓存工作。
2. 块缓存（*buffer cache*）
   - 以块为操作单位
   - 在进行 *I/O* 操作时，存取的单位是设备的各个块，而不是整个内存页
   - 尽管页长度对所有文件系统都是相同的，但块长度取决于特定的文件系统或其设置。因而，块缓存必须能够处理不同长度的块。

虽然缓冲区曾经是对块设备进行 *I/O* 操作的传统方法，但现在在这个领域中，缓冲区只用于支持规模很小的读取操作，这种场合下高级方法可能显得比较笨重。目前用于块传输的标准数据结构已经演变为 `struct bio`，用这种方式进行块传输更为高效，因为它可以合并同一请求中后续的块，加速处理的进行。
但表示对单个块的 *I/O* 操作时，缓冲区仍然是首选方法，即使底层 *I/O* 是通过 *bio* 进行的。特别是对经常按块读取元数据的系统，与其他更为强大的结构相比，缓冲区对此任务的处理更为容易。

在许多场合下，页缓存和块缓存是联合使用的。例如，一个缓存的页在写操作期间可以划分为不同的缓冲区，这样可以在更细的粒度下，识别出页被修改的部分。好处在于，在将数据写回时，只需要回写被修改的部分，无须将整页都传输回底层的块设备。

### 页缓存的结构
页缓存处理内存页。页缓存的任务在于，获得一些物理内存页，以加速在块设备上按页为单位执行的操作。

页缓存的运作方式对用户应用程序是透明的。

为支持对缓存页的使用，必须在代码中各个不同的位置加入“锚标”，与页缓存交互。无论目标页是否在缓存中，用于进程所要的操作总是必须执行。在缓存命令中，将快速地执行适当的操作（这也是缓存的目的所在）。倘若缓存失效，必须先从底层块设备读取所需的页，这花费的时间是比较长的。在页读入内存之后，页将被插入到缓存中，因而后续的访问可以快速进行。

在页缓存中搜索一页所花费的时间必须最小化，以确保缓存失效的代价尽可能低廉，因为在缓存失效时，进行搜索的计算时间实际上被浪费了。因而，页缓存设计的一个关键方面就是，对缓存的页进行高效的组织。

#### 管理和查找缓存的页
从大量数据的集合中快速获取单个数据元素的问题，并不是 *Linux* 内核特有的。对此用途而言，树数据结构是非常流行的，*Linux* 也采用了这种结构来管理页缓存中包含的页，称为基数树（*radix tree*）

树的各结点通过一个唯一的键来访问，键是一个整数。

树结点具备两种搜索标记（*search tag*）：`PAGECACHE_TAG_DIRTY` 和 `PAGECACHE_TAG_WRITEBACK`，用于指定给定页当前是否是脏的（即页的内容与后备存储器中的数据是不同的），或该页是否正在向底层块设备回写。重要的是，标记不仅对叶结点设置，还一直向上设置到根节点。如果某个层次 *n + 1* 的结点设置了某个标记，那么其在层次 *n* 的父节点也会获得该标记。这使得内核可以判断，在某个范围内是否有一页或多页设置了某个标记位。

页缓存中的标记，可用于快速判断某个区域中是否有脏页或正在回写的页，而无须扫描该区域中所有的页。但它们并不是用来代替 *page* 中的标志位的。

#### 回写修改的数据

## 第十七章-数据同步
内核为如何从缓存回写到磁盘，提供了几个选项，可分为如下两类：
1. 后台线程重复检查系统内存的状态，周期性地回写数据
2. 在系统缓存中脏页过多，而内核需要干净页的情况下，将进行显示刷出

### 概述
在页的刷出（*flushing*）、交换（*swapping*）、释放（*releasing*）操作之间，有着明确的关系。不仅需要定期检查内存页的状态，还需要检查空闲内存的大小。在完成检查后，未使用或很少使用的页将自动换出，但在换出前，其中包含的数据将与后备存储器同步，以防数据丢失。对动态产生的页，系统交换区充当后备存储器；对映射自文件的页来说，其交换区就是底层文件系统中与页对应的部分。如果内存发生严重的不足，必须强制刷出脏数据，以获得干净的页。

内存/缓存与后备存储器之间的同步，概念上分为两部分：
- 策略例程（*policy routine*）控制数据交换的时机。系统管理员可以设置各种参数，帮助内核判定何时交换数据，这实际上是系统负荷的一个函数。
- 技术实现处理缓存和后备存储器之间同步操作的硬件相关细节，并确保策略例程发出的指令得以执行。

同步和交换不能彼此混淆。同步只保证物理内存中保存的数据与后备存储器一致，而交换将导致从物理内存刷出数据，以释放空间，用于优先级更高的事项。在数据从物理内存清除之前，将与相关的后备存储器进行同步。

可能因不同原因、在不同的时机触发不同的刷出数据的机制：
- 周期性的内核线程，将扫描脏页的链表，并根据页变脏的时间，来选择一些页写回。如果系统不是太忙于写操作，那么在脏页的数目，以及刷出页所需的硬盘访问操作对系统造成的负荷之间，有一个可接受的比例。
- 如果系统中的脏页过多，内核将触发进一步的机制对脏页与后备存储器进行同步，直至脏页的数目降低到一个可接受的程度。而“脏页过多”和“可接受的程度”到底意味着什么，将在下文讨论。
- 内核的各个组件可能要求数据必须在特定事件发生时同步，例如在重新装载文件系统时。

前两种机制由内核线程 *pdflush* 实现，该线程执行同步代码，而第三种机制可能由内核中的多处代码触发。

- [ ] 图，数据同步涉及的一些函数的概述

内核可以从代码中任意位置发起数据同步，但所有的代码路径都经由 `sync_sb_inodes()`。该函数负责同步属于给定超级块的所有脏的 *inode*，对每个 *inode* 都使用 `writeback_single_inode()`。`sync()` 系统调用和各种通用的内核抽象层（如分区代码或块层）都利用了这种做法。

另一方面，系统也可能出现需要将所有超级块的脏 *inode* 进行同步的情况。对周期性回写和强制性回写，特别需要该操作。内核会在文件系统代码修改数据之前就启动同步操作，确保脏页的数目不失去控制。

对文件系统来说，同步一个超级块的所有脏 *inode* 通常粒度太粗了。它们通常需要同步一个脏的 *inode*，因而要直接使用 `writeback_single_inode()`。

即使同步实现围绕 *inode* 展开，但这并不意味着该机制只适用于已装载的文件系统所包含的数据。裸块设备由 *bdev* 伪文件系统的 *inode* 表示。因而，该同步方法也会影响到裸块设备，就像是普通的文件系统对象那样。

### pdflush 机制
*pdflush* 机制实现在一个文件中：`mm/pdflush.c`。

*pdflush* 是通常的内核线程机制启动的：
```c
// mm/pdflush.c
static void start_one_pdflush_thread(void) {
  kthread_run(pdflush, NULL, "pdflush");
}
```
- `start_one_pdflush_thread()` 启动一个 *pdflush* 线程，但内核通常会同时使用几个线程。
- 特定的 *pdflush* 线程并不总负责同一个块设备。
- 线程分配可能随时间变化，因为线程的数目不是常数，而且可能随系统负载而变化。实际上，内核在初始化 *pdflush* 子系统时，会启动 `MIN_PDFLUSH_THREADS` 个线程。通常，在普通负荷的系统上该数目是 2。`MAX_PDFLUSH_THREADS` 指定了 *pdflush* 实例的最大数目，通常为 8。并发 *pdflush* 线程数目的上下限分别定义为 `MIN_PDFLUSH_THREADS (2)` 和 `MAX_PDFLUSH_THREADS (8)`，内核总是遵守这两个限制。
- 并发线程的数目保存在 `nr_pdflush_threads` 全局变量中，但并不区分活动和睡眠的线程。用户空间可通过 `/proc/sys/vm/nr_pdflush_threads` 查看当前值。
- 通过 `ps` 可以看到进程列表中有活动的 *pdflush* 实例。
- 何时创建/销毁 *pdflush* 线程的策略是很简单的。如果 1 秒内都没有空闲线程可用，内核将创建一个新的线程。反之，如果某个线程的空闲时间已经超出 1 秒，则将被销毁。
- 现代系统通常具备多个块设备，如果系统中存在许多脏页，那么内核需要使这些设备尽可能忙于回写数据。不同块设备的队列是彼此独立的，因而数据可以并行写入。数据传输速率主要受限于 *I/O* 带宽，而不是当前硬件上 *CPU* 的计算能力。一个块设备可能有多个可以传输数据的队列，而一个 *pdflush* 线程可能服务于所有队列，也可能只向其中一个提供数据。
  - [ ] 图，pdflush 机制概述

### 启动新线程
pdflush 机制由两个主要部分构成，数据结构描述线程的工作，策略例程帮助执行工作。

数据结构定义：
```c
struct pdflush_work {
  struct task_struct *who;  // 指向线程 task_struct 实例
  void (*fn)(unsigned long) // 回调函数
  unsigned long arg0;       // 传递给回调函数的参数
  struct list_head list;    // 链表元素，用于在线程空闲时，将线程置于 pdflush_list 链表上
  unsigned long when_i_went_to_sleep;
};
```
- 该数据结构定义在 *C* 文件而非头文件中，向内核表明该结构只供内部代码使用。通用代码将使用其他机制来访问内核的同步功能。
- `when_i_went_to_sleep` 存储了线程上一次进入睡眠的时间。该值可用于从系统删除多余的 *pdflush* 线程（即在内存中已经有一段比较长的时间处于空闲状态的线程）

通过对 `fn` 使用不同的函数指针，内核能够将各种同步例程集成到 *pdflush* 框架中，以便为手头的工作选择正确的例程。

### 初始化线程
*pdflush* 用作内核线程的工作过程。在创建之后，*pdflush* 线程进入睡眠，直至内核的其他部分为线程指派任务，任务由 *pdflush_work* 描述。因而，*pdflush* 线程的数目无须与要执行的任务的数目匹配。创建的线程都处于待命状态，等待内核分配任务。

*pdflush* 的工作流程
```c
pdflush()
  __pdflush()
    // 添加到 pdflush_list 链表（操作对象为 pdflush_work 的实例）
    // 设置 when_i_went_to_sleep
    schedule()
    my_work->fn()
    // 创建/销毁线程
```
- 在工作函数结束时，内核将检查工作线程是否太多或太少。如果已经有 1 秒多的时间没有空闲工作线程，`start_one_pdflush_thread()` 创建一个新线程。如果睡眠时间最长的线程（在 `pdflush_list` 链表末尾）已经睡眠超过 1 秒，则退出无限循环，这使得当前线程被从系统删除。

### 执行实际工作
`pdflush_operation()` 为 *pdflush* 线程指定了一个工作函数，并唤醒该线程。如果没有可用的线程，则返回 -1；否则，从链表移除一个线程并唤醒。
```c
int pdflush_operation(void (*fn)(unsigned long), unsigned long arg0) {
  ...
}
```
- 如果 `pdflush_list` 链表是空的，没有可唤醒的 *pdflush* 守护进程，则返回一个错误码。
- 为确保总有足够的工作线程，内核在从 `pdflush_list` 链表移除当前实例之后、唤醒该线程之前，检查 `pdflush_list` 链表是否为空的。如果是空的，则 `last_empty_jifs` 将设置为当前系统时间。在线程结束时，内核将使用该信息来检查没有空闲线程可用的时间，如果超过 1 秒，将如上文所述那样创建一个新线程。

### 周期性刷出
pdflush 机制运作的框架，有两种方案可用，一种是周期性的，另一种是强制的。这里讨论周期性的回写机制。

在较早的内核版本中，使用一个用户态应用程序来执行周期性写操作。该应用程序在内核初始化时启动，每隔一定时间调用一个系统调用来回写脏页。与此同时，这个不那么优雅的过程被一个更为现代的方案替代，后者不通过用户态迂回，因而不仅更高效，而且更为优雅。

早期同步方法的名称是 *kupdate*，该名称会作为某些函数的一部分出现，通常用于描述刷出机制。

周期性地刷出脏的缓存数据需要两个组件：借助 *pdflush* 机制执行的工作函数，以及定期激活该机制的相关代码。

### 相关的数据结构
`mm/page-writeback.c` 中的 `wb_kupdate()` 函数负责刷出操作的技术实现。它基于地址空间概念，这一概念建立了物理内存与文件或 *inode* 和底层块设备之间的关联。

#### 页状态
`wb_kupdate()` 基于两个数据结构，二者控制了该函数的运作。

其中一个是全局数组 `vm_stat`，可用于查询所有系统内存页的状态：
```c
// mm/vmstat.c
atomic_long_t vm_stat[NR_VM_ZONE_STAT_ITEMS];
```
- 该数组保存了一组全面的统计信息，用于描述每个 *CPU* 的内存页面的状态。因而，系统中的每个 *CPU* 都对应该结构的一个实例。各个实例群集在一个数组中，以简化访问。
- 该结构的成员只是简单的基本类型的数字，表示具有特定状态的页的数目
- `NR_VM_ZONE_STAT_ITEMS`：
  ```c
  enum zone_stat_item {
    // 第一个缓存行，前 128 字节（假定字长为 64 位）
    NR_FREE_PAGES,
    NR_INACTIVE,
    NR_ACTIVE,
    NR_ANON_PAGES,    // 映射的匿名页
    NR_FILE_MAPPED,   // 页缓存中的页，映射到页表，只从进程上下文修改

    NR_FILE_PAGES,
    NR_FILE_DIRTY,
    NR_WRITEBACK,

    // 第二个 128 字节的缓存行
    NR_SLAB_RECLAIMABLE,
    NR_SLAB_UNRECLAIMABLE,
    NR_PAGETABLE,   // 用于页表
    NR_UNSTABLE_NFS,  // NFS 非稳定页
    NR_BOUNCE,
    NR_VMSCAN_WRITE,
  #ifdef CONFIG_NUMA
    // 忽略 NUMA 相关的统计量
  #endif
    NR_VM_ZONE_STAT_ITEMS
  };
  ```
  - `NR_PAGETABLE`: 用于存放页表的页的数目
  - `NR_SLAB_RECLAIMABLE` 和 `NR_SLAB_UNRECLAIMABLE` 表示用于 *slab* 缓存的页数目（这两个常数也用于 *slub* 缓存）
- 内核不仅维护了一个全局数组来收集页统计信息，还为各个内存域都提供了同样的信息：
  ```c
  struct zone {
    ...
    // 内存域统计量
    atomic_long_t vm_stat[NR_VM_ZONE_STAT_ITEMS];
  }
  ```

维护全局和特定于内存域的数组，使其状态能够反映最新的使用情况，是内存管理子系统的工作。

如何使用：为获得整个系统状态的概述，必须合并各数组项中的信息，才能获得整个系统的数据，而不是特定于 *CPU* 的数据。内核提供了辅助函数 `global_page_state()`，它可以提供 *vm_stat* 的一个特定字段的当前值：
```c
unsigned long global_page_state(enum zone_stat_item item)
```
- 因为各个 *vm_stat* 数组及其数组项并未由锁机制保护，在 `global_page_state()` 执行时，可能数据已经发生改变。所返回的结果不是准确值，而是近似值。这一点不成问题，因为该值只是工作分配的有效程度的一个一般性标志。在实际数据和返回值之间存在微小的差别，是可以接受的。

#### 回写控制
另一个数据结构保存了用于控制脏页回写的各种参数。上层使用该结构，将如何进行回写的相关信息传递给底层。但该结构也可以用来反向传播状态信息：
```c
// writeback.h
/* 一个控制结构，告知回写代码完成何种工作 */
struct writeback_control {
  struct backing_dev_info *bdi; // 如果不是 NULL，则只回写该队列
  enum writeback_sync_modes sync_mode;
  unsigned long *older_than_this; // 如果不是 NULL，则只回写变脏时间早于该值的 inode
  long nr_to_write;   // 回写该属性指定数目的页，每回写一页，将该属性值减 1
  long pages_skipped;   // 未回写的页的数目

  loff_t range_start;
  loff_t range_end;

  unsigned nonblocking:1; // 不要在请求队列上阻塞
  unsigned encountered_congestion:1;  // 一个状态输出，表示队列满
  unsigned for_kupdate:1; // kupdate 回写
  unsigned for_reclaim:1; // 从页分配器调用
  unsigned for_writepages:1;  // 这是一个 writepages() 调用
  unsigned range_cyclic:1;  // range_start 是循环的
}
```
...

#### 可调参数
内核支持通过参数对同步操作进行微调。这些参数可以由管理员设置，以帮助内核评估系统的使用状况和负荷。*sysctl* 机制即用于此目的，这意味着 *proc* 文件系统成为了操作这些参数的固有的接口，这些参数位于 `/proc/sys/vm`。有如下 4 个参数可以设置，都定义在 `mm/pagewriteback.c`：
1. `dirty_background_ratio`：指定脏页的百分比，当脏页比例超出该阈值时，*pdflush* 在后台开始周期性的刷出操作。默认值为 10，当与后备存储器相比，有超过 10% 的页变脏时，*pdflush* 机制将开始运转。
2. `vm_dirty_ratio`（对应的 *sysctl* 是 `dirty_ratio`）：指定了脏页（相对于非高端内存域）的百分比，脏页比例超出该阈值时，将开始刷出，默认值为 40。
3. `dirty_writeback_interval`（对应的 *sysctl* 时 `dirty_writeback_centisecs`）：定义了周期性刷出例程两次调用之间的间隔，单位是百分之一秒。默认值是 500，相当于两次调用的间隔为 5 秒。在进行大量写操作的系统上，降低该值将提高性能；但在写操作数量很少的系统上，增加该值只能带来很少的性能增益。
4. `dirty_expire_interval`（对应的 *sysctl* 是 `dirty_expire_centisecs`）：一页可以保持为脏状态的最长时间，单位是半分之一秒。默认值为 3000。

### 中央控制
周期性刷出操作中，关键性的一个组件是定义在 `mm/page-writeback.c` 中的 `wb_kupdate()` 过程，它负责指派底层例程在内存中查找脏页，并将其与底层块设备同步。

`wb_kupdate()` 的代码流程：
```c
wb_kupdate()
  sync_supers()
  global_page_state()
  writeback_inodes()
  // 拥塞？-> congestion_wait()
  // 重置回写定时器
```
...

### 超级块同步
超级块数据通过一个专用函数 `sync_supers()` 进行同步，这使得它与普通的同步操作区分开来。该函数及其他与超级块相关的函数都定义在 `fs/super.c`。

代码流程：
```c
sync_supers()
  // 超级块是脏的？-> write_super() -> write_sb->s_op->write_super()
```
- 内核提供了全局链表 `super_blocks` 来保存所有装载文件系统的 *super_block* 实例。`sync_supers()` 最初的任务就是遍历所有超级块，并根据超级块结构 `s_dirt` 成员来检查超级快是否是脏的。如果是，则通过 `write_super()` 将超级块数据的内容写到数据媒体。
- 实际的写入操作由特定于超级块的 *super_operations* 结构所包含的 `write_super()` 方法完成。如果该函数指针未设置，则该文件系统不需要超级块同步（例如虚拟的和基于物理内存的文件系统）。

### inode 同步
`writeback_inodes()` 通过遍历系统的 *inode* 对安装的映射进行回写（这称为 *inode* 回写，但事实上不只是 *inode*，连同相关的脏数据都进行了回写）。该函数肩负着主要的同步工作，因为大多数系统数据都是以地址空间映射的形式提供的，均利用了 *inode*。

- [ ] 图，writeback_inodes() 的代码流程图。

该函数利用了第八章讨论的数据结构，来建立超级块、*inode* 和相关数据之间的关联。

#### 遍历超级块
在逐 *inode* 回写各映射时，最初的路径是通过系统中表示已装载文件系统的所有超级块实例，对每个超级块实例都会调用 `sync_sb_inodes()` 以回写超级块 *inode* 数据。在以下两种情况下，对超级块链表的遍历会结束：
1. 已经顺序扫描了所有超级块实例。内核已经达到链表末尾，因而已经完成工作。
2. *writeback_control* 实例中指定的回写页的最大数目已经达到。由于回写需要获得各种重要的锁，因此不应该让回写操作干扰系统的正常运作太长时间，以便内核的其他部分能够恢复对相关 *inode* 的访问。

#### 考察超级块 inode
在借助于超级块结构确认文件系统确实包含带有脏数据的 *inode* 之后，内核将工作转交给 `sync_sb_inodes()`，该函数将同步脏的超级块 *inode*。

为便于检索脏的 *inode*，内核将所有脏 *indoe* 置于特定于超级块的链表 `super_block->s_dirty` 上。该链表中的 *indoe* 是按照时间逆序排列的，*inode* 变脏的时间越靠后，它就越接近链表的尾部。

为对这些 *inode* 进行同步，还需要两个链表头，*super_block* 结构的相关部分如下：
```c
struct super_block {
  ...
  struct list_head s_dirty; // 脏 inode 的链表
  struct list_head s_io;  // 等待回写
  struct list_head s_more_io; // 等待回写，另一个链表
  ...
};
```
- `s_io` 链表保存了同步代码当前考虑回写的所有 *inode*
- `s_more_io` 包含那些已经被选中进行同步的 *inode*，也置于 `s_io` 链表中，但不能一次处理完毕。看起来最简单的解决方案是由内核将这些 *inode* 放回到 `s_io` 链表，但这可能导致新近变脏的 *inode* 无法得到同步处理，或导致锁方面的问题，因此又引入了一个链表。

`sync_sb_inodes()` 的第一个任务是填充 `s_io` 链表。必须区分两种情况：...

在选择了 `s_io` 链表的成员之后，内核开始遍历各链表元素。

在实际回写之前，会进行一些检查，以确认相关的 *inode* 适合进行同步：... 

#### 回写单个 inode
内核将同步与一个 inode 相关联数据的任务委托给 `__writeback_single_inode()`。

- [ ] 图，代码流程

该函数本质上是 `__sync_single_inode()` 的分配器，但负有重任，需要区分是进行数据完整性回写，还是普通回写。这会影响对被锁定 *inode* 的处理方式。

...

### 拥塞
术语拥塞（*congestion*）表示在某个内核块设备队列负荷了过多读/写操作时，向队列增加更多与块设备通信的请求是没有意义的。在提交新的读/写请求之前，最好等待一段时间，使得一定数目的请求被处理完成，而队列能变得短一些。

接下来，将讲述内核在技术层次上对该定义的实现。

#### 数据结构
实现拥塞方法需要两个等待队列：
```c
static waitqueue_head_t congestion_wqh[2] = {
  __WAIT_QUEUE_HEAD_INITIALIZER(congestion_wqh[0]),
  __WAIT_QUEUE_HEAD_INITIALIZER(congestion_wqh[1])
};
```
- 两个队列，一个用于输入，另一个用于输出
- `<fs.h>` 中定义了两个预处理器常数（`READ` 和 `WRITE`）来访问数组元素
- 注意，上述两个队列不能直接用标准的等待队列方法来操作。相反，内核为此提供了若干辅助函数，声明在 `<backing-dev.h>` 中。

#### 阈值
内核根据对特定的请求队列判断请求的数目是否超出了最小值和最大值（或阈值）来区分一个请求队列是否拥塞。

内核对此并不使用固定的常数。相反，它根据系统主存储器来定义这些限制值，因为拥塞请求的数目是随着主存储器大小而变化的。

每个块设备都有一个请求队列，由 `struct request_queue` 定义，我们感兴趣的一些字段如下：
```c
struct request_queue {
  ...
  unsigned long nr_requests;  // 请求的最大数目
  unsigned int nr_congestion_on;
  unsigned int nr_congestion_off;
  unsigned int nr_batching;
  ...
};
```
- `nr_requests` 定义每个队列中 *request* 结构的最大数目。通常，该数目设置为 `BLKDEV_MAX_RQ`，其值为 128，但可以使用 `/sys/block/<device>/queue/nr_requests` 修改。请求数目的下界由 `BLKDEV_MIN_RQ` 给出，其值为 4。
- `nr_congestion_on` 表示队列请求数目达到拥塞的阈值。发生拥塞时，空闲的 *request* 结构的数目必定小于该值
- `nr_congestion_off` 也指定了一个阈值，该阈值表示队列不再被认为拥塞。当空闲 *request* 结构的数目多于该值时，内核认为该队列不是拥塞的。

`queue_congestion_on_threshold()` 和 `queue_congestion_off_threshold()` 函数用于读取当前的阈值。

拥塞阈值由 `blk_queue_congestion_threshold()` 计算：
```c
static void blk_queue_congestion_threshold(struct request_queue *q);
```

#### 拥塞状态的设置和清除
内核提供了两个标准函数（声明在 `<blkdev.h>` 中）来设置和清除队列的拥塞状态：
- `blk_set_queue_congested()`
- `blk_clear_queue_congested()`

#### 在拥塞队列上等待
内容使用 `congestion_wait()` 在拥塞发生时将一个进程添加到 `congestion_wqh` 等待队列。

该函数需要两个参数，数据流的方向（读或写）以及一个超时时间。在经过超时时间指定的时间间隔之后，总是会唤醒进程，即使队列仍然是拥塞的。这个超时设置用于防止出现长时间的停滞。毕竟，队列可能拥塞比较长的时间：
```c
long congestion_wait(int rw, long timeount);
```

### 强制回写
在系统负荷不高时，上述以后台活动来回写内存页的机制工作得很好。内核能够确保脏页的数目一定不会失去控制，并且在物理内存和底层块设备之间，数据可以充分交换。但是，如果某些进程的缓存数据快速变脏，致使需要比普通方法更多的同步操作时，情况就会发生改变。

在内核接收到对内存的紧急请求，而同时因为有大量脏页而不能满足请求时，内核必须设法尽快将脏页的内容传输到块设备，以尽快释放物理内存以用于其他目的。在这种情况下，使用的方法与后台刷出数据所用的方法是相同的，但同步操作不是由周期性过程发起的，而是由内核显式触发的。换句话说，这种回写是“强制”的。

这种立即式同步请求不仅可能由内核发起，也可能来自用户空间，如 *sync* 命令（和对应的 `sync()` 系统调用）通知内核将所有脏数据输出到块设备。内核为此还提供了其他系统调用。

同步是基于 `wakeup_pdflush()` 的，该函数实现在 `mm/page-writeback.c` 中，刷出页的数目作为参数传递给函数：
```c
int wakeup_pdflush(long nr_pages) { 
  ... 
}
```
- 如果传递了参数 0，即没有明确指定回写页的数目，内核将调用 `global_page_state()` 来确定整个系统范围内（基于文件的）脏页数目。
- 接下来激活一个 *pdflush* 线程，但这一次使用的是 `background_writeout()` 函数，而不是 `wb_kupdate()`。`background_writeout()` 并不显式等待页回写到后备存储器的完成，而只是发起一个对应的请求。与此相反，在进行数据完整性同步时（在请求发源于一个系统调用时，通常就是这样），内核必须一直等到发出的写请求完成为止。
- `background_writepages()` 和 `wb_kupdate()` 之间，只在细节上有以下的微小差别：
  - ...
- `wakeup_pdflush()` 仅在内核源码中两个地方以非 0 参数调用：
  - `free_more_memory()`: 在内存不足以生成页缓存时，总会使用该函数。在这种情况下，使用的参数是固定值 1024。
  - `try_to_free_pages()`: 页面回收，采用了 `wakeup_pdflush()` 方法，来回写扫描缓存时认为多余的页中的脏数据。（在使用膝上模式时，`try_to_free_pages()` 也会以 0 参数调用 `wakeup_pdflush()`。）
- 所有其他调用都回写所有脏页，即对页数没有最大限制。
- 可以理解，回写所有脏页时代价很高、很耗时的操作，因而其使用应该非常谨慎，在内核中只用于下述少量情况：
  - 在 `sync()` 系统调用明确请求同步脏数据时
  - 在紧急同步时，或使用 *magic system request key* 请求紧急重新装载时
  - `balance_dirty_pages()` 通知 `background_writeout()` 尽可能多回写一些页。在文件系统（或内核其他部分）在一个映射上产生脏页时，*VFS* 层将调用该函数。如果系统中脏页的数目过大，那么将使用 `background_writeout()` 开始同步。

### 膝上模式
笔记本计算机的用户倾向于尽可能降低机器的耗电量。在有些情况下，*pdflush* 是可以发挥作用的。硬盘的运作必然需要旋转（非固态元件），这会消耗电力，在不需要使用硬盘时，降低旋转速度有助于减少耗电量。

与匀速运转的磁盘相比，加速旋转的硬盘更糟糕，因为这需要更多能量。因而，优化内核有如下两方面：
- 使硬盘尽可能低俗旋转，这可以通过将写操作延迟更长的时间来做到
- 在磁盘旋转加速不可避免的情况下，执行所有待决的写操作，即使在普通情况下，这些操作仍然要继续延迟。这有助于防止硬盘来回升高/降低旋转速度。

本质上，磁盘操作是猝发执行的：如果必须从设备读取数据，那么所有待决写操作都可以执行，因为无论如何设备现在已经激活了。

为了实现这些目标，内核提供了一种膝上模式，可以通过 `/proc/sys/vm/laptop_mode` 激活。全局变量 `laptop_mode` 充当一个逻辑标志，表示当前膝上模式是否激活。例如，用户层守护进程可以根据供电是否来自于电池，使用该文件来启用或禁用膝上模式。内核文档：`Documentation/laptop-mode.txt`。

膝上模式对同步代码的改变：
- ...

### 用于同步控制的系统调用
可以从用户空间通过各种系统调用来启用内核同步机制，以确保内存和块设备之间（完全或部分）的数据完整性：
1. 使用 `sync()` 系统调用刷出整个缓存内容。在某些情况下，这可能非常耗时
2. 各个文件的内容（以及相关 *inode* 的元数据）可以被传输到底层的块设备。内核为此提供了 `fsync()` 和 `fdatasync()` 系统调用。它们专用于特定的应用程序，因为刷出的文件是通过特定于进程的文件描述符来选择的。因而，没有一个通用的用户空间工具可以回写特定的文件。
3. `msync()` 用于同步内存映射。

### 完全同步
`sync()` 系统调用在 `sys_sync()` 中实现，位于 `fs/buffer.c`。
- [ ] 图，sys_sync 的代码流程图

- `wakeup_pdflush()` 的调用参数为 0，这将导致回写系统中所有的脏页。
- 通过 `sync_inodes()` 同步 *inode* 的元数据
- `sync_supers()` 遍历 `super_blocks` 链表中的所有超级块，如果 `super_block->write_super()` 例程存在，则调用之。这会导致将特定于超级块的信息写回到对应的各文件系统。
- `sync_filesystems()` 通过再次遍历 `super_blocks` 链表并对每个以读/写模式装载并提供了 `sync_fs()` 方法的文件系统调用 `sync_fs()` 例程，来同步装载的各个文件系统。仅在通过系统调用请求显式同步时，才会调用该方法。

`sync_inodes()` 和 `sync_filesystems()` 会调用两次，首先用参数 0（异步），然后用参数 1（同步）。
...

#### inode 的同步
`sync_inodes()` 会同步所有脏 *inode*。
- [ ] sync_inodes 的代码流程图

#### 单个文件的同步
该选项由应用程序使用，以确保它们在内存中修改的数据总是写回到适当的块设备。因为普通的写访问操作总是先进入缓存，该选项为真正重要的数据提供了额外的安全性（当然，另一种选择是使用直接 *I/O* 操作，绕过缓存）。

有如下几个系统调用可用于此：
1. `fsync()` 同步一个文件的内容，并将与文件的 *inode* 相关的元数据写回到块设备
2. `fdatasync()` 仅写回数据内容，忽略元数据
3. `sync_file_range()` 是一个相对较新的系统调用。它可以对打开文件中精确定义的部分进行受控同步。本质上，其实现将选择目标内存页进行回写，可能会等待结果。

`fsync()` 和 `fdatasync()` 的实现只有一处不同：
```c
asmlinage long sys_fsync(unsigned int fd) {
  return __do_fsync(fd, 0);
}

asmlinage long sys_fdatasync(unsigned int fd) {
  return __do_fsync(fd, 1);
}
```
...

#### 内存映射的同步
内核提供了在 `sys_msync()` 中实现的 `msync()` 系统调用，以便同步内存映射的部分或全部数据：
```c
asmlinage long sys_msync(unsigned long start, size_t len, int flags);
```
- `start` 和 `len` 选择了进程的用户地址空间中的一个区域，其中映射的数据将与底层的文件同步。
- 参考：`man 2 msync`
- 该系统调用本质上分为两种模式：
  - 如果标志中设置了 `MS_SYNC`，那么脏页将同步写出到磁盘
  - `MS_ASYNC` 标志则将脏数据的回写调度到稍后执行

### 小结
数据在块设备上持久存储，但在物理内存中进行修改。这使得有必要定期同步二者的内容，本章介绍了相应的方法。

有几个系统调用可以显式请求将内存的一部分回写到磁盘。此外，内核使用线程来周期性地执行同一工作，以确保不会有太多修改过的数据存在于物理内存中。

尽管保持磁盘忙碌是件好事情，但内核需要确保回写的数据量不超出磁盘处理能力的限制，本章针对这一问题讨论了用于避免块层拥塞的技术。另外，在缺乏电力的系统上出现的问题可能需要一些修正，本章讲述了膝上模式对相关策略的修改。

到现在为止，本书已经介绍了数据在块设备和物理内存之间双向传输的所有技术细节。尚未讨论的是，在内核缺少内存时，如何决定从物理内存选择哪些页进行同步或丢弃，这是第十八章的主题。

## 第十八章-页面回收和页交换
页面回收（*page reclaim*）包含的内容：
1. 页交换（*swapping*）或换页（*paging*）：内核将很少使用的部分内存换出到块设备，这相当于提供了更多的主内存；
2. 如果一个很少使用的页的后备存储器是一个块设备（例如，文件的内存映射），那么就无需换出被修改的页，而是可以直接与块设备同步。腾出的页帧可以重用，如果再次需要该数据，可以从来源重新建立该页。
3. 如果页的后备存储器是一个文件，但不能在内存中修改（例如，二进制可执行文件的数据），那么在当前不需要的情况下，可以直接丢弃该页。
4. 选择很少使用的页的策略

注意，分配给核心内核（即并非用于缓存）的页是不能回收的，因为这种做法带来的复杂性的增加，将超出其好处。

页面回收是内核与缓存相关的一项基本决策的基石。缓存的长度从来都不是固定的，可以根据需要增长。其背后的原理很简单：没有使用的物理内存，与其浪费，还不如用来缓存一些数据。但如果一些重要的任务需要被缓存占用的内存，内核将回收内存以支持这些需求。

### 概述
（数据同步）数据与底层块设备的同步，将缓存的数据回写，可以释放一些内存页。但该过程也有其局限性：在某些时候，会遇到这样的情况，缓存和缓冲区都不能再收缩。另外，数据同步对动态产生的内存页是不适用的，因为这种页没有后备存储器。

（页交换）因为在通常的系统中硬盘容量比物理内存容量大很多，内核连同处理器可以征用部分磁盘，用作内存的扩展。由于硬盘比物理内存慢很多，页交换只是一种紧急情况下的备用方案，它使得系统可以运行，但速度会降低很多。交换（*swapping*）最初是指换出整个进程，包括其所有的数据、代码等，而不像现在这样，将进程数据选择性地逐页换出到二级存储。这里不区分交换和换页，二者都表示细粒度换出进程的数据。

在内核中考虑如何实现页交换和页面回收，必须回答下面两个问题：
1. 根据何种方案来回收页，即内核为确保最大可能利益以及最小可能损失，如何判断应该回收哪些页？
2. 换出的页在交换区中如何组织，内核如何将页写入到交换区，如何在此后需要时再次读取？内核如何将页与后备存储设备进行同步？

换出哪些页、在物理内存中保留哪些页的决策，对系统性能有决定性的影响。（不能选择一个频繁使用的页换出）

### 可换出页
只有少量几种页可以换出到交换区（对其他页来说，换出到块设备上与之对应的后备存储器即可）：
1. 类型为 `MAP_ANONYMOUS` 的页，没有关联到文件（或属于 `/dev/zero` 的一个映射）。例如，这可能是进程的栈，或是使用 `mmap()` 匿名映射的内存区。
2. 进程的私有映射用于映射修改后不向底层块设备回写的文件，通常换出到交换区。因为这种情况下文件不能再用作后备存储器，因而物理内存较少时，需要将相关页换出到交换区，因为此时不能从文件恢复页的内容。内核（以及 *C* 标准库）使用 `MAP_PRIVATE` 标志来创建此类映射。
3. 所有属于进程堆以及使用 `malloc()` 分配的页（*malloc* 又使用了 `brk()` 系统调用或匿名映射）
4. 用于实现某种进程间通信机制的页。例如，用于在进程之间交换数据的共享内存页。

不可换出：
1. 内核本身使用的内存页：由内核本身使用的内存页决不会换出。因为这将显著增加内核代码的复杂性。由于与其他用户应用程序相比，内核不需要非常多的内存，而且与付出的额外工作量相比，将内核内存页换出的潜在收益太低。
2. 用于将外设映射到内存空间的页也不能换出：换出这些页是没有意义的，特别是，这些页只用作应用程序和设备之间通信的手段，而并非用于持久存储数据。

#### 页颠簸
在进行页交换时，可能发生的另一个问题是页颠簸（*page thrashing*）。这个问题涉及交换区和物理内存之间密集的数据传输问题归结为页的来回、反复的交换。在系统进程的数目增加时，这种现象发生的几率也会增加。在换出重要的数据后不久再次需要该数据时，会发生这种现象。

为防止页颠簸，内核必须解决的主要问题是：尽可能精确地确定一个进程的工作集（*working set*，即使用最频繁的那些内存页），将最不重要的那些页移动到交换区或其他后备存储器，而真正重要的页则一直驻留在内存中。

为此，内核需要一种合适的算法，来评估整个系统中的页的重要性。

许多类型的 *CPU* 提供了不同的方法。*Linux* 不可能使用所有的方法，必须找到一个最小公分母，使得内核能够在此基础上建立硬件无关层。这里有一个特别简单但很重要的技巧，它完全独立于处理器的能力，即在系统中维护一个交换令牌（*swap token*），赋予换入页的进程。内核会试图避免从该进程换出页，以减轻该进程的颠簸，使之有时间完成任务。不久之后，交换令牌将传递到另一个进程，该进程也经历了页交换，而且比当前令牌持有者更需要内存。

#### 页交换算法
描述 *Linux* 页交换实现所基于的两种技术。

1. 第二次机会
2. LRQ 算法

### Linux 内核中的页面回收和页交换
#### 交换区的组织
换出的页或者保存在一个没有文件系统的专用分区中，或者存储在某个现存文件系统中的一个定长文件中。

可以同时使用几个这样的区域，还可以根据各个交换区的速度不同为其指定优先级。内核使用交换区时可以根据优先级进行选择。如果使用了几个优先级相同的交换区，内核将使用一种循环进程来确保尽可能均匀地利用各个交换区。如果交换区的优先级不同，内核首先使用高优先级的交换区，然后逐渐转移到优先级较低的交换区。

每个交换区都细分为若干连续的槽（*slot*），每个槽的长度刚好与系统的一个页帧相同。本质上，系统中的任何一页都可以容纳到交换区的任一槽中。但内核还使用了一种称为聚集（*clustering*）的构造法，使得能够尽快访问交换区。进程内存区中连续的页（或至少是连续换出的页）将按照特定的聚集大小（通常是 256 页）逐一写到硬盘上。如果交换区中没有更多空间可容纳此长度的聚集，内核可以使用其他任何位置上的空闲槽位。

为跟踪内存页在交换区中的位置，内核必须维护一些数据结构，将该信息保存在内存中。结构中，最重要的数据成员是一个位图，用于跟踪交换区中各槽位的使用/空闲状态。其他成员包含的数据结构用于支持选择接下来使用的槽位，以及聚集的实现。

有两个用户空间工具可用于创建和启用交换区，分别是 *mkswap*（用于“格式化”一个交换分区/文件）和 *swapon*（用于启用一个交换区）。

#### 检查内存使用情况
在换出内存页之前，内核会检查内存的使用情况，确定可用内存容量是否较低。与同步页的情况相似，内核联合使用了如下两种机制：
1. 一个周期性的守护进程（*kswapd*）在后台运行，该进程不断检查当前的内存使用情况，以便在达到特定的阈值时发起页的换出操作。使用该方法，确保了不会出现突然需要换出大量页的情况，这种情况将导致系统出现长时间的等待，必须不惜一切代价防止。
2. 但内核在某些情况下，必须能够预期可能突然出现的严重内存不足，例如在通过伙伴系统分配一大块内存时，或创建缓冲区时。如果没有足够的物理内存可用来满足对内存的请求，内核必须尽快换出页，以期释放一些内存空间。在紧急情况下的换出操作，属于直接回收（*direct reclaim*）的一部分。

如果内核无法满足对内存的需求，甚至在换出页之后也是如此，那么虚拟内存子系统只有一个选择，即通过 *OOM*（*out of memory*，内存不足）*killer* 来结束一个进程。虽然 *OOM killer* 有时候可能导致严重损失，但总比系统完全崩溃要好。如果在内存不足的情况下不采取措施，很可能导致系统崩溃。

#### 选择要换出的页
内核实现了一种粗粒度的 *LRU* 方法，只使用了一种硬件特性，即在访问一页之后设置一个访问位，该功能在内核支持的所有体系结构上都可用。

与通用的算法相比，内核对 *LRU* 的实现基于两个链表，分别称为活动链表和惰性链表（系统中的每个内存域都有这样的两个链表）。所有处于活动使用状态的页在一个链表上，而所有惰性页则保存在另一个链表上，这些页虽然可能映射到一个或多个进程，但不经常使用。为在两个链表之间分配页，内核需要定期执行均衡操作，通过上述访问位来确定一页是活动的还是惰性的，换言之，即该页是否经常被系统中的应用程序访问。页在两个链表之间可能会发生双向移动，但这种转移不是每次访问都会发生，它发生的事件间隔会比较长。

随着事件的推移，最不常用的页将收集到惰性链表的末尾。在出现内存不足时，内核将选择换出这些页。因为这些页到换出时，一直都很少使用，所以根据 *LRU* 原理，换出这些页对系统的破坏是最小的。

#### 处理缺页异常
*Linux* 运行的所有体系结构都支持缺页异常的概念，当访问虚拟地址空间中的一页，但该页不在物理内存中的时候，将触发缺页异常。缺页异常通知内核从交换区和其他的后备存储器读取缺失的数据。当然，可能需要先删除其他的页，以便为新数据腾出空间。

缺页处理分为两个部分。首先，必须使用与处理器相关性较强的代码（汇编语言）来截获该缺页异常，并查询相关的数据。其他，使用系统无关代码进一步处理该异常。由于内核在管理进程时所采用的优化，因而只在后备存储器中查找相关页并将其加载到物理内存是不够的，因为缺页异常可能是由其他原因触发的。例如，这可能涉及写时复制页，这些页仅在进程 *fork* 后执行第一次写访问时，才会进行复制。在按需换页时也会发生缺页异常，按需换页法是指映射的页仅在实际需要时才加载。

这种情况下，需要完成的工作同样不止是从交换区中查找到目标页。因为如果需要将磁头移动到一个新位置（磁盘寻道），对硬盘的访问可能比普通情况下更慢，所以内核使用了一种预读机制来预测接下来将需要哪些页，读操作包括对这些页的读取。

#### 缩减内核缓存
换出属于用户空间应用程序的页，并不是内核释放内存空间的唯一方法。缩减大量缓存，通常会有很好的效果。

因为内核缓存通常不是特别大，所以仅在万不得已时，内核才考虑缩减缓存。

内核在很多领域提供了各种缓存，这使得很难定义一个一般性的方案并据此缩减缓存，因为很难评估各种缓存所包含的数据的重要性。为此，早期的内核带有大量特定于缓存的函数，以便为各个缓存评估数据的重要性。

现在，用于缩减各种缓存的方法仍然是分别实现的，因为各种缓存的结构有很大的不同，很难采用一种通用的缓存收缩算法。但现在内核提供了一种通常框架，来管理各种缓存收缩方法。用于缩减缓存的函数在内核中称为收缩器（*shrinker*），可以动态注册。在缺乏内存时，内核将调用所有注册的收缩器来获得内存。

### 管理交换区
*Linux* 对交换区的支持，是相对比较灵活的：
- 可以用不同的优先级来管理几个交换区；
- 交换区可以是本地分区，也可以是具有预定义长度的文件
- 在活动的系统上，可以动态添加/删除交换分区，而无须重启

内核尽可能使各种方法在技术上的差别对用户空间透明。内核的模块结构也意味着，与页交换相关的算法可以采用一种通用设计，不同方法的差别只存在于较低的技术层次上。

#### 数据结构
交换区管理的基石是 `mm/swap-info.c` 中定义的 `swap_info` 数组，其中各数组项存储了关于系统中各个交换区的信息：
```c
struct swap_info_struct swap_info[MAX_SWAPFILES];
```
- 数组项的数目是在编译时由 `MAX_SWAPFILES` 静态定义的，该常数通常定义为 `2^5 = 32`
- 内核使用交换文件（*swap file*）这个术语时，不仅是指用于页交换的文件，还包括交换分区，因此上述数组包括了这两种类型
- 因为通常只使用一个交换文件，将数组长度限制为某个特定的数值，不会有什么影响

1. 交换区的特征
   - `struct swap_info_struct` 描述了一个交换区：
    ```c
    struct swap_info_struct {
      unsigned int flags;
      int prio;   // 交换区的优先级
      struct file *swap_file;
      struct block_device *bdev;
      struct list_head extent_list;
      struct swap_extent *curr_swap_extent;
      unsigned short *swap_map;
      unsigned int lowest_bit;
      unsigned int highest_bit;
      unsigned int cluster_next;
      unsigned int cluster_nr;
      unsigned int pages;
      unsigned int max;
      unsigned int inuse_pages;
      int next; // 交换区列表中的下一项
    };
    ```
    - `flags`：交换区状态的各个标志。`SWP_USED` 表明当前项在交换数组中处于使用状态，否则相应的数组项会用字节 0 填充，使得很容易区分使用和未使用的数组项；`SWP_WRITEOK` 指定当前项对应的交换区可写。在交换区插入到内核之后，这两个标志都会设置：二者合并后的缩写是 `SWP_ACTIVE`。
    - `swap_file`：指向与该交换区关联的 *file* 结构。对于交换分区，这是一个指向块设备上分区的设备文件的指针（例如，`/dev/hda5`）；对于交换文件，该指针指向相关文件的 *file* 实例（例如，`/mnt/swap1`）
    - `bdev`：指向文件/分区所在底层块设备的 *block_device* 结构（内核像独立块设备一样来管理分区，这导致即使交换区都在同一个磁盘，但它们 `bdev` 指针却指向不同的实例（不同分区实例不同））
    - `prio`：交换区的相对优先级；有符号数据类型，所以正负优先级都是可能的
    - `pages`：保存了交换区中可用槽位的总数，每个槽位可容纳一个内存域
    - `max`：保存了交换区当前包含的页数。不同于 `pages`，该成员不仅计算可用的槽位，而且包括那些（例如，因为块设备故障）损坏或用于管理目的的槽位。因为在当今的硬盘上，坏块是极端罕见的，`max` 通常等于 `pages` 加 1。二者差一个槽位有两个原因：首先，交换区的第一个槽位由内核用作标识；其次，内核还使用第一个槽位来存储状态信息，例如交换区的长度和坏扇区的列表，该信息必须持久保留。
    - `swap_map`：指向一个短整型数组（该数组在下文称为交换映射），其中包含的项数与交换区槽位数目相同。该数组用作一个访问计数器，每个数组项都表示共享对应换出页的进程的数目；
    - `next`：内核将交换数组中的各个数组项按优先级连接起来。因为表示各个交换区的数据位于一个线性数组的各数组项中，所以在固定的数组项位置之外，使用了 `next` 成员来建立一个相对的顺序。`next` 实际上是下一个交换区在 `swap_info[]` 数组中的索引。内核使用全局变量 `swap_list`，类型为 `swap_list_t`，用于专门查找第一个交换区：
      ```c
      struct swap_list_t {
        int head; // 按优先级排序的交换文件列表的第一项
        int next; // 交换文件的下一项
      };
      ```
    - `lowest_bit` 和 `highest_bit`：为了减少扫描整个交换区查找空闲槽位的搜索空间，内核借助这两个成员，来管理搜索区域的下界和上界。在 `lowest_bit` 之下和 `highest_bit` 之上，是没有空闲槽位的。
    - `cluster_next` 和 `cluster_nr`：实现上文提到的聚集技术。前者指定了在交换区中接下来使用的槽位的索引，而后者则表示当前聚集中仍然可用的槽位数，在消耗了这些空闲槽位之后，则必须建立一个新的聚集。
   - 交换区状态中一些主要的数据可以借助 *proc* 文件系统快速查询：`cat /proc/swaps`
2. 用于实现非连续交换区的区间
  内核使用 `extent_list` 和 `curr_swap_extent` 成员来实现区间（*extent*），用于创建假定连续的交换区槽位与交换文件的磁盘块之间的映射。

  如果使用分区作为交换区间，这是不必要的，因为内核可以依赖于一个事实，即分区中的块在磁盘上是线性排列的。因而槽位与磁盘块之间的映射会非常简单，从第一个磁盘块开始，加上所需的页数乘以一个常量得到的偏移量，即可获得所需地址。这种情况下，只需要一个 *swap_extent* 实例。
  - [ ] 图，用于管理非连续交换区的区间

  在使用文件作为交换区时，情况会更复杂，因为无法保证文件的所有块在磁盘上连续的。因而，在槽位与磁盘块之间的映射更为复杂。文件由多个部分组成，这些部分可能位于块设备的任意位置。（磁盘碎片的程度越轻，文件分成的部分就越少。毕竟，如果文件的各部分数据尽可能接近，才是最好的。）`extent_list` 链表的任务是，将文件散布在块设备上各处的块，与线性排布的槽位关联起来。没有必要将每个槽位都关联到块号，将一个连续块组的第一个块与对应槽位关联并标明块组中的块数，就可以非常紧凑地将文件的结构刻画出来。

  区间结构 `struct swap_extent` 定义如下：
  ```c
  struct swap_extent {
    struct list_head list;
    pgoff_t start_page;
    pgoff_t nr_pages;
    sector_t start_block;
  };
  ```

#### 创建交换区
新交换分区不是由内核直接创建的，这项任务委托给一个用户空间工具（*mkswap*），其源代码位于 *until-linux-ng* 工具集合中。

在使用交换区之间，创建交换区是一个强制性的步骤。

内核无须提供任何新系统调用来支持创建交换区，毕竟，内核也没有提供任何系统调用来创建普通的文件系统，这些显然都不是内核的问题。用于直接与块设备（或者，交换文件而言是块设备上的一个文件）通信的现存系统调用，已经足以按照内核的需求来组织交换区的内容。

*mkswap* 只需要一个参数，即分区的设备文件的名称，或交换区所在文件的名称。（可以指定其他参数，如交换区的长度、页长度。但在大多数情况下，这是无意义的，因为这些数据可以自动而可靠地计算出来。*mkswap* 的作者不建议用户自行指定这些参数。）

#### 激活交换区
为通知内核，已经用 *mkswap* 初始化了一个交换区，用于扩展物理内存，这需要与用户空间的交换。内核为此提供了 `swapon()` 系统调用。它在 `sys_swapon, mm/swapfile.c` 中实现。

1. 读取交换区特征信息
  交换区的特征信息保存在第一个槽位中，内核使用下列结构来解释该数据：
  ```c
  union swap_header {
    struct
    {
      char reserved[PAGE_SIZE - 10];
      char magic[10];   // SWAP-SPACE 或 SWAPSPACE2
    } magic;
    struct
    {
      char bottbits[1024];  // 用于存储磁盘标签等
      __u32 version;
      __u32 last_page;
      __u32 nr_badpages;
      unsigned char sws_uuid[16];
      unsigned char sws_volume[16];
      __u32 padding[17];
      __u32 badpages[1];
    } info;
  };
  ```
2. 创建区间链表
  `setup_swap_extents()` 用于创建区间链表。

### 交换缓存
前面已经根据页交换子系统的数据结构描述了其布局，以下几节详细讲述内核将页从内存写入交换区及从交换区读回内存所采用的技术。

内核利用了另一个缓存，称为交换缓存（*swap cache*），该缓存在选择换出页的操作和实际执行页交换的机制之间，充当协调者。

- [ ] 图，交换缓存与页交换子系统其他组件的交互

#### 标识换出页
内核必须能够正确标识出换出页，即必须能够根据给定的虚拟地址，找到内存页在交换区中的地址。

换出页在页表中通过一种专门的页表项来标记，其格式取决于所用的处理器体系结构。每个系统都使用了特定的编码，以满足特定的需求。在换出页的页表中，所有 *CPU* 都会存储下列信息：
- 一个标志，表明页已经换出
- 该页所在交换区的编号
- 对应槽位的偏移量，用于在交换区中查找该页所在的槽位

内核定义了一个体系结构无关的格式，可用于在交换区中确定页所在的位置，该格式可以（通过特定于处理器的代码）从体系结构相关的数据得出。在体系结构无关的表示中，内核必须存储交换分区的标识（也称为类型）和该交换区内部的偏移量，以便唯一确定一页。该信息保存在一个专门的数据类型中，称为 *swap_entry_t*：
```c
typedef struct {
  unsigned long val;
} swap_entry_t;
```
- 通过选择该成员中不同的比特位，来得出两个对应的信息项

为确保对 swap_entry_t 中两个信息项的访问，内核对比其中特位的布局定义了两个常数：
```c
#define SWP_TYPE_SHIFT(e)   (sizeof(e.val) * 8 - MAX_SWAPFILES_SHIFT)
#define SWP_OFFSET_MASK(e) ((1UL << SWP_TYPE_SHIFT(e)) - 1)
```
- `MAX_SWAPFILES_SHIFT` 值为 5，与平台无关。

从上述结构中提取各部分值的函数以及根据给出的类型和偏移量对产生一个 *swap_entry_t* 实例：
```c
static inline unsigned swp_type(swp_entry_t entry) {
  ...
}

static inline pgoff_t swp_offset(swp_entry_t entry) {
  ...
}

static inline swp_entry_t swp_entry(unsigned long type, pgoff_t offset) {
  ...
}
```

内核需要能够在体系结构无关和相关的两种页表项表示之间进行切换，而 `pte_to_swp_entry()` 函数正是为此而提供的。

查找可寻址的最大交换区偏移量：
```c
maxpages = swp_offset(pte_to_swp_entry(swp_entry_to_pte(swp_entry(0, ~0UL)))) - 1;
```
- `swp_entry(0, ~0UL)` 指定了一个所有比特位均置位的交换区偏移量

#### 交换缓存的结构
就数据结构而言，交换缓存无非是一个页缓存。其实现的核心是 `swapper_space` 对象，该对象中聚集了与交换缓存相关的内部函数和数据结构
```c
struct address_space swapper_space = {
  .page_tree = RADIX_TREE_INIT(GFP_ATOMIC | __GFP_NOWARN),
  .tree_lock = RW_LOCK_UNLOCKED(swapper_space.tree_lock),
  .a_ops = &swap_aops,
  .i_mmap_nonlinear = LIST_HEAD_INIT(swapper_space.i_mmap_nonlinear),
  .backing_dev_info = &swap_backing_dev_info,
};
```

尽管每个系统都可能有几个交换区，但页交换子系统以外的内核代码只通过一个变量来访问交换缓存。在数据实际写回之前，页并不按交换区进行组织。从确定换出页那部分内核代码的角度看，只需要向一个交换缓存发送适当的指令，该缓存由上述 `swapper_sapce` 对象表示。

内核提供了一组交换缓存访问函数，可以由任何涉及内存管理的内核代码使用。例如，这些函数可用于向交换缓存添加页，或查找交换缓存中的页。这些函数构成了交换缓存和页面替换逻辑之间的接口，因而可用于发出换入/换出页的命令，而无须关注此后数据如何传输的技术细节。

内核还提供了一组函数，来处理通过交换缓存提供的地址空间。与地址空间和页缓存类似，这些函数聚集在一个 *address_space_operations* 实例中，通过 `a_ops` 成员关联到 `swapper_space`。这些函数构成了交换缓存“向下”的接口。换言之，在交换缓存下是在系统的交换区和物理内存之间传输数据的实现部分，这些函数是交换缓存与数据传输部分之间的接口。与稍早提到的函数集不同，这些例程并不关注换出/换入哪些页，而负责对选定页进行数据传输的技术细节。

`swap_aops` 定义如下：
```c
static struct address_space_operations swap_aops = {
  .writepage = swap_writepage,
  .sync_page = block_sync_page,
  .set_page_dirty = __set_page_dirty_nobuffers,
};
```
- `swap_writepage` 将脏页与底层块设备同步。其目的并非像其他页缓存那样，用来维护物理内存和块设备之间的一致性。其目的是将页从交换缓存移除，将其数据传输到交换区。因而，该函数负责从物理内存到磁盘上交换区的数据传输；
- 页必须在交换缓存中标记为“脏”，而绝不能分配新的内存，因为在使用换出机制时，内存资源肯定已经匮乏到一定程度了。在第十六章讨论过，一种将页标记为脏的可能做法是创建缓冲区，使数据逐块回写。但这种做法需要额外的内存来保存 *buffer_head* 实例。因为交换缓存中只回写整页，所以这是无意义的。因此内核使用 `__set_page_dirty_nobuffers()` 函数来将页标记为脏，它设置 `PG_dirty` 标志但不创建缓冲区。
- 与多数页缓存一样，交换缓存也使用了内核的标准实现（`block_sync_page()`）来将页同步到交换区。该函数只负责拔出对应的块设备队列。就交换缓存而言，这意味着接下来将执行发送给块层的所有数据传输请求。

- [ ] 图，实现页交换和页面回收的最重要的函数的概述...

#### 添加新页
向交换缓存添加新页只需要使用适当的页缓存机制。标准方法就是调用 `add_to_page_cache()`，该函数将一个给定页的数据结构插入到 `swapper_space` 地址空间中对应的链表和树中。

但这并非任务的全部内容。该页不仅需要添加到交换缓存中，还需要在某个交换区中分配空间。尽管数据不会在此时复制到硬盘，但内核仍然必须考虑为该页选择的交换区和对应的槽位。该决策必须保存到交换缓存的数据结构中。

下面两个内核方法可以向交换缓存添加页：
- `add_to_swap()`：在内核想要主动换出一页时，会调用该函数，即当策略算法确定可用内存不足时。该例程不仅将页添加到交换缓存（在页数据写出到磁盘之前，会一直停留在其中），还在某个交换区中为该页分配一个槽位。
- `add_to_swap_cache()`：当从交换区读入由几个进程共享的一页（可以根据交换区中的使用计数器判定）时，该页将同时保持在交换区和交换缓存中，直至被再次换出，或被所有共享该页的进程换入。内核通过 `add_to_swap_cache()` 函数实现该行为，该函数将一页添加到交换缓存，而不对交换区进行操作。

1. 分配槽位
  内核将在交换区中分配槽位的任务委托给 `get_swap_page()`，该函数没有参数，将返回接下来所使用槽位的编号。

  该函数首先必须确保系统确实有交换区，标志是全局变量 `nr_swap_pages` 的值大于 0。
  
  `swap_list.next` 总是当前使用的交换区的编号（如果只有一个交换区，该值总是相同的）。逻辑上，内核开始在该交换区中查找一个空闲槽位。`scan_swap_map()` 扫描槽位分配位图，利用聚集技术。

  如果在当前交换区灭有找到空闲槽位，内核将检查备用的交换区。为此，内核将扫描所有交换区的列表（根据交换区的优先级），直至找到一个空闲的槽位。在到达优先级最低的交换区后，内核将返回到开头（优先级最高的交换区）重新开始搜索。如果遍历了系统中所有的交换区，也无法找到空闲槽位，则搜索结束。此时，内核是无法换出页的，需要向调用方代码返回页号 0，表示发生了这种情况。

  关于对聚集的处理，略...

2. 分配交换空间
  在策略例程确定了需要换出的页之后，`mm/filemap.c` 中的 `add_to_swap()` 开始运作。该函数接收一个 `struct page` 实例作为参数，并将换出请求转发给页交换的技术实现部分。

  代码流程：
  ```c
  add_to_swap()
    get_swap_page()
    __add_to_swap_page_cache()
    SetPageUptodata()
    SetPageDirty()
  ```
  - 在某个交换区中分配了一个槽位之后，只需要将换出页移动到交换区。这由 `__add_to_swap_page_cache()` 函数负责。
  - 最后，`SetPageUpdate()` 和 `SetPageDirty()` 会适当地修改页的标志。应该将页设置为脏，因为页的内容尚未包含在交换区中。页缓存中的页（第十七章）变脏后才会与底层块设备同步，对于交换页来说，对应的底层块设备是交换区，因而同步（几乎）就等价于将页换出。在页写入到对应的交换区槽位之后，剩下的工作就是更新页表，来反映这一事实。

  在换出页时，对策略例程没有其他的要求。剩余的工作，特别是将数据从内存传输到交换区的任务，是由与 `swapper_space` 关联的特定于地址空间的操作完成的。就策略例程来说，只需要知道：内核会负责将页数据实际写出到交换区，因而在调用 `add_to_swap()` 之后，就释放了一页。

3. 缓存交换页
  与 `add_to_swap()` 不同，`add_to_swap_cache()` 将一页添加到交换缓存，但要求已经为该页分配了一个槽位。

  如果一页已经有了对应的槽位，为什么将其添加到交换缓存呢？...

  `add_to_swap()` 和 `add_to_swap_cache()` 的主要区别在于，后者没有设置 `PG_uptodate` 或 `PG_dirty` 这两个标志。本质上，这意味着内核并不需要将页写入到交换区，即二者的内容当前是同步的。（注：在内核版本 2.6.25 中，将会对这里提到的函数名做一些调整。`add_to_swap_cache()` 将合并到其唯一的调用者 `read_swap_cache_async()` 中，不会继续存在。而 `__add_to_swap_cache()` 将替换前者，重命名为 `add_to_swap_cache()`。这两个函数的调用者也会相应进行更新。）

#### 搜索一页
`lookup_swap_cache()` 检查一页当前是否位于交换缓存中。如果没有找到页，则代码返回一个 `NULL` 指针，此时内核必须从硬盘取得数据。

### 数据回写
页交换实现的另一部分是其“向下”的接口，用于将页数据写入到交换区中选定的位置（或确切地说，向块层发出适当的请求）。这是在交换缓存中使用地址空间操作 `writepage()` 完成的，该函数指针指向 `swap_writepage(), mm/page_io.c`。

`swap_writepage()` 代码流程：
```c
swap_writepage()
  remove_exclusive_swap_page()
  get_swap_bio()
    map_swap_page()
    // 设置 bio 实例
  set_page_writeback()
  submit_bio()
```
流程解析，略...

注意，将页的内容写入到交换区中对应的槽位后，换出页的工作还没有完全结束。这时还需要更新页表，然后才能认为该页已经完全从物理内存移除。一方面，页表项需要指定该页不在内存中；另一方面，页表项还需要指向对应槽位在交换区中的位置。这项修改必须对所有使用该页的进程进行。

### 页面回收
到现在为止，已经解释了回写的技术细节，下面把注意力转向页交换子系统的第二个主要的方面：即交换策略。该策略用于确定哪些页可以从物理内存换出而同时又不会严重降低内核性能。因为该策略可以释放页帧，使得有新的内存可用于紧急需求，所以该技术也称为页面回收（*page reclaim*）。

与前几节关注交换地址空间中的页相比，本节关注的是任何地址空间中的页。交换策略的原理可以适用于所有没有后备存储器的页，无论其数据读取自文件，还是动态生成的。唯一的差别在于，内核决定将相关的页从内存移除时，页数据所写入的位置。而这个问题对页是否换出没有影响。有些页有持久后备存储器，页数据可以写出到后备存储器；而其他页则必须放入到交换区中。

交换策略算法的实现是内核中比较复杂的内容。这不仅是因为要使交换速度最大化，而更主要的是各种必须解决的特殊情况。下面的例子专注于构成页交换子系统主要工作的那些最常见的情况。这里不会讨论罕见的情况，这些可能是因 *SMP* 系统上各处理器间交互而导致的，或是单处理器系统上出现的随机巧合情况。

#### 概述
- [ ] 图，页面回收实现的“整体图示”。...
- 如果内核检测到在某个操作期间内存严重不足，将调用 `try_to_free_pages()`。该函数检查当前内存域中所有的页，并释放最不常用的那些。
- 一个后台守护进程，名为 *kswapd*，会定期检查内存使用情况，并检测即将发生的内存不足。可使用该守护进程换出页，作为预防措施，以防内核在执行其他操作期间发现内存不足。

它们的执行路径在 `shrink_zone()` 函数中合并。

在 *NUMA* 机器上，所有处理器对内存的共享并不是一致的，对每个 *NUMA* 结点来说，都有一个单独的 *kswapd* 守护进程。每个守护进程负责一个 *NUMA* 结点上所有的内存域。
在非 *NUMA* 系统上，只有一个 *kswapd* 实例，负责系统中所有的内存域（如 *DMA* 内存域、*Normal* 内存域、高端内存域）。

在 `try_to_free_pages()` 中处理系统内存严重不足时，以及在 *kswapd* 守护进程中定期检查内存使用时，在使用为此设计的算法确定为向系统提供新的空闲内存所需换出的页数以后，内核还需要确定具体换出哪些页（并最终将相关信息从策略代码部分传递到负责将页写回到后备存储器的例程，以及负责修改页表项的例程）。

内核试图将页分类到两个 *LRU* 链表中：一个用于活动页，另一个用于不活动页。这些链表是按内存域管理的：
```c
struct zone {
  struct list_head active_list;
  struct list_head inactive_list;
};
```

判定给定页属于哪一类是内核的一项必要工作，本章的很大一部分内容都在回答该问题。

有关回收页的数目、具体回收哪些页的决策，是按下列步骤作出的：
1. `shrink_zone()` 是从内存移除很少使用的页的入口点。该方法负责两件事：通过在活动链表和惰性链表之间移动页（使用 `shrink_active_list()`），试图在一个内存域中维护活动页和不活动页的数目的均衡；还通过 `shrink_cache()`，控制了选择换出页的过程。在确定内存域中换出页数的逻辑和具体换出哪些页的决策之间，`shrink_zone()` 充当了一个中间层。
2. `shrink_active_list()` 是一个综合性的辅助函数，内核使用该函数在活动页和不活动页的两个链表之间移动页。该函数会被告知需要在两个链表之间转移的页数，而后该函数试图选择使用最少的页。因而在本质上，`shrink_active_list()` 负责决定随后将换出哪些页，保留哪些页。换言之，该函数实现了页面选择的策略部分。
3. `shrink_inactive_list()` 从给定内存域的惰性链表移除选定数目的不活动页，将其传送到 `shrink_page_list()` 函数，后者将向各个对应的后备存储器发出回写数据的请求，以便在物理内存中释放空间，回收所选定的页。如果由于任何原因，不能回写页，`shrink_inactive_list()` 必须将不能回写的页放回活动链表或惰性链表。

#### 数据结构
1. 页向量
   - 页向量，即借助于一个数组来保存特定数目的页，可以对这些页执行同样的操作。最好以“批处理模式”执行，这比分别对每个页执行同样的操作要快得多。
   - 结构体定义：
    ```c
    struct pagevec {
      unsigned nr;
      int cold;
      struct page *pages[PAGEVEC_SIZE];
    };
    ```
    - `PAGEVEC_SIZE`（默认值是 14）
    - `cold` 成员用于帮助内核区分热页（*hot page*，在 *CPU* 的高速缓存中）和冷页(*cold page*，不在 *CPU* 的高速缓存中)
    - 内核对此提供的相关函数主要涉及页的释放：
      - `pagevec_release()` 将向量中所有页的使用计数器减 1。如果某些页的使用计数器归 0，即不再使用，则自动返回到伙伴系统。如果页在系统的某个 *LRU* 链表上，则从该链表移除，无论其使用计数器为何值。
      - `pagevec_free()` 将一组页占用的内存空间返还给伙伴系统。调用者负责确认页的使用计数器为 0，且未包含在任何LRU链表中。
      - `pagevec_release_nonlru()` 是另一个用于释放页的函数，它将一个给定页向量中所有页的使用计数器减 1。在计数器归 0 时，对应页占用的内存将返还给伙伴系统。与 `pagevec_release()` 不同，该函数假定向量中所有的页都不在任何 *LRU* 链表上。
      - 这些函数还有另一种带有两个下划线的版本（例如 `__pagevec_release()`），它们并不测试向量是否包含页。
      - 向页向量添加页的函数：`static inline unsigned pagevec_add(struct pagevec *pvec, struct page *page)`
2. *LRQ* 缓存
   - 将页置于内存域的活动链表或惰性链表上的 *LRU* 缓存机制

#### 确定页的活动程度
#### 收缩内存域
1. 控制扫描
2. 实现
#### 隔离 LRQ 页和集中回收
#### 收缩活动页链表
#### 回收不活动页
1. 收缩惰性链表
2. 执行页面回收

### 交换令牌

### 处理交换缺页异常
#### 换入页
#### 读取数据
#### 交换预读

### 发起内存回收
#### 用 kswapd 进行周期性内存回收
#### 在严重内存不足时换出页

### 收缩其他缓存
#### 数据结构
#### 注册和删除收缩期
#### 收缩缓存

### 小结

## 附录 D-系统启动
### initcall 机制