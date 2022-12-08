# Lab 4: Preemptive Multitasking（可抢占式多任务）

## 介绍

在本实验室中，你将在多个同时活跃的用户模式环境中实现抢占式多任务处理。

在 A 部分中，你将为 JOS 添加多处理器支持，实现循环调度，并添加基本的环境管理系统调用(创建和销毁环境的调用，以及分配/映射内存的调用)。

在 B 部分，你将实现一个类 unix 的 fork()，它允许用户模式环境创建自身的副本。

最后，在 C 部分中，你将添加对进程间通信(IPC)的支持，允许不同的用户环境显式地相互通信和同步。你还将添加对硬件时钟中断和抢占的支持。

### 开始

[omit]

### 实验要求

该课程被分为三部分，A，B 和 C。我们每周完成一个部分。

和以前一样，你需要做所有在实验室中描述的常规练习和至少一个挑战性问题。(你不需要每个部分做一个挑战题，整个实验室只做一个。)另外，你需要对你所实现的挑战问题写一个简短的描述。如果你实现了多个挑战问题，你只需要在报告中描述其中一个，当然也欢迎你做更多。在提交作业之前，将作业记录放在实验室目录的顶层名为 answers-lab4.txt 的文件中。

## Part A: 多处理器支持和协同多任务处理

在本实验的第一部分中，你将首先扩展 JOS 以在多处理器系统上运行，然后实现一些新的 JOS 内核系统调用，以允许用户级环境创建额外的新环境。你还将实现协作轮询调度，允许内核在当前环境自愿放弃 CPU(或退出)时从一个环境切换到另一个环境。在后面的 C 部分中，你将实现抢占式调度，它允许内核在经过一段时间后重新从环境中获得对 CPU 的控制，那怕该环境不合作。

### 多处理器支持

我们打算让 JOS 支持“对称多处理”(SMP)，这是一种多处理器模型，在这种模型中，所有 CPU 对系统资源(如内存和 I/O 总线)都有同等的访问权限。虽然在 SMP 中，所有 CPU 的功能都是相同的，但在引导过程中，它们可以分为两类：引导处理器(bootstrap processor, BSP)负责初始化系统和引导操作系统；只有在操作系统启动并运行之后，BSP 才会激活应用程序处理器(AP)。哪个处理器是 BSP 由硬件和 BIOS 决定。到目前为止，所有现有的 JOS 代码都在 BSP 上运行。

在 SMP 系统中，每个 CPU 都有一个相应的本地 APIC (LAPIC)单元。LAPIC 单元负责在整个系统中传递中断。LAPIC 还为其连接的 CPU 提供一个唯一标识符。在本实验室中，我们使用了以下 LAPIC 单元的基本功能(在 kern/LAPIC.c 中)：

-   读取 LAPIC 标识符(APIC ID)来告诉我们的代码当前运行在哪个 CPU 上(参见 cpunum())。
-   从 BSP 向 AP 发送 STARTUP interprocessor interrupt (IPI)来启动其他 CPU(参见 lapic_startap())。
-   在 C 语言部分，我们编写了 LAPIC 的内置定时器来触发时钟中断，以支持抢占式多任务(参见 apic_init())。

处理器使用内存映射 I/O (MMIO)访问它的 LAPIC。在 MMIO 中，物理内存的一部分硬连接到一些 I/O 设备的寄存器，因此通常用于访问内存的加载/存储指令也可以用于访问设备寄存器。你已经看到物理地址 0xA0000 上有一个 IO 孔(我们使用它来写入 VGA 显示缓冲区)。LAPIC 位于一个从物理地址 0xFE000000 (32MB 差 4GB)开始的漏洞中，因此我们无法在 KERNBASE 上使用通常的直接映射来访问它。JOS 虚拟内存映射在 MMIOBASE 中留下了 4MB 的空白，所以我们有一个地方可以像这样映射设备。由于后面的实验引入了更多的 MMIO 区域，因此你将编写一个简单的函数来从该区域分配空间并将设备内存映射到该区域。

练习 1。在 kern/map.c 中实现 mmio_map_region。要了解如何使用它，请查看 kern/lapic.c 中 lapic_init 的开头。在运行 mmio_map_region 的测试之前，你也必须进行下一个练习。

#### Application Processor Bootstrap（AP 处理器的启动）

在启动 AP 之前，BSP 应该首先收集多处理器系统的信息，如 CPU 总数、APIC ID 和 LAPIC 单元的 MMIO 地址。kern/mpconfig.c 中的 mp_init()函数通过读取驻留在 BIOS 内存区域中的 MP 配置表来获取这些信息。

boot_aps()函数(在 kern/init.c 中)驱动 AP 引导进程。AP 在实模式下启动，就像引导加载程序在 boot/boot.S 中启动一样。因此 boot_aps()将 AP 入口代码(kern/mpentry.S)复制到一个在实际模式下可寻址的内存位置。与引导加载器不同的是，我们对 AP 从哪里开始执行代码有一些控制;我们将入口代码复制到 0x7000 (MPENTRY_PADDR)，但是任何未使用的、页面对齐的、低于 640KB 的物理地址都可以工作。

在此之后，boot_aps()通过向相应 AP 的 LAPIC 单元发送 STARTUP IPIs 以及初始 CS:IP 地址依次激活 AP, AP 应该在该 IP 地址开始运行它的入口代码(在本例中为 MPENTRY_PADDR)。kern/mpentry.S 中的入口代码。与 boot/boot.S 非常相似。经过一些简短的设置后，它将 AP 放入启用分页的保护模式，然后调用 C 设置例程 mp_main()(也在 kern/init.c 中)。boot_aps()等待 AP 在其结构体 CpuInfo 的 cpu_status 字段中发出 CPU_STARTED 标志，然后继续唤醒下一个 AP。

