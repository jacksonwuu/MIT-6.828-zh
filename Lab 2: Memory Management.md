> 原文链接：https://pdos.csail.mit.edu/6.828/2018/labs/lab2/

# Lab 2: Memory Management

## 介绍

在本实验中，你将为你的操作系统编写内存管理代码。内存管理有两个组件。

第一个组件是内核的物理内存分配器，这样内核可以分配内存，然后释放它。你的分配器将以 4096 字节(称为页)为单位进行操作。你的任务将是维护数据结构，这些数据结构记录了哪些物理页面是空闲的，哪些被分配了，以及有多少进程共享每个分配的页面。你还将编写用于分配和释放内存页的例程。

内存管理的第二个组件是虚拟内存，它将内核和用户软件使用的虚拟地址映射到物理内存中的地址。当指令使用内存时，x86 硬件的内存管理单元(MMU)会参考一组页表来执行映射。你将根据我们提供的规范修改 JOS 以设置 MMU 的页表。

### 开始

In this and future labs you will progressively build up your kernel. We will also provide you with some additional source.

在这个以及未来的实验中，你会逐渐地构建你的内核。我们会给你提供一些额外的源代码。

Lab 2 包括了如下新的源代码文件，你应该浏览一遍：

-   inc/memlayout.h
-   kern/pmap.c
-   kern/pmap.h
-   kern/kclock.h
-   kern/kclock.c

memlayout.h 描述了虚拟地址空间的布局，你要通过修改 pmap.c 来实现的这个布局。memlayout.h 和 pmap.h 定义了 PageInfo 结构体，你将使用它来跟踪哪些物理内存页面是空闲的。kclock.c 和 kclock.h 操作 PC 的时钟（里面有电池）和 CMOS RAM 硬件，在这些硬件中，BIOS 记录了 PC 所包含的物理内存的数量以及其他内容。pmap.c 中的代码需要读取这个设备硬件，以便弄清楚有多少物理内存，但是这部分代码已经为你完成了：你不需要知道 CMOS 硬件如何工作的细节。

要特别注意 memlayout.h 和 mmap.h，因为本实验要求你使用和理解它们所包含的许多定义。你可能还想回顾一下 inc/mmu.h，因为它还包含了一些对本实验室有用的定义。

### 实验要求

## Part 1: 物理内存页管理

操作系统必须跟踪物理 RAM 的哪些部分是空闲的，哪些部分目前正在使用。JOS 使用页面粒度管理 PC 的物理内存，以便使用 MMU 映射和保护分配的每个内存块。

现在，你将编写物理页面分配器。它通过一个 PageInfo 结构体对象的链表来跟踪哪些页面是空闲的(与 xv6 不同的是，这些对象并没有嵌入到空闲页面本身中)，每个结构体对应一个物理页面。在编写虚拟内存实现的其余部分之前，你需要编写物理页分配器，因为页表管理代码将需要分配用于存储页表的物理内存。

练习 1：在文件 kern/mmap.c 中，实现以下函数的代码(可能按照给定的顺序)。

```
boot_alloc()
mem_init() (only up to the call to check_page_free_list(1))
page_init()
page_alloc()
page_free()
```

check_page_free_list() 和 check_page_alloc() 用来测试你的物理内存页分配器。你应该启动 JOS 来看看是否 check_page_alloc()现实成功。调整你的代码直到通过测试。你也可以添加自己的 assert()来检查你的假设是正确的。

这个实验，以及 6.828 的所有实验，将要求你做一些调查工作，以明确你需要做什么。这个任务并没有描述必须添加到 JOS 的代码的所有细节。在 JOS 源代码中需要修改的部分查找注释；这些注释通常包含了规范和提示。你还需要查看 JOS 的相关部分、Intel 手册，可能还需要查看 6.004 或 6.033 的笔记。

## Part 2: 虚拟内存

在做任何事情之前，先熟悉一下 x86 的保护模式的内存管理架构：分段和分页机制。

