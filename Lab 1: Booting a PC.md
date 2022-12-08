# Lab 1: Booting a PC

## 介绍

这个实验室分成三个部分。第一部分主要介绍如何熟悉 x86 汇编语言、QEMU x86 仿真器和 PC 的开机引导程序。第二部分检查 6.828 内核的引导加载程序，它位于实验 boot 目录中。最后，第三部分研究了 6.828 内核本身的初始模板，名为 JOS，它位于 kernel 目录中。

## Part 1: PC Bootstrap

第一个练习的目的是向你介绍 x86 汇编语言和 PC 引导过程，并让你开始 QEMU 和 QEMU/GDB 调试。你不需要为实验室的这一部分编写任何代码，但你应该根据自己的理解仔细阅读它，并准备好回答下面提出的问题。

### 从 x86 汇编开始

如果你没有太熟悉 x86 汇编语言，你会在这个课程里很快对它熟悉！[PC Assembly Language Book](https://pdos.csail.mit.edu/6.828/2018/readings/pcasm-book.pdf)是个绝佳的开始点。这个书包含了新旧材料。

警告：非常不幸的是书里的例子都是用 NASM 汇编器，但是我们会用 GNU 汇编器。NASM 使用所谓的 Intel 语法，但是 GNU 使用 AT&T 语法。然而语法上都是等价的，用不同语法写的汇编从表面上看会非常不一样。幸运的是这两种语法非常简单，这本书有讲到：[Brennan's Guide to Inline Assembly](http://www.delorie.com/djgpp/doc/brennan/brennan_att_inline_djgpp.html)。

练习 1：熟悉[the 6.828 reference page](https://pdos.csail.mit.edu/6.828/2018/reference.html)里的汇编语言材料。你现在不需要都读，但是在读写 x86 汇编程序集时，你几乎肯定会想要参考其中一些资料。

我们确实推荐阅读[Brennan's Guide to Inline Assembly](http://www.delorie.com/djgpp/doc/brennan/brennan_att_inline_djgpp.html)的"The Syntax"部分。它很好地描述了 AT&T 汇编语法，我们会在 JOS 里用到的。

当然，x86 汇编语言编程的权威参考是 Intel 的指令集架构参考，你可以在[the 6.828 reference page](https://pdos.csail.mit.edu/6.828/2018/reference.html)中找到两种风格：旧的[80386 Programmer's Reference Manual](https://pdos.csail.mit.edu/6.828/2018/readings/i386/toc.htm)的 HTML 版本，比最近的手册更短，更容易浏览，但描述了我们将在 6.828 中使用的所有 x86 处理器特性；和完整的、最新的和最好的[IA-32 Intel Architecture Software Developer's Manuals](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)从英特尔，涵盖所有最新的处理器的特点，我们不需要在课堂上使用，但你可能会对感兴趣学习。同样的(通常更友好的)一套手册[available from AMD](http://developer.amd.com/resources/developer-guides-manuals/)。将 Intel/AMD 架构手册保存起来，以备以后使用，或者当你想要查找特定处理器特性或指令的明确解释时，将其作为参考。

### 模拟 x86

我们不是在真实的、实际的个人计算机(PC)上开发操作系统，而是使用一个程序来忠实地模拟一个完整的 PC：你为模拟器编写的代码也将在真实的 PC 上启动。使用模拟器简化了调试；例如，你可以在模拟的 x86 中设置断点，这在 x86 的硅版中很难做到。

在 6.828 中，我们将使用[QEMU 模拟器](http://www.qemu.org/)，这是一个现代且相对快速的模拟器。虽然 QEMU 的内置监视器只提供有限的调试支持，但 QEMU 可以充当[GNU 调试器(GDB)](http://www.gnu.org/software/gdb/)的远程调试目标，我们将在本实验中使用它来逐步完成早期引导过程。

首先，将 Lab 1 文件解压缩到 Athena 上你自己的目录中，如“软件安装”中所述，然后在 Lab 目录中输入 make(或 BSD 系统上的 gmake)，以构建最小的 6.828 引导加载程序和内核。(把我们在这里运行的代码称为“内核”有点慷慨，但我们将在整个学期充实它。)

```
athena% cd lab
athena% make
+ as kern/entry.S
+ cc kern/entrypgdir.c
+ cc kern/init.c
+ cc kern/console.c
+ cc kern/monitor.c
+ cc kern/printf.c
+ cc kern/kdebug.c
+ cc lib/printfmt.c
+ cc lib/readline.c
+ cc lib/string.c
+ ld obj/kern/kernel
+ as boot/boot.S
+ cc -Os boot/main.c
+ ld boot/boot
boot block is 380 bytes (max 510)
+ mk obj/kern/kernel.img
```

(如果你得到像“undefined reference to '\_\_udivdi3'”这样的错误，你可能没有 32 位的 gcc multilib。如果你正在 Debian 或 Ubuntu 上运行，尝试安装 gcc-multilib 包。)

现在你已经准备好运行 QEMU，并提供文件 obj/kern/kernel.img，创建在上面，作为模拟 PC 的“虚拟硬盘”的内容。这个硬盘映像包含我们的引导加载程序(obj/boot/boot)和我们的内核(obj/kernel)。

```
athena% make qemu
or
athena% make qemu-nox
```

这将执行 QEMU，其中包含设置硬盘和直接串口输出到终端所需的选项。一些文本应该出现在 QEMU 窗口：

```
Booting from Hard Disk...
6828 decimal is XXX octal!
entering test_backtrace 5
entering test_backtrace 4
entering test_backtrace 3
entering test_backtrace 2
entering test_backtrace 1
entering test_backtrace 0
leaving test_backtrace 0
leaving test_backtrace 1
leaving test_backtrace 2
leaving test_backtrace 3
leaving test_backtrace 4
leaving test_backtrace 5
Welcome to the JOS kernel monitor!
Type 'help' for a list of commands.
K>
```

‘Booting from Hard Disk...’之后的一切都是我们的 JOS 内核打印的；K> 是由我们在内核中包含的小监视器或交互式控制程序打印的提示符。如果执行 make qemu，内核打印的这些行将出现在运行 qemu 的常规 shell 窗口和 qemu 显示窗口中。这是因为测试和打分的目的，我们设立了 JOS 内核写它的控制台输出不仅在虚拟 VGA 显示(见 QEMU 窗口)，而且模拟电脑的虚拟串口，QEMU 反过来输出自己的标准输出。类似地，JOS 内核将从键盘和串口接受输入，因此你可以在 VGA 显示窗口或运行 QEMU 的终端中向它提供命令。或者，你可以通过运行 make qemu-nox 在没有虚拟 VGA 的情况下使用串行控制台。这可能是方便的，如果你是 SSH 进入一个 Athena 拨号。要退出 qemu，输入 Ctrl+a x。

内核监视器只有两个命令可以用，help 和 kerninfo。

```
K> help
help - display this list of commands
kerninfo - display information about the kernel
K> kerninfo
Special kernel symbols:
  entry  f010000c (virt)  0010000c (phys)
  etext  f0101a75 (virt)  00101a75 (phys)
  edata  f0112300 (virt)  00112300 (phys)
  end    f0112960 (virt)  00112960 (phys)
Kernel executable memory footprint: 75KB
K>
```

help 命令很明显，我们将很快讨论 kerninfo 命令输出的含义。虽然很简单，但需要注意的是，这个内核监控器“直接”运行在模拟 PC 的“原始(虚拟)硬件”上。这意味着你应该能够复制 obj/kern/kernel 的内容。将硬盘插入真正的 PC，打开它，并在 PC 的真实屏幕上看到与你在上面的 QEMU 窗口中所做的完全相同的事情。(但是，我们不建议你在硬盘上有有用信息的真正机器上这样做，因为复制 kernel.img 添加到其硬盘的开始部分将回收主引导记录和第一个分区的开始部分，从而有效地导致硬盘上之前的所有内容丢失！)

### PC 的物理地址空间

现在我们将更详细地介绍一下 PC 是如何启动的。PC 机的物理地址空间是硬连接的，有以下总体布局：

```

+------------------+  <- 0xFFFFFFFF (4GB)
|      32-bit      |
|  memory mapped   |
|     devices      |
|                  |
/\/\/\/\/\/\/\/\/\/\

/\/\/\/\/\/\/\/\/\/\
|                  |
|      Unused      |
|                  |
+------------------+  <- depends on amount of RAM
|                  |
|                  |
| Extended Memory  |
|                  |
|                  |
+------------------+  <- 0x00100000 (1MB)
|     BIOS ROM     |
+------------------+  <- 0x000F0000 (960KB)
|  16-bit devices, |
|  expansion ROMs  |
+------------------+  <- 0x000C0000 (768KB)
|   VGA Display    |
+------------------+  <- 0x000A0000 (640KB)
|                  |
|    Low Memory    |
|                  |
+------------------+  <- 0x00000000
```

第一代 PC 基于 16 位 Intel 8088 处理器，只能寻址 1MB 的物理内存。因此，早期 PC 的物理地址空间将从 0x00000000 开始，到 0x000FFFFF 结束，而不是 0xFFFFFFFF。标记为“低内存”的 640KB 区域是早期 PC 能够使用的唯一随机访问内存(RAM)；事实上，最早的电脑只能配置 16KB、32KB 或 64KB 的内存!

从 0x000A0000 到 0x000FFFFF 的 384KB 区域由硬件预留，用于特殊用途，如视频显示缓冲区和非易失性内存中的固件。这个保留区域中最重要的部分是 Basic Input/Output System (BIOS)，它占用了从 0x000F0000 到 0x000FFFFF 的 64KB 区域。在早期的 pc 中，BIOS 保存在真正的只读存储器(ROM)中，但现在的 PC 将 BIOS 存储在可更新的闪存中。BIOS 主要负责对系统进行基本的初始化操作，如激活显卡、检查内存总量等。执行这个初始化之后，BIOS 从一些适当的位置(如软盘、硬盘、CD-ROM 或网络)加载操作系统，并将机器的控制权传递给操作系统。

当 Intel 最终用 80286 和 80386 处理器“突破了 1MB 的障碍”，它们分别支持 16MB 和 4GB 的物理地址空间，PC 架构师仍然保留了原始的 1MB 物理地址空间布局，以确保与现有软件的向后兼容性。因此，现代 PC 在物理内存中有一个从 0x000A0000 到 0x00100000 的“洞”，将 RAM 划分为“低内存”或“常规内存”(前 640KB)和“扩展内存”(其他一切)。此外，PC 的 32 位物理地址空间(尤其是物理 RAM)顶部的一些空间现在通常由 BIOS 保留，供 32 位 PCI 设备使用。

最近的 x86 处理器可以支持超过 4GB 的物理 RAM，所以 RAM 可以扩展到 0xFFFFFFFF 以上。在这种情况下，BIOS 必须安排在系统 RAM 的 32 位可寻址区域的顶部留下第二个洞，为这些 32 位设备的映射留下空间。由于设计限制，JOS 将只使用 PC 的物理内存的前 256MB，所以现在我们假设所有 PC 都“只有”一个 32 位的物理地址空间。但是处理复杂的物理地址空间和硬件组织的其他方面是操作系统开发的一个重要的实际挑战。

### The ROM BIOS

在本部分中，你将使用 QEMU 的调试工具来研究 IA-32 兼容的计算机是如何引导的。

打开两个终端窗口并将两个 shell cd 到你的实验室目录中。其中，输入 make qemu-gdb(或 make qemu-nox-gdb)。这将启动 QEMU，但是 QEMU 在处理器执行第一个指令之前停止，并等待来自 GDB 的调试连接。在第二个终端中，在运行 make 的同一个目录运行 make gdb。你应该看到这样的东西，

```
athena% make gdb
GNU gdb (GDB) 6.8-debian
Copyright (C) 2008 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "i486-linux-gnu".
+ target remote localhost:26000
The target architecture is assumed to be i8086
[f000:fff0] 0xffff0:	ljmp   $0xf000,$0xe05b
0x0000fff0 in ?? ()
+ symbol-file obj/kern/kernel
(gdb)
```

我们提供了一个.gdbinit 文件，该文件设置 GDB 来调试早期引导期间使用的 16 位代码，并将其附加到侦听的 QEMU 上。(如果它不能工作，你可能必须在你的主目录的.gdbinit 中添加一个 add-auto-load-safe-path，来让 GDB 处理我们提供的.gdbinit。GDB 会告诉你是否必须这样做。)

以下行:

```
[f000:fff0] 0xffff0:	ljmp   $0xf000,$0xe05b
```

是 GDB 反汇编第一行执行的指令。从输出端你可以得到一些结论：

-   IBM PC 在物理地址 0x000ffff0 处开始执行，也就是位 BIOS ROM 预留的 64KB 的最顶端。
-   PC 开始执行时 CS = 0xf000，IP = 0xfff0。
-   第一个执行的指令是 jmp 指令，它会跳转到 CS = 0xf000 和 IP = 0xe05b 处。

QEMU 为什么会这样开始呢？英特尔就是这样设计 8088 处理器的，IBM 在他们最初的个人电脑中使用了这种处理器。因为在 PC BIOS 被“硬接线”到物理地址范围 0x000f0000-0x000fffff 处，这种设计可以确保机器的 BIOS 总是在开启或系统重启时第一个获得控制权——这是至关重要的，因为在 RAM 里没有软件可以执行。QEMU 模拟器自己自带的 BIOS，它将 BIOS 放置在处理器模拟物理地址空间的这个位置。处理器复位时，(模拟)处理器进入实模式，并将 CS 设置为 0xf000, IP 设置为 0xfff0，因此执行从那个(CS:IP)段地址开始。分段地址 0xf000:fff0 如何变成物理地址？

为了回答这个问题，我们需要了解一些关于实际模式寻址的知识。在实模式下(PC 启动的模式)，地址转换按如下公式进行：物理地址= 16\*段+偏移量。因此，当 PC 将 CS 设置为 0xf000, IP 设置为 0xfff0 时，所引用的物理地址为:

```
   16 * 0xf000 + 0xfff0   # in hex multiplication by 16 is
   = 0xf0000 + 0xfff0     # easy--just append a 0.
   = 0xffff0
```

0xffff0 是 BIOS 结尾（0x100000）之前的 16 字节。因此不要惊讶做的第一件 BIOS 做的事情是 jmp 跳转回 BIOS 更早之前的位置；毕竟，16 个字节能完成多少任务?

练习 2。使用 GDB 的 si（Step Instruction）命令来追踪 ROM BIOS 更多的指令，然后试着猜猜它在做什么。你可能想要看一下[Phil Storrs I/O Ports Description](http://web.archive.org/web/20040404164813/members.iweb.net.au/~pstorr/pcbook/book2/book2.htm)，以及其他材料[6.828 reference materials page](https://pdos.csail.mit.edu/6.828/2018/reference.html)。不需要弄清楚所有的细节——只要先弄懂 BIOS 的大概运作就行了。

当 BIOS 运行时，它设置一个中断描述符表并初始化各种设备，例如 VGA 显示器。这就是你在 QEMU 窗口看到的 "Starting SeaBIOS" 消息来自的地方。

在 BIOS 初始化它所知道的 PCI bus 和所有重要的设备之后，它搜索一个可启动设备，比如软盘、硬盘或者 CD-ROM。最终，当它找到一个可启动的磁盘时，BIOS 读取磁盘里的 boot loader，然后把控制权转交给它。

## Part 2: The Boot Loader

软盘和硬盘被分成很多 512 比特的区域，这些区域被叫做扇区。一个扇区是磁盘最小的传输粒度：每次读写操作必须是一个或多个扇区的大小，并在扇区边界对齐。如果磁盘是可启动的，第一个扇区被叫做启动扇区，因为这是 boot loader 代码所处的地方。当 BIOS 找到一个可启动的软盘或硬盘时，它加载 512 比特的 boot 扇区到物理地址 0x7c00-0x7dff 处，然后使用一个 jmp 指令设置 CS:IP 为 0000:7c00，传递控制权给 boot loader。就像 BIOS 加载地址一样，这些地址是相当随意的，但对于 PC 来说是固定和标准化的。(译者注：因为历史原因，BIOS 的加载地址被约定为 0x7c00，不用想为什么是这个地址，没有什么直接的原因。)

从 CD-ROM 启动的能力是在 PC 进化过程中很久以后才出现的，因此，PC 架构师利用这个机会稍微重新思考了一下引导过程。因此，现代 BIOS 从 CD-ROM 启动的方式有点复杂(也更强大)。CD-ROM 使用一个 2048 比特的扇区大小而不是 512 比特，BIOS 可以加载一个更大的来自磁盘的 boot 镜像进入内存（不仅只是一个扇区），然后在转交控制权。更多信息可以看["El Torito" Bootable CD-ROM Format Specification](https://pdos.csail.mit.edu/6.828/2018/readings/boot-cdrom.pdf)。

然而，对于 6.828 来说，我们会使用传统的硬盘 boot 机制，也就意味着我们的 boot loader 必须填入一个 512 比特的磁盘区域。这个 boot loader 包含了一个汇编语言的源文件，boot/boot.S 和一个 C 源文件，boot/main.c，细心地把这些源文件通读一遍，确保你理解了运作原理。boot loader 必须执行两个主要的功能：

1. 首先，这个 boot loader 把处理器从实模式切换到 32 位保护模式，因为在保护模式下，软件才可以访问处理器物理地址空间的所有 1MB 以上的内存。[PC Assembly Language](https://pdos.csail.mit.edu/6.828/2018/readings/pcasm-book.pdf)的 1.2.7 和 1.2.8 小结对保护模式作了简单的描述，详细的细节可以参考 Intel 架构手册。此刻你只需要理解分段地址在保护模式中转化为物理地址的方式不一样，转换的 offset 是 32 位而不是 16 位的。
2. 第二，这个 boot loader 从磁盘上读取内核，通过使用 x86 特殊的 I/O 指令来直接访问 IDE 磁盘寄存器来做到这一点。如果你想要更好地理解特别的 I/O 指令的含义，查看[the 6.828 reference page](https://pdos.csail.mit.edu/6.828/2018/reference.html)的"IDE hard drive controller"小结。你在这门课程里不必学习太多特殊设备的编程：写设备驱动程序在操作系统开发里是非常重要的部分，但是从概念和架构的角度来看，这也是最无聊的部分。

在你理解了 boot loader 的源代码之后，查看 obj/boot/boot.asm 文件。这个文件是我们的 GUNMakefile 创建 boot loader 之后对其进行反汇编。这个反汇编文件可以很容易地看到这个 boot loader 的代码会驻留在物理内存的哪些确切位置，然后就很容易在 GDB 里追踪 boot loader 发生了什么事情。同样的，obj/kern/kernel.asm 包含了一个对 JOS 内核的反汇编，对于 debug 来说很有用。

你可以在 GDB 里用 b 命令来设置地址断点。例如，b \*0x7c00 在地址 0x7c00 处设置了一个断点。一旦到某个断点，你可以进用 c 和 si 命令继续执行：c 让 QEMU 继续执行直到下一个断点（或者直到你按下 Ctrl-C），si N 可以一次性步进 N 条指令。

为了查看内存里的指令（除了即将执行的下一个，GDB 会自动打印），你可以使用 x/i 指令。这个命令的语法是 x/Ni ADDR，N 是连续反汇编指令的数目，ADDR 是开始反汇编的内存地址。

练习 3。看一眼[lab tools guide](https://pdos.csail.mit.edu/6.828/2018/labguide.html)，尤其是 GDB 命令部分。哪怕你熟悉 GDB，这包括一些对于操作系统工作很有用的深奥的 GDB 命令。

在地址 0x7c00 处设置一个断点，也就是 boot 扇区会被加载到的地方。继续执行直到那个断点。追踪整个 boot/boot.S 的代码，用反汇编文件 obj/boot/boot.asm 里的源代码来追踪你所处的代码位置。再用 GDB 的 x/i 指令来反汇编 boot laoder，然后对比一下原始的 boot loader 源代码和反汇编文件里的代码、以及 GDB 产生的反汇编代码。

追踪 boot/main.c 的 bootmain()，然后再追踪 readsect()。确定与 readsect()中的每个语句对应的确切的程序集指令。追踪 readsect()剩余的代码，然后回到 bootmain()，并标识从磁盘读取内核剩余扇区的 for 循环的开始和结束。找到什么代码会在 loop 循环结束的时候运行，在那里设置断点，然后继续到那个断点。然后步进查看 boot loader 的所有剩余代码。

你要能回答以下的问题：

-   在什么时候处理器开始执行 32 位代码？是什么导致了 16 位到 32 位到模式的转变？
-   boot loader 最后执行的指令是什么，kernel 刚加载时执行的第一个指令是什么？
-   kernel 的第一个指令在哪儿？
-   boot loader 如何决定要读取多少扇区才能加载整个内核？它是在哪里找到这个信息的？

### Loading the Kernel（加载内核）

我们现在将进一步详细研究 boot loader 的 C 语言部分，boot/main.c。在开始之前，最好可以先好好回顾一下 C 编程的基础。

练习 4。阅读 C 语言里的指针部分。C 语言最好的材料就是 The C Programming Language by Brian Kernighan and Dennis Ritchie（也被叫做 K&R）。我们推荐学生买这本书(here is an [Amazon Link](http://www.amazon.com/C-Programming-Language-2nd/dp/0131103628/sr=8-1/qid=1157812738/ref=pd_bbs_1/104-1502762-1803102?ie=UTF8&s=books))，或者[MIT's 7 copies](http://library.mit.edu/F/AI9Y4SJ2L5ELEE2TAQUAAR44XV5RTTQHE47P9MKP5GQDLR9A8X-10422?func=item-global&doc_library=MIT01&doc_number=000355242&year=&volume=&sub_library=)。

阅读 K&R 的 5.1（指针与地址）至 5.5（字符指针和函数）部分。然后下载[pointers.c](https://pdos.csail.mit.edu/6.828/2018/labs/lab1/pointers.c)，运行它，确保你理解了所有打印的值都来自哪里。特别是，要确保你理解打印第 1 行和第 6 行中的指针地址来自哪里，打印第 2 行到第 4 行中的所有值是如何到达那里的，以及为什么在第 5 行中打印的值似乎是损坏的。

有很多关于 C 语言指针的材料(e.g., [A tutorial by Ted Jensen](https://pdos.csail.mit.edu/6.828/2018/readings/pointers.pdf)，虽然并没有那么强烈推荐。

警告：除非你已经完全精通 C 语言，否则不要跳过甚至略读这个阅读练习。如果你不能真正理解 C 语言中的指针，你将在随后的实验中经历无数的痛苦和痛苦，然后最终艰难地理解它们。相信我们；你不会想知道什么是"困难的道路"的。

为了理解 boot/main.c，你需要知道 ELF 二进制文件是什么。当你编译和链接一个 C 程序比如说 JOS 内核时，编译器会把每个 C 源文件（'.c'）转化为一个对象（'.o'）文件，它包含了硬件可以接受的汇编指令的编码。链接器然后组合所有这些编译过的对象文件到一整个二进制镜像比如说 obj/kern/kernel 里，在这种情况下是在 ELF 格式里的二进制，ELF 表示"Executable and Linkable Format"（可执行和可链接的格式）。

关于这种格式的完整信息可以在[ELF specification](https://pdos.csail.mit.edu/6.828/2018/readings/elf.pdf)这里找到，但在这门课中，你不需要深入钻研这种格式的细节。 尽管作为一个整体，这种格式非常强大和复杂，但大多数复杂的部分都用于支持共享库的动态加载，但我们在这个课程中不会做这个。[维基百科](http://en.wikipedia.org/wiki/Executable_and_Linkable_Format)里有一个简短的描述。

对于 6.828，你可以把 ELF 可执行文件看作是一个带有加载信息的头文件，后面跟着几个程序段，每个程序段都是一个连续的代码块或数据，打算加载到指定地址的内存中。boot loader 不修改代码或数据；它只是把 ELF 文件加载到内存中并开始执行。

ELF 二进制由一段固定长度的 ELF 头开始，后面是一个可变长度的程序头，列出要加载的每个程序段。这鞋 ELF 头的 C 语言定义在 inc/elf.h 里。我们关心的程序段是：

-   .text：程序的可执行指令们。
-   .rodata：只读数据，比如说 C 编译器产生的 ASCII 字符串常量。（我们不会费心设置硬件来禁止写入。）
-   .data：data 段放置程序段初始化数据，比如声明时初始化的全局变量，比如说 x=5;。

当链接器计算程序的内存布局时，它会为未初始化的全局变量预留空间，比如说 int x;，在一个紧跟着.data 段的叫做.bss 的段。C 要求未初始化的全局变量以零值开始。因此，ELF 二进制的.bss 段里不需要存储内容；相反，链接器只记录.bss 节的地址和大小。加载器或程序本身必须将.bss 节归零。

检查内核可执行文件的名字、大小和链接地址列表，可以输入：

```
athena% objdump -h obj/kern/kernel
```

（如果你自己编译你自己的工具链，你可能要用 i386-jos-elf-objdump）

你将看到比上面列出的更多的部分，但其他部分对我们的目的并不重要。其他的大多数用来保存调试信息，这些信息通常包含在程序的可执行文件中，但不会被程序加载器加载到内存中。

特别注意.text 段的“VMA”（link address）和“LMA”（load address）。一个段的加载地址就是该段应该被加载进内存的地址。

一个段的链接地址是该段预期执行的地址。链接器有几种方式把链接地址编码为二进制，比如说当代码需要全局变量的地址时，其结果是，如果从一个没有链接的地址执行，该二进制通常不会工作。（可以生成不包含任何绝对地址的位置无关代码。现代共享库广泛使用它，但它有性能和复杂性方面的代价，所以我们不会在 6.828 中使用它。）

通常，链接和加载地址是相同的。例如，查看 boot loader 的.text 部分:

```
athena% objdump -h obj/boot/boot.out
```

boot loader 使用 ELF 程序头来决定如何加载这些段。程序头指定将 ELF 对象的哪些部分加载到内存中，以及每个部分应该占用的目标地址。你可以查看程序头，键入：

```
athena% objdump -x obj/kern/kernel
```

程序头会在 objdump 的输出的"Program Headers"之下列出。ELF 对象中需要加载到内存的区域是那些标记为“LOAD”的区域。给出了每个程序头的其他信息，如虚拟地址(“vaddr”)、物理地址(“paddr”)和加载区域的大小(“memsz”和“filesz”)。

回到 boot/main.c，每个程序头的 ph->p_pa 字段包含了段的目标物理地址（在这种情况下，它真的是一个物理地址，虽然 ELF 标准对于这个字段的定义很模糊。）

BIOS 加载 boot 扇区到内存里，在地址 0x7c00 处开始，所以这个是 boot 扇区的加载地址。这也是 boot 扇区执行的位置，所以这也是它的链接地址。我们设置链接地址，通过 boot/Makefrag 传入参数 -Ttext 0x7c00 给链接器来实现，所以链接器会在生成的代码里加入正确的内存地址。

练习 5。再次追踪 boot loader 的最开始的几个指令，然后找出如果 boot loader 的链接地址错误的话会在哪个地址第一次出现“break”或其他错误。然后修改 boot/Makefrag 里的链接地址为错误的地址，运行 make clean，重新编译 make，然后再次追踪 boot loader 去看看会发生什么。不要忘了把链接地址给修改回去然后再重新编译回去。

回顾一下内核的链接和加载地址。不像 boot loader，这两个地址不一样：内核告诉 boot loader 去把它加载到内存的低地址（1M），但是期望从高地址开始执行。我们会在下一节看看如何做到这一点。

在段信息之外，ELF 头里还有另外一个字段对我们很重要，叫做 e_entry。这个字段持有程序入口的链接地址：在 text 段中程序应该执行的内存地址。你可以看到入口点：

```
athena% objdump -f obj/kern/kernel
```

你应该现在可以理解 boot/main.c 的最小 ELF loader 程序了。它从磁盘把内核的每个部分都读取出来，放到内存的加载地址里，然后跳转到内核的入口点。

练习 6。我们可以用 GDB 的 x 命令来查看内存。[GDB manual](https://sourceware.org/gdb/current/onlinedocs/gdb/Memory.html)有完整的细节，但是现在，知道 x/Nx ADDR 命令会打印 ADDR 的 N 字长内存就够了。（注意两个 x 都是小写的。）警告：word 的长度不是广泛的标准。在 GNU 汇编代码里，一个 word 是两个字节（xorw 里的 w 代表 word，意思是两个字节）。

重置机器(退出 QEMU/GDB 并重新启动它们)。在 BIOS 进入 boot loader 的 0x00100000 处的 8 个 word 长度的内存，然后再在 boot loader 进入内核时做一遍。为什么他们不一样？在第二个断点处有什么？（你不需要 QEMU 就能回答这个问题，思考就行。）

## Part 3: The Kernel（第 3 部分：内核）

我们现在查看最小的 JOS 内核的更多细节。（然后你要写一些代码！）就像 boot loader，内核是从一些汇编代码开始的，这些汇编代码设置一些事情来让 C 代码可以恰当执行。

### Using virtual memory to work around position dependence（使用虚拟内存来解决位置依赖问题）

当你检查上面的 boot loader 链接和加载地址时，它们完全匹配，但是内核的链接地址(由 objdump 打印)和它的加载地址之间有一个(相当大的)差异。回去检查一下，确保你能明白我们在说什么。(链接内核比 boot loader 更复杂，所以链接和加载地址在 kern/kernel.ld 的顶部。)

操作系统内核经常链接并运行在非常高的虚拟地址，比如 0xf0100000，为了给处理器虚拟地址空间的低处留给用户程序来用。这种安排的原因在下一个实验室中将会变得更清楚。

许多机器没有在 0xf010000 处的物理内存，所以我们不能指望把内核存储在那里。反而，我们会使用处理器的内存管理硬件（译者注：也就是 MMU）来映射虚拟地址 0xf0100000（内核代码期望去运行的链接地址）到物理地址 0x00100000（boot loader 加载内核进物理内存的位置）。通过这种方式，尽管内核的虚拟地址足够高，可以为用户进程留下足够的地址空间，但它将被加载到 PC RAM 中物理内存的 1MB 处，就在 BIOS ROM 上面。这种方法要求 PC 至少有几兆字节的物理内存(这样物理地址 0x00100000 才行)，但这可能适用于 1990 年以后构建的任何 PC。

实际上，在下一个实验中，我们会映射 PC 的整个底部 256MB 的物理地址空间，从物理地址 0x00000000 到 0x0fffffff，分别由虚拟地址 0xf0000000 到 0xffffffff 所映射。你现在应该知道为什么 JOS 可以使用物理地址的最开头的 256MB。

现在，我们将只映射第一个 4MB 的物理内存，这将足以让我们启动和运行。我们手写来完成这个，静态初始化 kern/entrypgdir.c 里的页目录和页表。到现在，你不需要了解它的实现细节，你只需要它取得的效果。在 kern/entry.S 设置 CR0_PG 标志位之前，内存引用都会被视为物理地址（杨哥来说，他们是线性地址，但是 boot/boot.S 设置一个特定的从线性地址到物理地址的映射，我们也不会修改这个）。一旦 CR0_PG 被设置，内存引用是由虚拟内存硬件转换为物理地址的虚拟地址。entry_pgdir 把虚拟地址的 0xf0000000 到 0xf0400000 的范围转换为物理地址的 0x00000000 到 0x00400000 范围，以及虚拟地址 0x00000000 到 0x00400000 到物理地址 0x00000000 到 0x00400000。任何虚拟地址不在这两个范围内的话，就会产生一个硬件异常，因为我们还没有设置好中断处理，所以它会导致 QEMU 去 dump 的机器状态并退出（或者无限重启，如果你没有使用 QEMU 的 6.828 补丁版本）。

练习 7。使用 QEMU 和 GDB 来追踪 JOS 内核，然后在 movl %eax, %cr0 处停止。检查 0x00100000 和 0xf0100000 处的内存。现在，使用 steppi GDB 命令跳过该指令。再一次，检查 0x00100000 和 0xf0100000 处的内存。确保你理解发生了什么。

建立新映射后，如果映射不到位，将无法正常工作的第一个指令是什么？注释掉 kern/entry.S 的 movl %eax, %cr0，追踪它，看看你是不是对的。

### Formatted Printing to the Console（格式化打印到控制台）

大多数人认为 printf()这样的函数是理所当然的，有时甚至认为它们是 C 语言的“原语”。但是在 OS 内核里，我们要自己去实现所有的 I/O。

通读 kern/printf.c、lib/printfmt.c 和 kern/console.c，确保你理解了它们的关系。在之后的实验中就会清楚为什么 printfmt.c 会被分开放置在 lib 目录里。

练习 8。我们省略了一小段代码——使用“%o”格式打印八进制数所需的代码。找到并填充这个代码。

要能够回答如下问题：

1. 解释 printf.c 和 console.c 接口的区别。具体来说，console.c 导出什么函数？什么函数被用在 printf.c 里？
2. 解释 console.c 如下代码：
    ```
    1      if (crt_pos >= CRT_SIZE) {
    2              int i;
    3              memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));
    4              for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
    5                      crt_buf[i] = 0x0700 | ' ';
    6              crt_pos -= CRT_COLS;
    7      }
    ```
3. 对于下列问题，你可能想要 Lecture 2 的笔记。这些笔记包含了 x86 上 GCC 的的调用约定。
   一步一步地追踪如下代码的执行：

    ```
    int x = 1, y = 3, z = 4;
    cprintf("x %d, y %x, z %d\n", x, y, z);
    ```

    - 在调用 cprintf()， fmt 指向什么?ap 指向什么?
    - 列出来（按照执行顺序）每个对 cons_putc，va_arg 和 vcprintf 的调用。对于 cons_putc，也把它的参数也列举出来。对于 va_arg，列出 ap 指针在调用之前和之后指向的内容。对于 vcprintf，列出它的两个参数的值。

4. 运行如下代码。

    ```
    unsigned int i = 0x00646c72;
    cprintf("H%x Wo%s", 57616, &i);
    ```

    输出是什么？解释这个输出是如何在前面练习的逐步方式中得到的。[Here's an ASCII table](http://web.cs.mun.ca/~michael/c/ascii-table.html)将字节映射到字符。

    输出依赖于 x86 是小端序的这一事实。如果 x86 是大端序，为了产生相同的输出，你会把 i 设置为什么？你需要把 57616 改变为另外一个值吗？

    这里有一份[小端序和大端序的描述](http://www.webopedia.com/TERM/b/big_endian.html)和[一个更古怪的描述](http://www.networksorcery.com/enp/ien/ien137.txt)。

5. 在如下代码里，'y='之后会打印什么?（注意：答案不是一个特定的值）这个是怎么发生的？

    ```
    cprintf("x=%d y=%d", 3);
    ```

6. 让我们假设 GCC 改变了它的调用约定，它按照声明的顺序将参数推入堆栈，这样最后一个参数就会最后一个被推入。那你要如何修改 cprintf 或者它的接口来让还是可以去传递给它一个可变数量的参数？

挑战！增强 console 来允许 text 被打印为不同的颜色。传统的方法是让它解释打印到控制台的文本字符串中嵌入的[ANSI 转义序列](http://rrbrandt.dee.ufcg.edu.br/en/docs/ansi/)，但是你可以使用任何你喜欢的机制。这里有很多信息：[the 6.828 reference page](https://pdos.csail.mit.edu/6.828/2018/reference.html)或关于 VGA 显示硬件的编程。如果你真的想冒险的话，你可以尝试将 VGA 硬件切换到图形模式，并使控制台将文本绘制到图形帧缓冲区上。

### The Stack

在这个实验的最后练习中，我们将更详细地探讨 C 语言在 x86 上使用堆栈的方式，并在此过程中编写一个有用的新的内核监视函数，用于打印堆栈的回溯信息：指向当前执行点的嵌套调用指令中保存的指令指针(IP)值的列表。

练习 9。内核在什么时候初始化它的栈的？以及它的栈在内存里什么位置？内核如何为它的栈预留空间的？保留区域的哪个“end”是栈指针最初初始化指向的位置？

x86 栈指针（esp 寄存器）指向当前使用的堆栈的最低位置。在那个位置的下面所有为栈预留的区域都是空闲的。将值压入堆栈需要减小堆栈指针，然后将值写入堆栈指针所指向的位置。从一个栈里推出值来读取值，然后增加栈指针的值。在 32 位模式，栈只能保存 32 位的值，esp 永远可以被 4 整除。很多 x86 的指令，比如说 call，是“硬接线”来使用栈指针寄存器的。

ebp（基值指针）寄存器，相反，主要是通过软件约定来和栈相联系。在 C 函数的入口，函数的序言代码通常通过将前一个函数的基指针压入堆栈来保存它，然后在函数运行期间将当前 esp 值复制到 ebp 中。如果一个程序里所有函数都遵循约定，那么在程序执行的任意一点，通过跟踪保存的 ebp 指针链并确定到底是哪个嵌套的函数调用序列导致程序中到达这个特定的点，可以通过堆栈进行回溯。这个功能可能特别有用，例如，当一个特定函数由于传递了错误的参数而导致断言失败或 panic 时，但是你不确定是谁传递了错误的参数。堆栈回溯可以让你找到有问题的函数。

练习 10。为了熟悉 x86 的 C 语言的调用约定，找到 obj/kern/kernel.asm 的 test_backtrace 函数的地址，在那儿设置断点，然后检查在内核启动后每次调用它时发生了什么。每个递归的 test_backtrace 嵌套级别在堆栈上推入多少个 32 位的 word，这些 word 是什么?

注意，为了让这个练习能正常工作，你应该使用补丁版的 QEMU，可以看看这个[tools](https://pdos.csail.mit.edu/6.828/2018/tools.html)页面。否则，必须手动将所有断点和内存地址转换为线性地址。

上面的练习应该提供了实现堆栈回溯函数所需的信息，你应该调用该函数 mon_backtrace()。这个函数的原型已经在 kern/monitor.c 中等着你了。完全可以在 C 语言中完成，但你可能会发现 inc/x86.h 中的 read_ebp()函数很有用。你还必须将这个新函数挂接到内核监视器的命令列表中，以便用户可以以交互方式调用它。

backtrace 函数应该以以下格式显示函数调用帧的列表:

```
Stack backtrace:
  ebp f0109e58  eip f0100a62  args 00000001 f0109e80 f0109e98 f0100ed2 00000031
  ebp f0109ed8  eip f01000d6  args 00000000 00000000 f0100058 f0109f28 00000061
  ...
```

每一行都包含了 ebp，eip 和 args。ebp 值表明函数使用的栈基址指针：也就是，堆栈指针在进入函数之后的位置，函数序言代码设置了基指针。列出的 eip 值是函数的返回指令指针：当函数返回时，控制权将返回到的指令地址。返回指令指针通常指向调用指令之后的指令(为什么?)最终，args 后的这 5 个十六进制的数字是传入函数的 5 个参数，这几个参数会在函数被调用的时候被推入栈。如果函数调用参数少于 5 个参数，当然，那么并不是所有这五个值都是有用的。（为什么 backtrace 不能实际上有多少个参数？如何修改这个限制？）

打印的第一行反应了当前执行的函数，被叫做 mon_backtrace，第二行反映了调用 mon_backtrace 的函数，第三行反映调用该函数的函数，依此类推。您应该打印所有未完成的堆栈帧。通过研究 kern/entry.S，你会发现有一种简单的方法告诉你什么时候停止。

以下是你在 K&R 第 5 章中读到的一些特别的要点，在接下来的练习和未来的实验中值得记住。

-   如果 int \*p = (int\*)100，那么(int)p + 1 and (int)(p + 1)是不同的数字：第一个是 101，而第二个是 104。当添加一个整数到指针上时，就像第二种情况，这个整数隐式乘以指针所指向的对象的大小。
-   p[i] 被定义为\*(p+i)，也就是 p 所指向的第 i 个对象。
-   &p[i] 和 (p+i)是一样的，返回 p 所指向的内存中第 i 个对象的地址。

尽管大多数 C 程序从不需要在指针和整数之间进行转换，但操作系统经常需要这样做。每当您看到涉及内存地址的加法时，请自问它是整数加法还是指针加法，并确保所加的值是否适当地相乘。

练习 11。实现上述指定的 backtrace 函数。使用与示例相同的格式，否则评分脚本会混淆。当你认为它可以正常工作时，运行 make grade 去看看它的输出和我们的打分脚本所期待的是否相同，如果不是那就继续修改它。当你递交你的 Lab 1 代码后，你可以随意修改 backtrace 的输出的格式。

如果使用 read_ebp()，请注意 GCC 可能会生成“优化”代码，在 mon_backtrace()的函数序言之前调用 read_ebp()，这会导致不完整的堆栈跟踪(最近的函数调用的堆栈帧丢失)。虽然我们已经尝试禁用导致这种重新排序的优化，但您可能需要检查 mon_backtrace()的程序集，并确保在函数序言之后发生对 read_ebp()的调用。

此时，你的回溯函数应该给你导致 mon_backtrace()被执行的堆栈上的函数调用者的地址。然而在实践中，您通常想知道与这些地址对应的函数名。例如，您可能想知道哪些函数可能包含导致内核崩溃的 bug。

为了帮助您实现这个功能，我们提供了 debuginfo_eip()函数，它在符号表中查找 eip 并返回该地址的调试信息。这个函数在 kern/kdebug.c 中定义。

练习 12。修改您的堆栈回溯函数，以显示每个 eip 对应的函数名、源文件名和行号。

In debuginfo*eip, where do \_\_STAB*\* come from? This question has a long answer; to help you to discover the answer, here are some things you might want to do:

-   look in the file kern/kernel.ld for \__STAB_\*
-   run objdump -h obj/kern/kernel
-   run objdump -G obj/kern/kernel
-   run gcc -pipe -nostdinc -O2 -fno-builtin -I. -MD -Wall -Wno-format -DJOS_KERNEL -gstabs -c -S kern/init.c, and look at init.s.
-   see if the bootloader loads the symbol table in memory as part of loading the kernel binary

通过插入对 stab_binsearch 的调用来查找地址的行号，以完成对 debuginfo_eip 的实现。

在内核监视器中添加一个 backtrace 命令，并扩展 mon_backtrace 的实现，调用 debuginfo_eip 并为表单的每个堆栈帧打印一行：

```
K> backtrace
Stack backtrace:
  ebp f010ff78  eip f01008ae  args 00000001 f010ff8c 00000000 f0110580 00000000
         kern/monitor.c:143: monitor+106
  ebp f010ffd8  eip f0100193  args 00000000 00001aac 00000660 00000000 00000000
         kern/init.c:49: i386_init+59
  ebp f010fff8  eip f010003d  args 00000000 00000000 0000ffff 10cf9a00 0000ffff
         kern/entry.S:70: <unknown>+0
K>
```

每一行给出了堆栈帧 eip 的文件名和文件中的行，后面跟着函数名和 eip 从函数的第一个指令开始的偏移量(例如，monitor+106 表示返回的 eip 在 monitor 的开始位置后 106 字节)。

确保在不同行打印文件和函数名，这样可以避免让打分脚本困惑。

Tip：printf 格式字符串提供了一种简单(尽管有些模糊)的方式来打印非以空结束的字符串，比如 stab 表中的字符串。printf("%.\*s", length, string)打印字符串的最大长度。请查看 printf 的 man 页，了解为什么它可以工作。

你可能会发现一些函数在回溯时丢失了。比如，你可能会看到一个对 monitor()的调用而不是 runcmd()。这是因为编译器内联了一些函数调用。其他优化可能会导致您看到意外的行号。如果您从 GNUMakefile 中去掉-O2，回溯可能会更有意义(但您的内核将运行得更慢)。

每一行给出了堆栈帧 eip 的文件名和文件中的行，后面跟着函数名和 eip 从函数的第一个指令开始的偏移量(例如，monitor+106 表示返回的 eip 在 monitor 的开始位置后 106 字节)。

确保将文件和函数名打印在单独的行上，以避免混淆打分脚本。