练习 2。阅读 kern/init.c 中的 boot_aps()和 mp_main()，阅读 kern/mpentry.S 中的汇编代码。确保你理解 ap 引导过程中的控制流传输。然后修改 kern/pmap.c 中的 page_init()的实现，以避免将 MPENTRY_PADDR 上的页面添加到空闲列表中，这样我们就可以安全地复制并运行该物理地址上的 AP 引导代码。你的代码应该通过更新的 check_page_free_list()测试(但可能无法通过更新的 check_kern_pgdir()测试，我们将很快修复它)。

问题

1. 一行一行地对比 kern/mpentry.S 和 boot/boot.S。记住 kern/mpentry.S 被编译和链接，然后运行在 KERNBASE 之上，就像内核的其他部分一样，MPBOOTPHYS 宏的目的是什么？为什么在 kern/mpentry.S 里而不是 boot/boot.S 里？换言之，如果把它在 kern/mpentry.S 里去掉会发生什么？
   提示：重谈我们在实验一里讨论过的链接地址和加载地址的区别。

#### Per-CPU State and Initialization（每个 CPU 的状态及其初始化）

在编写多处理器操作系统时，区分每个处理器私有的每 cpu 状态和整个系统共享的全局状态是很重要的。kern/cpu.h 定义了每个 cpu 的大部分状态，包括 struct CpuInfo，它存储每个 cpu 的变量。cpunum()总是返回调用它的 CPU 的 ID，它可以用作 CPU 等数组的索引。或者说，宏 thiscpu 是当前 CPU 结构体 CpuInfo 的简写。

这里是你需要注意的每个 CPU 的状态：

-   每个 CPU 的内核栈（kernel stack）
    因为多 CPU 可以同时陷入内核，所以我们需要为每一个 CPU 弄一个内核栈，来防止它们相互干涉对方的执行。数组 percpu_kstacks[NCPU][kstksize]为 NCPU 保留了内核栈的空间。

    在实验 2 中，你已经在物理地址的 KSTACKTOP 下为 BSP 映射了内核栈，用 bootstack 来引用。同样地，在这个实验中，你会映射每个 CPU 的内核栈到某些守护内存页的区域，作为 CPU 之间的缓存。CPU 0 的栈还是会从 STACKTOP 往下增长；CPU 1 的栈会从 CPU 0 栈的底部更低 KSTKGAP 比特的位置开始，以此类推。inc/memlayout.h 展示了这个映射的布局。

-   每个 CPU 的 TSS 和 TSS 描述符
    为了指定每一个 CPU 的内核栈都存在于哪里，需要每个 CPU 的任务状态段（TSS）。CPU i 的 TSS 存放在 cpu[i].cpu_ts 中，对应的 TSS 描述符定义在 GDT 项 gdt[(GD_TSS0 >> 3) + i]里。全局 ts 变量定义在 kern/trap.c 不会再有用处。

-   每个 CPU 当前运行的环境的指针
    因为每个 CPU 可以同时运行用户进程，所以我们重新定义符号 curenv，让它指向 cpus[cpunum()].cpu_env（或者是 thiscpu->cpu_env），也就是指向当前 CPU 正在执行的环境（当前代码执行在哪个 CPU 上）。

-   每个 CPU 的系统寄存器
    所有寄存器，包括系统寄存器，都是 CPU 私有的。因此，用来初始化这些寄存器的指令，比如说 lcr3(), ltr(), lgdt(), lidt()等，必须在每个 CPU 上都执行一次。函数 env_init_percpu()和 trap_init_percpu()是为这种目的而设定的。

    除此之外，如果你在你的解决方案中添加了任何额外的每 CPU 状态或执行了任何额外的特定于 CPU 的初始化（比如，在 CPU 寄存器中设置新位）以挑战早期实验室中的问题，请确保每个 CPU 上都复制了它们。

练习 3。修改 kern/pmap.c 里的 mem_init_mp()，使其映射每个 CPU 栈到 KSTACKTOP 的初始位置，正如 inc/memlayout.h 里展示的那样。每个堆栈的大小是 KSTKSIZE 字节加上未映射保护页的 KSTKGAP 字节。你的代码应该通过新测试 check_kern_pgdir()。

练习 4。trap_init_percpu()（位于 kern/trap.c）里的代码为 BSP 初始化 TSS 和 TSS 描述符。这个函数可以在实验三里正常运行，但是它在其他 CPU 上运行就会出错。修改代码，让它可以在所有的 CPU 上运行。（注意：你的新代码不应该再使用全局 ts 变量。）

当你完成了以上练习，在 QEMU 中以用 4 个 CPU 来运行 JOS，`make qemu CPUS=4`或者`make qemu-nox CPUS=4`，你应该可以看到这样的输出：

```
...
Physical memory: 66556K available, base = 640K, extended = 65532K
check_page_alloc() succeeded!
check_page() succeeded!
check_kern_pgdir() succeeded!
check_page_installed_pgdir() succeeded!
SMP: CPU 0 found 4 CPU(s)
enabled interrupts: 1 2
SMP: CPU 1 starting
SMP: CPU 2 starting
SMP: CPU 3 starting
```

#### Locking（锁）

我们当前的代码 mp_main()在初始化 AP 之后就开始自旋。在进一步之前，我们需要弄清楚多个 CPU 运行内核代码时的竞争条件。最简单的方式是使用 big kernel lock。big kernel lock 是单个全局的锁，当一个环境进入内核态的时候获取，当该环境退出内核态的时候释放。在这个模型中，用户态代码可以同时运行在多个 CPU 上，但是同一时刻不能超过一个用户环境运行在内核态上；任何想要进入内核态的环境都被迫等待。

kern/spinlock.h 声明了 big kernel lock，命名为 kernel_lock。它也提供了 lock_kernel()和 unlock_kernel()函数来获取和释放锁。你应该在这四个地方使用 big kernel lock：