练习 2：如果你还没有阅读[Intel 80386 参考手册](https://pdos.csail.mit.edu/6.828/2018/readings/i386/toc.htm)的第 5 章和第 6 章。仔细阅读关于分页转换和基于分页保护的章节(5.2 和 6.4)。我们建议你也浏览一下关于分段的部分；虽然 JOS 使用分页硬件来进行虚拟内存和保护，但在 x86 上不能禁用段转换和基于段的保护，所以你需要对它有基本的了解。

### 虚拟地址、线性地址和物理地址

在 x86 术语中，虚拟地址由段选择器和段内的偏移量组成。线性地址是在段转换之后，页面转换之前得到的地址。一个物理地址是你在段和页转换之后最终得到的，它最终会通过硬件总线发送到你的 RAM。

```

           Selector  +--------------+         +-----------+
          ---------->|              |         |           |
                     | Segmentation |         |  Paging   |
Software             |              |-------->|           |---------->  RAM
            Offset   |  Mechanism   |         | Mechanism |
          ---------->|              |         |           |
                     +--------------+         +-----------+
            Virtual                   Linear                Physical

```

C 指针是虚地址的“偏移量”组件。在 boot/boot.S，我们建立了一个全局描述表(Global Descriptor Table， GDT)，通过将所有段的基址设置为 0 并限制至 0xffffffff，有效地禁用了段转换。因此，“选择子”没有作用，线性地址总是等于虚拟地址的偏移量。在实验 3 中，我们将不得不与分段机制进行更多的交互，以设置特权级别，但是对于内存转换，我们可以忽略整个 JOS 实验室中的分段机制，只关注分页机制。

回想一下，在实验 1 的第 3 部分，我们创建了一个简单的页表，以便内核可以在它的链接地址 0xf0100000 处运行，尽管它实际上是在物理内存中加载的，就在 ROM BIOS 上面的 0x00100000 处。这个页表只映射了 4MB 的内存。在这个实验室中，你将为 JOS 设置虚拟地址空间布局，我们将对其进行扩展，以映射 0xf0000000 开始的虚拟地址到物理内存的前 256MB，并映射虚拟地址空间的许多其他区域。

练习 3：虽然 GDB 只能通过虚拟地址访问 QEMU 的内存，但在设置虚拟内存时能够查看物理内存，这通常就很有用。查看实验室工具指南中的 QEMU [监视器命令](https://pdos.csail.mit.edu/6.828/2018/labguide.html#qemu)，特别是 xp 命令，它允许你检查物理内存。要访问 QEMU 监视器，请在终端中按 `Ctrl-a c`。

使用 QEMU 监视器中的 xp 命令和 GDB 中的 x 命令检查相应的物理和虚拟地址的内存，并确保你看到了相同的数据。

我们的 QEMU 修补版提供了一个 info pg 命令，它可能也很有用：它显示当前页表的紧凑但详细的表示，包括所有映射的内存范围、权限和标志。Stock QEMU 还提供了一个 info mem 命令，它显示映射的虚拟地址范围以及具有哪些权限的概述。

从在 CPU 上执行的代码开始，一旦我们进入了保护模式(我们首先进入 boot/boot.s)，就无法直接使用线性或物理地址。所有内存引用都被解释为虚拟地址并被 MMU 转换，这意味着 C 中的所有指针都是虚拟地址。

JOS 内核经常需要将地址操作为不透明的值或整数，而不取值，例如在物理内存分配器中。这些地址有时是虚拟地址，有时是物理地址。为了写代码的文档，JOS 源代码区分了这两种情况：类型 uintptr_t 表示不透明的虚拟地址，而 physaddr_t 表示物理地址。这两种类型实际上只是 32 位整数的同义词(uint32_t)，所以编译器不会阻止你从一个类型分配到另一个！因为它们是整型(不是指针)，如果你试图对它们进行取值，编译器会报错。

JOS 内核可以通过将 uintptr_t 转换为指针类型来以此来取值。相反，内核不能合理对一个物理地址取值，因为 MMU 会转换所有的内存引用。如果将 physaddr_t 转换为指针并对其取值，你可能能够加载并存储到该地址(硬件将其解释为一个虚拟地址)，但你可能无法获得你真正想要的内存位置。

总结一下：

```
C type          Address type
T*  	        Virtual
uintptr_t  	    Virtual
physaddr_t      Physical
```

问题：假设下面的 JOS 内核代码是正确的，变量 x 应该是什么类型，uintptr_t 或 physaddr_t？

```
mystery_t x;
char* value = return_a_pointer();
*value = 10;
x = (mystery_t) value;
```

JOS 内核有时需要读取或修改它只知道物理地址的内存。例如，向页表添加映射可能需要分配物理内存来存储页目录，然后初始化该内存。然而，内核不能绕过虚拟地址转换，因此不能直接加载和存储物理地址。JOS 从虚拟地址 0xf0000000 处的物理地址 0 开始重新映射所有物理内存的原因之一是帮助内核读写它只知道物理地址的内存。为了将物理地址转换为内核可以实际读写的虚拟地址，内核必须在物理地址中添加 0xf0000000，以便在映射区域中找到其对应的虚拟地址。你应该使用`KADDR(pa)`来获取虚拟地址。

JOS 内核有时还需要能够根据存储内核数据结构的内存的虚拟地址找到物理地址。由 boot_alloc()分配的内核全局变量和内存位于内核被加载的区域，也就是从 0xf0000000 开始的区域，也就是我们映射所有物理内存的区域。因此，要将该区域中的虚拟地址转换为物理地址，内核可以简单地减去 0xf0000000。你应该用`PADDR(va)`来做这个减法，获取物理地址。

### 引用计数

在之后的实验室中，你通常会将相同的物理页面同时映射到多个虚拟地址(或多个环境的地址空间)。你将在与物理页对应的结构 PageInfo 的 pp_ref 字段中保持对每个物理页的引用数量的计数。当物理页的这个计数为 0 时，该页可以被释放，因为它不再被使用。通常，这个计数应该等于物理页面在所有页表中出现在 UTOP 下面的次数(UTOP 上面的映射大部分是在内核启动时设置的，不应该被释放，所以不需要引用计数)。我们还将使用它来跟踪指向页目录页的指针的数量，以及页目录对页表页的引用数量。

使用 page_alloc 时要小心。它返回的页面的引用计数总是 0，所以 pp_ref 应该在你对返回的页面做了一些操作(比如将其插入到页表中)之后递增。有时这是由其他函数(例如，page_insert)处理的，有时调用 page_alloc 的函数必须直接执行。

### 页表管理

现在，你将编写一组例程来管理页表：插入和删除线性地址到物理地址的映射，以及在需要时创建页表页。

练习 4：在 kern/pmap.c 文件里，你必须实现以下函数。

    pgdir_walk()
    boot_map_region()
    page_lookup()
    page_remove()
    page_insert()

从 mem_init()调用 check_page()测试你的页表管理例程。在继续之前，你应该确保它报告成功。

## Part 3: 内核地址空间

JOS 会将处理器的 32 位线性地址空间分为两部分。我们将在 lab 3 中开始加载和运行用户环境(进程)，它将控制下面部分的布局和内容，而内核始终保持对上面部分的完全控制。在 inc/memlayout.h 中，分隔线由符号 ULIM 任意定义，为内核保留了大约 256MB 的虚拟地址空间。这解释了为什么我们需要在 lab 1 中给内核一个如此高的链接地址：否则内核的虚拟地址空间将没有足够的空间来同时映射到下面的用户环境中。

你会发现参考 inc/memlayout.h 中的 JOS 内存布局图对这部分和以后的实验都很有帮助。

### 权限和故障隔离

由于内核和用户内存都存在于每个环境的地址空间中，所以我们必须在 x86 页表中使用权限位来允许用户代码只访问地址空间的用户部分。否则，用户代码中的 bug 可能会覆盖内核数据，导致崩溃或更微妙的故障；用户代码也可以窃取其他环境的私有数据。请注意，可写权限位(PTE_W)同时影响用户和内核代码！

用户环境对 ULIM 以上的任何内存都没有权限，而内核将能够读写这些内存。对于地址范围[UTOP，ULIM)，内核和用户环境都有相同的权限：它们可以读但不能写这个地址范围。这个地址范围用于向用户环境以只读方式公开某些内核数据结构。最后，UTOP 下面的地址空间是供用户环境使用的；用户环境将设置访问该内存的权限。

### 初始化内核地址空间

现在你可以设置 UTOP 之上的地址空间了：地址空间的内核部分。inc/memlayout.h 展示你应该使用的内存布局。你会使用

练习 5：填充 mem_init()里在调用里 check_page()之后丢失的代码。

你的代码应该通过 check_kern_pgdir()和 check_page_installed_pgdir()这两个函数的检查。

问题

2、此时，页面目录中的哪些条目(行)已经被创建？它们映射了什么地址，指向什么地方？换句话说，尽可能多地填写这个表格:

| Entry | Base Virtual Address | Points to (logically)                 |
| ----- | -------------------- | ------------------------------------- |
| 1023  | ？                   | Page table for top 4MB of phys memory |
| 1022  | ？                   | ？                                    |
| .     | ？                   | ？                                    |
| .     | ？                   | ？                                    |
| .     | ？                   | ？                                    |
| 2     | 0x00800000           | ？                                    |
| 1     | 0x00400000           | ？                                    |
| 0     | 0x00000000           | [see next question]                   |

3、我们把内核和用户环境放在同一个地址空间。为什么用户程序不能读写内核内存？哪些特定的机制保护内核内存？

4、这个操作系统能支持的最大物理内存是多少？为什么？

5、如果我们有最大的物理内存，那么管理内存需要多少空间开销？这个开销是怎么分解的？

6、重新在 kern/entry.S 中设置页表。年代和 kern/entrypgdir.c。打开分页后，EIP 仍然是一个较低的数字(略高于 1MB)。在什么时候我们可以过渡到运行在 KERNBASE 之上的 EIP？是什么使我们能够在启用分页和开始在 KERNBASE 之上的 EIP 上运行之间继续以较低的 EIP 执行呢？为什么这种转变是必要的？

挑战！我们消耗了很多物理页来保存 KERNBASE 映射的页表。在页目录条目中使用 PTE_PS(“Page Size”)位来节省空间。这个位在最初的 80386 中不受支持，但是在最新的 x86 处理器中得到了支持。因此，你必须参考[Volume 3 of the current Intel manuals](https://pdos.csail.mit.edu/6.828/2018/readings/ia32/IA32-3A.pdf)。确保你设计的内核只在支持这种优化的处理器上使用这种优化!

挑战！Extend the JOS kernel monitor with commands to:

-   Display in a useful and easy-to-read format all of the physical page mappings (or lack thereof) that apply to a particular range of virtual/linear addresses in the currently active address space. For example， you might enter 'showmappings 0x3000 0x5000' to display the physical page mappings and corresponding permission bits that apply to the pages at virtual addresses 0x3000， 0x4000， and 0x5000.
-   Explicitly set， clear， or change the permissions of any mapping in the current address space.
-   Dump the contents of a range of memory given either a virtual or physical address range. Be sure the dump code behaves correctly when the range extends across page boundaries!
-   Do anything else that you think might be useful later for debugging the kernel. (There's a good chance it will be!)

### 地址空间布局替换方案

我们在 JOS 中使用的地址空间布局不是唯一的。一个操作系统可能会将内核映射为低线性地址，而将线性地址空间的上部留给用户进程。x86 内核一般不采取这种方法，然而，因为 x86 的向后兼容模式，称为虚拟 8086 模式，“硬接线”使用处理器底部的线性地址空间，因此如果内核映射到这里就不可用。

尽管更加困难，但是这是有可能的，设计内核使其不必为自己保留任何固定的处理器线性或虚拟地址空间，而是有效地允许用户级进程不受限制地使用整个 4GB 的虚拟地址空间——同时仍然完全保护内核不受这些进程的影响，并保护不同的进程不受其他进程的影响!

Challenge! Each user-level environment maps the kernel. Change JOS so that the kernel has its own page table and so that a user-level environment runs with a minimal number of kernel pages mapped. That is， each user-level environment maps just enough pages mapped so that the user-level environment can enter and leave the kernel correctly. You also have to come up with a plan for the kernel to read/write arguments to system calls.

Challenge! Write up an outline of how a kernel could be designed to allow user environments unrestricted use of the full 4GB virtual and linear address space. Hint: do the previous challenge exercise first， which reduces the kernel to a few mappings in a user environment. Hint: the technique is sometimes known as "follow the bouncing kernel." In your design， be sure to address exactly what has to happen when the processor transitions between kernel and user modes， and how the kernel would accomplish such transitions. Also describe how the kernel would access physical memory and I/O devices in this scheme， and how the kernel would access a user environment's virtual address space during system calls and the like. Finally， think about and describe the advantages and disadvantages of such a scheme in terms of flexibility， performance， kernel complexity， and other factors you can think of.

Challenge! Since our JOS kernel's memory management system only allocates and frees memory on page granularity， we do not have anything comparable to a general-purpose malloc/free facility that we can use within the kernel. This could be a problem if we want to support certain types of I/O devices that require physically contiguous buffers larger than 4KB in size， or if we want user-level environments， and not just the kernel， to be able to allocate and map 4MB superpages for maximum processor efficiency. (See the earlier challenge problem about PTE_PS.)

Generalize the kernel's memory allocation system to support pages of a variety of power-of-two allocation unit sizes from 4KB up to some reasonable maximum of your choice. Be sure you have some way to divide larger allocation units into smaller ones on demand， and to coalesce multiple small allocation units back into larger units when possible. Think about the issues that might arise in such a system.