-   i386_init()，在 BSP 唤醒其他 CPU 的之前获得锁。
-   mp_main()，初始化 AP 之后获取锁，接着调用 sched_yield()让 AP 开始运行环境。
-   trap()，用户态陷入内核态时获取锁。为了决定该陷入发生在用户态还是内核态，查看 tf_cs 的低位。
-   env_run()，切换到用户态时释放锁。不要太早也不要太晚做这件事，否则你会遇到竞争和死锁。

练习 5。应用上述的 big kernel lock，在适当的地方调用 lock_kernel() 和 unlock_kernel()。

如何测试你的锁是否之正常工作？此时还不行！但是你在下一个练习里可以实现调度器。

问题

1. 看似使用 big kernel lock 可以保证某一时刻只有一个 CPU 可以运行内核代码，那为什么我们需要给每个 CPU 都弄一个内核栈呢？在什么场景下，共享的内核栈会出错，哪怕是在 big kernel lock 保护下？

挑战！big kernel lock 简单易用。尽管如此，它消除了内核态的并行。大多数现代操作系统使用不同的锁来保护不同的部分，一个方法是 fine-grained locking。fine-grained locking 可以显著增加性能，但是它更难以实现，也很容易出错。如果你足够勇敢，丢弃 big kernel lock，让 JOS 拥抱并行。

由你决定锁的粒度（该锁要保护多少数据）。作为提示，你可能要考虑使用自旋锁来保证 JOS 内核里共享组件的排他访问：

-   The page allocator.
-   The console driver.
-   The scheduler.
-   The inter-process communication (IPC) state that you will implement in the part C.

### Round-Robin Scheduling（循环调度）

你的下一个任务是去改变 JO 内核，以便于它可以循环切换多环境。循环调度的运行如下：

-   kern/sched.c 里的 sched_yield()负责选取一个新的环境来运行。它按照环形顺序来搜索 envs[]数组，从刚刚运行的环境开始搜索起（或者刚刚如果没有运行的环境，那就从最开头开始开始搜索），选取第一个它发现的 ENV_RUNNABLE 状态的环境，然后调用 env_run()来跳进去执行这个环境。
-   sched_yield()永远不要在两个 CPU 上运行同一个环境。看环境的状态为 ENV_RUNNING 就知道它当前运行在一些 CPU 上（很有可能就是当前 CPU）。
-   我们已经为你实现了一个新的系统调用 sys_yield()，用户环境可以调用这个来调用内核的 sched_yield()函数，这样就自愿地放弃 CPU，让给其他环境。

练习 6。在 sched_yield()里实现如上所述的循环调度。不要忘记修改 syscall()来分发 sys_yield()系统调用。

确保在 mp_main 里调用了 sched_yield()。

修改 kern/init.c 来创建三个（甚至更多）的环境，让他们都运行程序 user/yield.c。

运行`make qemu`。你应该看到环境切换来切换去，五次之后终止运行，如下所示。

测试多 CPU：`make qemu CPUS=2`.

```
...
Hello, I am environment 00001000.
Hello, I am environment 00001001.
Hello, I am environment 00001002.
Back in environment 00001000, iteration 0.
Back in environment 00001001, iteration 0.
Back in environment 00001002, iteration 0.
Back in environment 00001000, iteration 1.
Back in environment 00001001, iteration 1.
Back in environment 00001002, iteration 1.
...
```

在 yield 程序退出后，系统里就没有可运行的环境了，调度器应该触发 JOS 内核监控器。如果没有发生，那就修改你的代码，之后再进行下一步。

问题

1. 在你的 env_run()实现中，你应该调用了 lcr3()。在调用 lcr3()前后，你的代码对变量 e 做了引用，也就是 env_run 的参数。在加载%cr3 寄存器之后，MMU 使用的上下文一下子被切换了。但是一个虚拟地址和地址上下文有联系——地址上下文指出虚拟地址要映射到哪个物理地址上。为什么指针 e 依旧可以在地址切换前后进行取值？
2. 不管内核从一个环境切换到另外一个环境，它必须确保老环境寄存器被保存起来了，这样它们就可以正确地恢复。为什么？这个在哪里发生的？

挑战!向内核添加一个不那么简单的调度策略，比如一个固定优先级的调度器，它允许为每个环境分配一个优先级，并确保总是选择高优先级的环境，而不是低优先级的环境。如果你真的想冒险，尝试实现一个 unix 风格的可调优先级调度程序，或者甚至是一个彩票或跨步调度程序。(在谷歌中查找"lottery scheduling"和"stride scheduling"。)

编写一两个测试程序来验证你的调度算法是否正确工作(即，正确的环境以正确的顺序运行)。在本实验室的 B 部分和 C 部分实现了 fork()和 IPC 之后，编写这些测试程序可能会更容易。

挑战!JOS 内核目前不允许应用程序使用 x86 处理器的 x87 浮点单元(FPU)、MMX 指令或流 SIMD 扩展(SSE)。扩展 Env 结构，为处理器的浮点状态提供一个保存区域，并扩展上下文切换代码，以便在从一个环境切换到另一个环境时正确地保存和恢复这个状态。FXSAVE 和 FXRSTOR 指令可能是有用的，但请注意，旧的 i386 用户手册中没有这些指令，因为它们是在最新的处理器中引入的。编写一个用户级的测试程序，用浮点做一些很酷的事情。

### System Calls for Environment Creation（进行环境创建的系统调用）

尽管你的内核现在能够在多个用户级环境之间运行和切换，但它仍然局限于内核最初设置的运行环境。现在，你将实现必要的 JOS 系统调用，以允许用户环境创建和启动其他新用户环境。

Unix 提供 fork()系统调用作为它的进程创建原语。Unix fork()复制调用进程(父进程)的整个地址空间来创建一个新进程(子进程)。用户空间中两个可观察对象之间的唯一区别是它们的进程 id 和父进程 id(由 getpid 和 getppid 返回)。在父进程中，fork()返回子进程的进程 ID，而在子进程中，fork()返回 0。默认情况下，每个进程都有自己的私有地址空间，两个进程对内存的修改都不可见。

你将提供一组不同的、更原始的 JOS 系统调用来创建新的用户模式环境。通过这些系统调用，你将能够完全在用户空间中实现类 unix 的 fork()，以及其他类型的环境创建。你将为 JOS 编写的新系统调用如下:

-   sys_exofork：
    这个系统创建一个新的环境，几乎空白的环境：没有地址空间的映射，不可运行。当调用该系统调用时，新环境会和父环境拥有相同的寄存器状态。在父环境中，sys_exofork 将返回新创建的环境的 envid_t(如果环境分配失败，则返回负的错误代码)。然而，在子进程中，它将返回 0。(由于子进程一开始被标记为不可运行，sys_exofork 实际上不会在子进程中返回，直到父进程使用....显式地将子进程标记为可运行。)
-   sys_env_set_status：
    设置指定环境的状态为 ENV_RUNNABLE 或 ENV_NOT_RUNNABLE。这个系统调用通常用于标记一个新环境准备运行，一旦它的地址空间和寄存器状态已经完全初始化。
-   sys_page_alloc:
    分配一页物理内存，并将其映射到给定环境的地址空间中的给定虚拟地址。
-   sys_page_map:
    将一个页面映射(而不是页面的内容!)从一个环境的地址空间复制到另一个环境，保留一个内存共享安排，以便新映射和旧映射都引用物理内存的同一页。
-   sys_page_unmap:
    取消在给定环境中给定虚拟地址映射的页的映射。

对于以上接受环境 id 的所有系统调用，JOS 内核支持这样的约定:值 0 表示“当前环境”。这个约定由 kern/env.c 中的 envid2env()实现。

我们已经在测试程序 user/dumbfork.c 中提供了一个类 unix 的 fork()的非常原始的实现。这个测试程序使用上面的系统调用来创建和运行带有自己地址空间副本的子环境。然后，这两个环境使用 sys_yield 在前面的练习中来回切换。父进程在迭代 10 次后退出，而子进程在迭代 20 次后退出。

练习 7。实现 kern/syscall.c 中描述的系统调用，并确保 syscall()调用它们。你将需要使用 kern/pmap.c 和 kern/env.c 中的各种函数，特别是 envid2env()。现在，无论何时调用 envid2env()，都要在 checkperm 参数中传递 1。确保你检查了任何无效的系统调用参数，在这种情况下返回-E_INVAL。使用 user/dumbfork 测试你的 JOS 内核，确保它能正常工作。

挑战!添加必要的附加系统调用，以读取现有环境的所有重要状态并设置它。然后实现一个用户模式程序，它分叉一个子环境，运行它一段时间(例如，sys_yield()的几个迭代)，然后获取子环境的完整快照或检查点，运行子环境一段时间，最后将子环境恢复到它在检查点时的状态，并从那里继续它。因此，你可以有效地从中间状态“重放”子环境的执行。使孩子环境执行一些与用户的交互使用 sys_cgetc()或 readline(),以便用户可以查看和改变其内部状态,与你的检查站,并验证/重启可以给孩子环境的选择性失忆,这使得“忘记”,超过某特定点上发生的一切。

这完成了实验室的 A 部分;当你运行 make grade 时，确保它通过所有的 A 部分测试，然后像往常一样使用 make handin 交上来。如果你试图找出为什么某个特定的测试用例失败，运行./grade-lab4 -v，它将显示内核构建的输出，并为每个测试运行 QEMU，直到一个测试失败。当测试失败时，脚本将停止，然后你可以检查 jos。出来看看内核实际打印了什么。

## Part B: Copy-on-Write Fork（写时复制 Fork）

如前所述，Unix 提供 fork()系统调用作为它的主要进程创建原语。fork()系统调用复制调用进程(父进程)的地址空间，以创建一个新进程(子进程)。

xv6 Unix 通过将所有来自父节点的数据复制到分配给子节点的新页面来实现 fork()。这基本上与 dumbfork()采用的方法相同。将父节点的地址空间复制到子节点是 fork()操作中开销最大的部分。

然而，在调用 fork()之后，经常会立即在子进程中调用 exec()，这将用一个新程序替换子进程的内存。例如，这就是 shell 通常做的事情。在这种情况下，花在复制父进程地址空间上的时间基本上是浪费的，因为子进程在调用 exec()之前只会使用很少的内存。

由于这个原因，Unix 的后续版本利用虚拟内存硬件，允许父进程和子进程共享映射到各自地址空间的内存，直到其中一个进程真正修改它。这种技术称为写时复制。要做到这一点，在 fork()上，内核会将地址空间映射从父节点复制到子节点，而不是映射页面的内容，同时将现在共享的页面标记为只读。当两个进程中的一个试图写入这些共享页面时，该进程将出现页面错误。在这一点上，Unix 内核意识到页面实际上是一个“虚拟”或“写时复制”副本，因此它为出现故障的进程创建了一个新的、私有的、可写的页面副本。通过这种方式，在实际写入各个页面之前，不会实际复制各个页面的内容。这种优化使得子进程中 fork()后跟 exec()的代价更低:子进程在调用 exec()之前可能只需要复制一页(堆栈的当前页)。

在本实验的下一部分中，你将实现一个“适当的”类 unix 的带有 copy-on-write 的 fork()，作为一个用户空间库例程。在用户空间中实现 fork()和 copy-on-write 支持的好处是，内核仍然更简单，因此更有可能是正确的。它还允许单独的用户模式程序为 fork()定义它们自己的语义。如果一个程序需要一个稍微不同的实现(例如，像 dumbfork()这样代价昂贵的总是复制版本，或者父和子实际上在之后共享内存的版本)，那么它可以很容易地提供自己的实现。

### User-level page fault handling（用户级页错误处理）

用户级 write-copy-fork()需要知道写保护页面上的页面错误，所以这是你首先要实现的。写时复制只是用户级页面错误处理的许多可能用途之一。

设置一个地址空间是很常见的，这样页面错误就可以指示什么时候需要执行某些操作。例如，大多数 Unix 内核最初只映射一个新进程的堆栈区域中的单个页面，然后随着进程的堆栈消耗增加，“按需”分配和映射额外的堆栈页面，并导致尚未映射的堆栈地址的页面错误。典型的 Unix 内核必须跟踪在进程空间的每个区域发生页面错误时应该采取的操作。例如，堆栈区域中的故障通常会分配和映射物理内存的新页。程序的 BSS 区域中的错误通常会分配一个新页面，用零填充它，并映射它。在具有按需分页可执行文件的系统中，文本区域中的错误将从磁盘中读取相应的二进制文件页，然后映射它。

这是内核需要跟踪的大量信息。你将决定如何处理用户空间中的每个页面错误，而不是采用传统的 Unix 方法，在用户空间中，错误的破坏性较小。这种设计还有一个额外的好处，即允许程序在定义它们的内存区域时具有很大的灵活性;稍后，你将使用用户级页面错误处理来映射和访问基于磁盘的文件系统上的文件。

#### Setting the Page Fault Handler（设置页错误处理器）

为了处理自己的页面错误，一个用户环境需要注册一个页错误处理器端点到内核里。用户环境通过新的 sys_env_set_pgfault_upcall 系统调用来注册这个页错误处理端点。我们已经在 Env 结构体里添加了一个新成员，env_pgfault_upcall 来记录这个信息。

练习 8。实现 sys_env_set_pgfault_upcall 函数调用。确保在查询目标环境 id 的时候做了权限检查，因为这是一个“危险”的系统调用。

#### Normal and Exception Stacks in User Environments（用户环境的正常栈和异常栈）

当正常执行时，JOS 里的一个用户环境会运行在正常的用户栈上：它的 ESP 寄存器开始指向 USTACKTOP，它推送的数据驻留在 USTACKTOP-PGSIZE 和 USTACKTOP-1 之间。 当一个页错误出现在用户态时，然而，内核会重启用户环境去运行一个委托的用户级页错误处理程序在一个不同的栈上，这个栈被叫做用户异常栈。本质上，我们将让 JOS 内核实现代表用户环境的自动“堆栈切换”，就像 x86 处理器在从用户模式转换到内核模式时已经代表 JOS 实现了堆栈切换一样!

JOS 用户异常栈也是只有一页大小，它的顶部定义在虚拟地址的 UXSTACKTOP，所以确定的用户异常栈在 UXSTACKTOP-PGSIZE 到 UXSTACKTOP-1 之间。当运行在这个异常栈时，这个用户级页错误处理程序可以使用 JOS 的常规系统调用来映射新的页或者调整映射来恢复页面错误导致的任何问题。之后这个用户级页错误处理程序通过一个汇编 stub 返回错误码到原始栈上。

每一个想要支持用户级页错误处理的用户环境，都需要为自己的异常栈申请内存，使用 A 部分的 sys_page_alloc()系统调用。

#### Invoking the User Page Fault Handler（触发用户级页错误处理程序）

你现在需要去修改 kern/trap.c 里的页错误处理代码，来处理来自用户态的页错误。我们将用户环境发生错误时的状态称为 trap-time 状态。

如果没有事先注册好的页错误处理程序，JOS 内核就会像之前一样，破坏掉该用户环境，发出一个 message。否则，内核会在异常栈上建立一个 trap frame，看起来像是来自 inc/trap.h 的 UTrapframe 结构体。

```
                    <-- UXSTACKTOP
trap-time esp
trap-time eflags
trap-time eip
trap-time eax       start of struct PushRegs
trap-time ecx
trap-time edx
trap-time ebx
trap-time esp
trap-time ebp
trap-time esi
trap-time edi       end of struct PushRegs
tf_err (error code)
fault_va            <-- %esp when handler is run
```

内核通过构建这个栈帧来帮助恢复用户环境的运行；你必须弄清楚这个是如何发生的。fault_va 是导致页错误时的虚拟地址。

如果当一个错误出现时，用户环境已经运行在用户异常栈了，然后页错误处理程序它本身发送错误了。在这种情况下，你应该开始一个新的栈帧，就在当前的 tf->tf_esp 之下，而不是 UXSTACKTOP。你应该开始推入一个空的 32 位字，然后推入一个 UTrapframe 结构体。

要是想测试是否 tf->tf_esp 已经在用户异常栈上，那就查看它是否在 UXSTACKTOP-PGSIZE 和 UXSTACKTOP-1 之间。

练习 9。实现 kern/trap.c 里 page_fault_handler 函数的代码，这个函数在分发页错误的时候被用户级处理程序所需要。在写入异常堆栈时，请确保采取适当的预防措施。（如果用户环境在异常栈上用完了内存空间会怎么样？）

#### User-mode Page Fault Entrypoint（用户态页错误端点）

接着，你需要去实现一个汇编例程，它要负责调用 C 语言写的页错误处理程序以及恢复原始指令的执行。这个汇编例程会用 sys_env_set_pgfault_upcall()把它注册到内核里。

练习 10。采用 实现 lib/pfentry.S 里的\_pgfault_upcall 例程。有趣的部分是返回到原来产生页错误的用户代码处。你会直接回到那儿，不需要经过内核。难点在于同时切换栈和重新加载 EIP。

最终，你需要实现相关的 C 语言用户库。

练习 11。完成 lib/pgfault.c 里的 set_pgfault_handler()。

#### Testing（测试）

运行 user/faultread（make run-faultread）。你应该看到：

```
...
[00000000] new env 00001000
[00001000] user fault va 00000000 ip 0080003a
TRAP frame ...
[00001000] free env 00001000
```

Run user/faultdie. You should see:

```
...
[00000000] new env 00001000
i faulted at va deadbeef, err 6
[00001000] exiting gracefully
[00001000] free env 00001000

```

运行 user/faultalloc。你应该看到：

```
...
[00000000] new env 00001000
fault deadbeef
this string was faulted in at deadbeef
fault cafebffe
fault cafec000
this string was faulted in at cafebffe
[00001000] exiting gracefully
[00001000] free env 00001000
```

如果只看到"this string"这一行，这意味着你没有正确地处理递归页面错误。

运行 user/faultallocbad。你应该看到：

```
...
[00000000] new env 00001000
[00001000] user_mem_check assertion failure for va deadbeef
[00001000] free env 00001000
```

确保你理解了为什么 user/faultalloc 和 user/faultallocbad 的是不同的。

挑战！扩展你的内核，不仅限于页错误，而且还处理所有用户会产生的错误，可以重定向到用户态异常处理程序。写一些用户态的测试程序来测试用户态的不同错误，比如说除零错误，通用保护错误和非法操作码错误。

### Implementing Copy-on-Write Fork（实现写时复制 Fork）

现在，你拥有了完全在用户空间中实现 copy-on-write fork()的内核工具。

我们已经给你提供了一个 fork()的骨架在 lib/fork.c 文件里。像 dumbfork()，fork()应该创建一个新环境，然后扫描父环境的整个地址空间以及创建子环境对应的页映射。关键的不同是，dumbfork()复制页，而 fork()最初只会复制页映射。fork()只有当某个环境尝试写入时才会复制各个页。

fork()的基本控制流如下：

1. 父环境注册 pgfault()为 C 语言级别的页面错误处理程序，就用你之前实现的 set_pgfault_handler()函数来注册。
2. 父环境调用 sys_exofork()来创建一个子环境。
3. 对于它在 UTOP 之下地址空间里每一个可写或写时复制的页，父环境会调用 duppage，应该会映射该页到子环境的地址空间，然后重新映射自己地址空间的该页。【注意：这里的顺序是很重要的（为子环境标记一个页为 COW 要在为父环境标记之前）。你能明白吗？尝试想一个具体的调转顺序会发生问题的情况。】duppage 会在两个过程中都设置 PTE 位，这样每个页都是不可写的，“avail”字段里的 PTE_COW 用来标记写时复制页和真正的只读页。
   然而，异常栈不会通过这种方式重映射。而是，你需要在子环境里为异常栈申请一个新的页。因为页错误处理函数会做真正的复制，页错误处理函数也会运行在异常栈上，异常栈不能写时复制：谁会复制？

    fork()也需要掌管当前的页，但不是可写的或写时复制。

4. 父环境为子环境设置用户页错误处理端点，让子环境看起来这个就像是它自己设置的一样。
5. 子环境现在准备好运行了，所以父环境会标记它为 runnable。

每次某个环境写入一个“写时复制”页时，它会产生一个页错误。如下是用户页错误处理程序的控制流：

1. 内核将页面错误传播到 \_pgfault_upcall，它会调用 fork()的 pgfault()处理函数。
2. pgfault()查看这个错误是由一个写操作（查看错误码是不是 FEC_WR）导致的，以及页的 PTE 被标记为 PTE_COW。如果不是，panic。
3. pgfault()申请一个新的页，这个页映射到一个暂时地址，然后复制错误页的内容进去。然后错误处理程序映射一个新的页到合适的地址，设置好读写权限，以此代替旧的只读映射。

用户级别的 lib/fork.c 代码必须查看环境的页表来执行上面的几个操作(例如，页面的 PTE 标记为 PTE_COW)。内核在 UVPT 上映射环境的页表正是为了这个目的。它使用了一个[聪明的映射技巧](https://pdos.csail.mit.edu/6.828/2018/labs/lab4/uvpt.html)，使其更容易查找用户代码的 PTE。lib/entry.S 设置 uvpt 和 uvpd，这样你才可以很容易地在 lib/fork.c 里查找页表信息。

练习 12。实现 lib/fork.c 里的 fork，duppage 和 pgfault。

用 forktree 程序来测试你的代码。它应该产生如下消息，分散的‘new env’，‘free env’，‘exiting gracefully’消息。这个消息可能不会按顺序出现，以及环境 ID 可能不同。

```
	1000: I am ''
	1001: I am '0'
	2000: I am '00'
	2001: I am '000'
	1002: I am '1'
	3000: I am '11'
	3001: I am '10'
	4000: I am '100'
	1003: I am '01'
	5000: I am '010'
	4001: I am '011'
	2002: I am '110'
	1004: I am '001'
	1005: I am '111'
	1006: I am '101'
```

挑战！实现一个内存分享型 fork()，叫做 sfork()。这个版本应该让父环境和子环境共享所有它们的除了栈区外的内存页（所以写入一个环境也会出现在另一个里）。另外，一旦你在 C 部分实现了 IPC 之后，使用你的 sfork()来运行 user/pingpongs。你会找到一个新方法去提供全局 thisenv 指针功能。

挑战！你实现的 fork 会非常多的系统调用。在 x86 里，使用中断切换进内核有很大的代价。增加系统调用接口，这样就可以一次发送一批系统调用。然后更改 fork 来使用此接口。

你的新 fork 变得有多快了？

你可以（大概）回答这个问题，通过使用这些参数来估计改进版的 fork 有多少性能上的改进：一个 int 0x30 指令的开销有多大？你的 fork 里会执行多少次 int 0x30 指令？TSS 栈切换昂贵吗？等等……

或者，你可以在真实的硬件上启动你的内核来做一个真正的基准测试你的代码。参见 RDTSC（read time-stamp counter）指令，在 IA32 手册里有定义，它计算上次处理器重置以来经过的时钟周期数。QEMU 不会如实地模拟这个指令（它要么记录虚拟指令的执行数，要么使用主机的 TSC，哪种方法都不会反映所需的真实 CPU 的周期数）。

这是 B 部分的结尾，确保你通过了所有 B 部分的测试（运行 make grade）。和往常一样，通过 make handin 来提交作业。

## Part C: Preemptive Multitasking and Inter-Process communication (IPC)（抢占式多任务和进程间通信）

在实验 4 的最后部分，你会修改内核来抢占不合作的内核，来允许环境去显式地传递消息给对方。

### Clock Interrupts and Preemption（时钟中断和抢占）

运行 user/spin 测试程序。这个测试程序 fork 一个子环境，一旦它获取到了 CPU 的控制权，它就会简单地在一个紧凑的循环里无限循环下去。无论是它的父环境还是内核都无法重新获取 CPU。这个很明显不是一个理想的状态来保护系统免受用户态程序的 bug 或恶意代码，因为任何用户态环境可以让导致整个系统停顿，仅仅只是进入了一个无限的循环并且不交出 CPU。为了允许内核去抢占一个运行的环境，强制性夺回 CPU 的控制权，我们必须扩展 JOS 来让它可以支持时钟的硬件中断。

#### Interrupt discipline（中断规律）

外部中断（比如说设备中断）被叫做 IRQ。有 16 个可能的 IRQ，编号 0 到 15。IRQ 编号到 IDT 条目的映射不是固定的。picirq.c 里的 pic_init 映射 IRQ 0-15 到 IDT 条目 IRQ_OFFSET 到 IRQ_OFFSET+15。

在 inc/trap.h 里，IRQ_OFFSET 被定义为十进制的 32。因此 IDT 条目 32-47 对应于 IRQ 0-15。比如，时钟中断是 IRQ 0。因此，IDT[IRQ_OFFSET+0]（比如说 IDT[32]）包含了内核里用于处理时钟中断处理例程的地址。这个 IRQ_OFFSET 被选择以至于设备中断不会和处理器异常产生重合，不然会产生很明显的困惑。（实际上，早期的 PC 运行了 MS-DOS，这个 IRQ_OFFSET 被高效地设置为 0，这个确实导致了非常多的在处理硬件中断和处理处理器异常之间的困惑！）

在 JOS 里，我们相对于 xv6 Unix 做了一个关键的简化。处于内核中时，外部设备中断永远被禁用（但是就像 xv6 一样，会在用户空间里启用外部设备中断）。外部中断由位于%eflags 寄存器里的 FL_IF 标志位所控制（参见 inc/mmu.h）。当这个标志位被设置时，外部中断将被允许。虽然这个位可以通过几个方式来修改，因为我们简化了，我们处理的方法只是在我们进入（enter）和离开（leave）用户态时保存和恢复%eflags 寄存器。

你必须确保当用户环境运行时，FL_IF 标志被设置，这样当中断到达时，它被传递到处理器，由你的中断代码处理。否则，中断被屏蔽，或被忽略，直到中断被重新启用。我们用引导加载程序的第一个指令屏蔽了中断，到目前为止，我们还没有设法重新启用它们。

练习 13。修改 kern/trapentry.S 和 kern/trap.c 去初始化 IDT 以适当的条目，为 IRQ 0-15 提供处理程序。然后修改 kern/env.c 里的 env_alloc()的代码来确保用户环境总是运行的时候开启了中断。

还要取消 sched_halt()中的 sti 指令的注释，以便把空闲 CPU 的中断取消掩码（开启中断）。

处理器在调用硬件中断处理程序时从不推送错误代码。此时，你可能想要重读[80386 参考手册](https://pdos.csail.mit.edu/6.828/2018/readings/i386/toc.htm)的 9.2 小结，或者[IA-32 Intel Architecture Software Developer's Manual, Volume 3](https://pdos.csail.mit.edu/6.828/2018/readings/ia32/IA32-3A.pdf)的 5.8 小结。

做完这个练习之后，如果你在内核中运行一段时间的测试程序(例如，spin)，你应该看到内核打印硬件中断的陷阱帧（trap frame）。虽然现在处理器中启用了中断，JOS 还没有处理它们，所以你应该看到它把每个中断错误地归结为当前运行的用户环境，然后把用户环境给销毁掉。最终它应该会会把所有的环境都销毁掉，然后掉进 monitor。

### Handling Clock Interrupts（处理时钟中断）

在 user/spin 程序里，当子环境第一次运行，它只是在一个循环里自旋，内核永远不会重新掌握控制权。我们需要让硬件产生定期的时钟中断，这个会强迫把控制权还给内核，这样我们就可以在不同的用户环境之间切换了。

lapic_init 和 pic_init（来自于 init.c 里的 i386_init），我们已经为你写了，设置好时钟和中断控制器来产生中断。你现在需要写处理这些中断的代码。

练习 14。修改内核的 trap_dispatch()函数，使它当一个时钟中断到来时，它调用 sched_yield()来找到并运行一个不同的环境。

你现在应该可以通过 user/spin 测试了：父环境应该 fork 子环境，sys_yield()多次但是每次都会重新获取 CPU 的控制权，然后最终 kill 掉子环境并优雅地终止。

这是做一些回归测试的好时机。确保你没有破坏实验的之前部分，让它开启中断。也尝试使用 make CPUS=2 来运行多个 CPU。你目前应该也可以通过 stresssched 了。运行 make grade 来确保通过。现在可以获得整个实验的 65/80 分了。

### Inter-Process communication (IPC)（进程间通信）

(从技术上讲，在 JOS 中这是“环境间通信”或“IEC”，但其他人都称它为 IPC，所以我们将使用标准术语。)

我们专注于操作系统的隔离层面，它给人一种错觉，每个程序都有一台属于自己的机器。 操作系统的另一个重要服务是允许程序在需要时相互通信。让程序与其他程序交互是非常强大的。Unix 管道模型就是典型的例子。

有许多进程间通信的模型。即使在今天，关于哪种模型是最好的仍然存在争论。我们就不讨论这个了。相反，我们将实现一个简单的 IPC 机制，然后进行试验。

#### IPC in JOS

你要为 JOS 内核实现一些额外的系统调用，它们共同提供一个简单的进程间通信机制。你要实现两个系统调用，sys_ipc_recv 和 sys_ipc_try_send。然后你要实现两个库来包装 ipc_recv 和 ipc_send。

用 JOS 进程间通信机制，允许用户环境互相发送“消息”，这个“消息”包含两个组件：一个简单的 32 位值，以及可选的单页映射。允许环境在消息中传递页面映射提供了一种有效的方式来传输比单个 32 位整数更大的数据，还允许环境轻松设置共享内存。

#### Sending and Receiving Messages（发送与接受消息）

为了接受消息，一个环境调用 sys_ipc_recv。这个系统调用重新对当前环境调度，让它不要运行直到一个消息被接收到。当一个环境正在等待接收消息时，任何其他环境可以给它发送消息-不只是一个特定的环境，也不只是有父子关系的环境可以发送消息给它。换句话说，你在 A 部分中实现的权限检查将不适用于 IPC，因为 IPC 系统调用是精心设计的，保证它很“安全”：一个环境不能仅仅通过向另一个环境发送消息就导致它发生故障(除非目标环境也有错误)。

为了尝试发送一个值，环境需要带上接收环境的 id 和要发送的值去调用 sys_ipc_send。如果指定的环境正在接收（它已经调用了 sys_ipc_recv 但还没有接收到一个值），那么它发送消息并返回 0。否则发送返回 -E_IPC_NOT_RECV 来指示目标环境当前并不期待接收到一个值。

用户空间中的库函数 ipc_recv 将负责调用 sys_ipc_recv，然后在当前环境的结构体 Env 中查找接收到的值的信息。

同样地，库函数 ipc_send 会重复调用 sys_ipc_try_send 直到发送成功。

#### Transferring Pages（传递内存页）

当一个环境调用 sys_ipc_recv 带着一个可用的 dstva 参数（这个值在 UTOP 之下），表明这个环境想要接收到一个页映射。如果发送者发送一个页，那么这个页应该映射到接收者的地址空间的 dstva 处。如果接收者已经有一个页映射在 dstva，那么就把已经映射的这个页取消映射。

当一个环境调用 sys_ipc_try_send 带着一个可用的 srcva 参数（这个值在 UTOP 之下），这表明发送者想要去发送当前映射在 srcva 的页（带着权限位）给接收者。IPC 成功之后，发送者保留它地址空间原始映射在 srcva 的页，但是接收者也获取了对同一个物理页的映射，也就是接收者指定的 dstva （接收者的地址空间）所映射的物理页。最终的结果就是发送者和接收者共享这个页。

如果发送者和接收者其中之一没有指定要传递内存页，那么就没有页会被传递。在任何 IPC 之后，内核都会将接收方的 Env 结构体中的新字段 env_ipc_perm 设置为所接收页的权限，如果没有接收页，则设置为 0。

#### Implementing IPC（实现 IPC）

练习 15。实现 kern/syscall.c 里的 sys_ipc_recv 和 sys_ipc_send。在实现它们之前，先读一下注释，因为它们需要协同工作。当你调用这些例程里的 envid2env 时，你应该设置 checkperm 标志位为 0，意思是任何环境都被允许发送 IPC 消息给任何其他的环境，这样除了验证目标 envid 是否有效外，内核不进行特殊的权限检查。

然后实现 lib/ipc.c 里的 ipc_recv 和 ipc_send 函数。

使用 user/pingpong 和 user/primes 函数去测试你的 IPC 机制。user/primes 会为每个质数生成一个新环境直到 JOS 用完所有的环境。你可能会发现读 user/primes.c 很有趣，可以看看背后的一切的 fork 和 IPC 是怎么弄的。

挑战！为什么 ipc_send 要循环？修改系统调用接口让它不需要这样做。确保你可以这种情况：处理多个环境同时尝试发送到一个环境。

挑战！初始筛选只是在大量并发程序之间传递消息的一种巧妙使用。阅读 C. A. R. Hoare, ``Communicating Sequential Processes,'' Communications of the ACM 21(8) (August 1978), 666-667，然后实现矩阵乘法的例子。

挑战！Doug McIlroy 的幂级数计算器是消息传递功能最令人印象深刻的例子之一，在[M. Douglas McIlroy, ``Squinting at Power Series,'' Software--Practice and Experience, 20(7) (July 1990), 661-683](https://swtch.com/~rsc/thread/squint.pdf)有描述。实现他的幂级数计算器，计算 sin(x+x^3)的幂级数。

挑战！通过应用 Liedtke 论文中的一些技术，使 JOS 的 IPC 机制更加高效，[Improving IPC by Kernel Design](http://dl.acm.org/citation.cfm?id=168633)，或者其他你能想到的技巧。为此，你可以随意修改内核的系统调用 API，只要你的代码是向后兼容我们的评分脚本的就行。

这是部分 C 的结尾。确保你通过了所有的 make grade 测试，不要忘记在 answer-lab4.txt 中写下你的问题答案和你的挑战练习解决方案的描述。

在递交之前，使用 git status 和 git diff 来查看你的修改，不要忘记 git add answer-lab4.txt。当你准备好了，就用 git commit -am 'my solutions to lab 4'来提交你的修改，然后 make handin。
