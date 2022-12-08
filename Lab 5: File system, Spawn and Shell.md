# Lab 5: File system, Spawn and Shell

在这个实验中，你将要实现 spawn，一个用来加载并运行磁盘可执行文件的库程序。然后你将会充实你的内核和库以达到可以在 console 里允许 shell 的程度。这些特性需要一个文件系统，本实验会介绍一个简单的读/写文件系统。

## 文件系统初步

你要做的文件系统比大部分真实的文件系统都要简单，包括 xv6 Unix 的，但是提供基本的特性就非常不错了：创建、读取、写入以及删除文件，这些文件通过层级结构来组织。

我们(至少在目前)只开发一个单用户操作系统，它提供了足够的保护来捕捉 bug，但不能保护多个相互怀疑的用户。因此我们的文件系统不支持 UNIX 文件所有权或权限的概念。我们的文件系统目前也不支持硬链接、符号链接、时间戳或像大多数 UNIX 文件系统那样的特殊设备文件。

### 磁盘文件系统结构

大多数 UNIX 文件系统把磁盘空间切分为两种主要区域：inode 区域和 data 区域。UNIX 文件系统给文件系统里的每一个文件分配一个 inode；一个文件的 inode 包含了文件重要的元数据，比如说文件的属性和指向数据块的指针，文件系统在这个数据块里存储文件数据和目录元数据。目录项包含文件名和指向 inode 的指针；如果文件系统中的多个目录项指向该文件的 inode，则该文件被称为硬链接，我们不需要这种级别的文件关联，因此我们可以做一些简化：我们的文件系统根本就不使用 inode，而是简单地把所有文件的元数据存到描述这个文件的目录项。

文件和目录在逻辑上包含了一系列的数据块，这些数据块可能会分散在磁盘各处，就好像一个用户环境（进程）的虚拟地址分散映射到各个物理内存页。文件系统环境隐藏了数据块的分布细节，对外展示一个接口去读写文件任意偏移量的字节序列。文件系统环境在内部处理对目录的所有修改，作为执行文件创建和删除等操作的一部分。我们的文件系统确实允许用户环境直接去读取目录的元数据，这意味着用户环境可以自己执行目录扫描操作（实现 ls 程序），而不是依赖于额外的文件系统调用。这种方式的目录扫描的缺点，也是大多数现代 UNIX 所不推荐的，就是如果想更改文件系统的内部布局，应用程序也要跟着修改或者重新编译。

#### 扇区和区块

大多数磁盘无法执行字节粒度的读写，而是直接读写整个扇区。在 JOS 里，每个扇区 512 个字节。文件系统实际上以块为单位分配和使用磁盘存储。请注意这两个术语的区别：扇区大小是磁盘硬件决定的，而区块大小是操作系统决定的。一个文件系统的区块大小必须是该磁盘扇区大小的整数倍。

UNIX xv6 文件系统的一个区块为 512 比特，和扇区大小一样。但是大多数文件系统采用的是更大的区块，因为存储空间很便宜，可以在较大粒度上管理存储。我们的文件系统会使用 4096 比特大小的区块，以便和处理器的页大小匹配。

#### 超级区块（Superblocks）

文件系统把存储整个文件系统元数据的区块放到一个“非常好找”的磁盘位置（比如说磁盘的起始或末尾），这些元数据存储了区块大小、磁盘大小，以及任何任何寻找根目录所需的元数据，文件系统上次挂载的时间，文件系统上次 check error 的时间等等。这些特别的区块被称为超级区块。

我们的文件系统会有一个超级区块，它永远都是在磁盘的区块 1 上。它的布局被定义在`inc/fs.h`的`Super`结构体里。区块 0 是用来存放 boot loader 和分区表的，所以文件系统一般不会使用磁盘的第一个区块。许多“真实”的文件系统维护了多个超级区块，把这几个超级区块复制到多个宽区域里，这样如果其中一个区域的损坏或磁盘在区域中出现了错误，仍然可以找到其他超级块并使用它们访问文件系统。

#### 文件元数据

我们在`inc/fs.h`里定义的`File`结构体中定义里元数据的布局，这个用来描述一个文件系统的文件。这个元数据包含了文件名、大小、类型（正常文件或目录），以及一个指向存储该文件区块的指针。如上所述，我们不需要 inode，所以这个元数据存储在目录项里。不像大多数“真实“文件系统，为了简化，我们使用一个`File`结构体来代表文件元数据，这个元数据会存储在磁盘，也会出现在内存里。

`File`结构体里的`f_direct`数组，它包含了文件的前 10 个（NDIRECT）区块，我们把这些区块叫做直接文件直接区块。对于一个小于 10\*4096 = 40KB 大小的文件，它本身就可以完全包含在`File`结构体里。但是对于大一些的文件来说，我们需要其他地方来放置文件区块的编号。对于任何大于 40KB 的文件来说，因此，我们申请一个额外的磁盘区块，叫做文件间接区块，来保存额外的 4096/4 = 1024 个区块编号。我们的文件系统也因此允许文件达到 1034 个区块，也就是刚刚超过 4MB 大小。为了支持更大的文件，真实的文件系统一般都是 double-indirect 或者 trible-indirect 的区块。

#### 目录 vs 常规文件

我们文件系统的一个文件结构可以代表普通文件或者目录；这两种“文件”通过`type`字段来做区分。文件系统管理普通文件和目录的方式都是一样的，文件系统用同样的方式去管理普通文件和目录文件，除了一点，它不对普通文件相关联的数据块的内存进行解析，这个文件系统把目录文件的内容解析为一系列的 File 结构体，这些结构体用来描述该目录下的文件和子目录。

我们文件系统里的超级区块包含了一个`File`结构体（`Super`结构体里的`root`字段），这里面保存了文件系统根目录的元数据。这个目录里的内容是一串`File`结构体，这些结构体描述了在根目录下的文件和目录。任何根目录下的子目录也会包含`File`结构体来表示孙子目录，以此类推。

## 文件系统

这个实验的目的不是实现整个文件系统，而是实现一些关键的组件。特别的，你要负责把区块读到区块缓存里以及把它们重新刷回磁盘；申请磁盘区块；映射文件偏移到磁盘区块；实现 read、write、open（in the IPC interface）。因为你不用自己实现整个文件系统，但是很重要的是你要熟悉已经提供代码和不同类型的文件系统的接口。

### 访问磁盘

我们操作系统的文件系统需要能访问磁盘，但是我们还没有实现任何磁盘访问功能。我们并不把 IDE 磁盘驱动也放到内核里，而是放到用户级别的文件系统环境里。我们只会轻微地改造内核，以便让文件系统环境有访问磁盘的权限。

只要我们依赖于轮训、“programmed I/O” (PIO)-based 的磁盘访问、不使用磁盘中断，就很容易实现在用户空间的磁盘访问。也有可能在用户态下实现一个中断驱动的设备驱动（比如说 L3 和 L4 内核就是这样做的），但是由于内核必须接收设备中断并将它们分派到正确的用户模式环境中，这就更加困难了。

x86 处理器通过 EFLAGS 寄存器里的 IOPL 位来决定是否保护模式代码可以指向 I/O 指令，比如说 IN 和 OUT 指令。因为所有的 IDE 磁盘寄存器都是在 x86 的 I/O 空间，而不是通过存储器映射的方式，给文件系统“I/O 权限”是让它访问这些寄存器的唯一方式。实际上，EFLAGS 寄存器里的 IOPL 位给内核提供了一种“all-or-nothing”的方法来控制用户态代码是否可以访问磁盘。在我们的情况下，我们想要文件系统环境可以访问 I/O 空间，但是我们不想要任何用户环境可以访问 I/O 空间。

练习 1：把`ENV_TYPE_FS`传入用户环境创建函数`env_create`，`i386_init`通过这个参数来区分是不是文件系统环境。修改`env.c`里的的`env_create`，以便给文件系统环境 I/O 权限，但是一定不要给其他环境这个权限。

问题

你是如何保证当一个环境随后被切换为另外一个环境时，I/O 权限是正确被保存和恢复的？为什么？

注意，就像之前一样，GNUmakefile 文件设置了 QEMU 去使用 obj/kern/kernel.img 作为镜像，放到磁盘 0（在 DOS/Windows 里叫做“驱动 C”），然后使用一个新的文件 obj/fs/fs.img 作为镜像，放到磁盘 1（“驱动 D”）。在这个实验里，我们的文件系统应该只会接触磁盘 1；磁盘 0 只被用于启动内核。如果你以某种方式损坏任一磁盘映像，你可以将它们还原如初，“原始”版本只需要键入：

```
$ rm obj/kern/kernel.img obj/fs/fs.img
$ make
```

或者：

```
$ make clean
$ make
```

挑战！实现中断驱动的 IDE 磁盘访问，使用或不使用 DMA。你可以决定是否把设备驱动移到内核里，将它与文件系统一起放到用户空间里，或者甚至(如果您真的想了解微内核的精神)将它移到一个单独的环境中。

### 区块缓存

在我们的文件系统里，我们会在处理器虚拟内存系统的帮助下，实现一个简单的“buffer cache”（真的只是一个区块缓存）。代码在`fs/bc.c`里。

我们的文件系统把磁盘的大小限制在 3GB 以下。我们保留 3GB 大小的地址空间区域给文件系统，从 0x10000000（DISKMAP） 到 0xD0000000（DISKMAP+DISKMAX），作为“内存映射”版本的磁盘。例如，磁盘块 0 会被映射到虚拟地址 0x10000000，磁盘块 1 映射到虚拟地址 0x10001000，以此类推。`fs/bc.c`里的`diskaddr`函数实现里这个从磁盘区块号到虚拟地址的映射（包括一些完整性检查）。

我们的文件系统环境有它自己的虚拟地址空间，这和其他所有的用户环境的地址空间相独立的，文件系统环境环境唯一要做的事情就是去实现文件访问，以这种方式预留文件系统地址空间是合理的。一个真实的文件系统如果这样实现在 32 位机器上，那么是很尴尬的，因为现代磁盘很多比 3GB 大。像这样的 buffer cache 管理在 64 位的地址空间机器上来说也是合理的。

当然了，把整个磁盘读进内存要花很多时间，so instead we'll implement a form of demand paging, wherein we only allocate pages in the disk map region and read the corresponding block from the disk in response to a page fault in this region. This way, we can pretend that the entire disk is in memory.

练习 2：实现`fs/bc.c`里的`bc_pgfault`和`flush_block`两个函数，就像你在之前的实验里写的 copy-on-write fork 一样，除了这个是从磁盘中加载页之外。当写这个的时候，记住（1）`addr`可能没有和磁盘块对其（2）`ide_read`是在操作扇区而不是区块。

flush_block 函数应该把一个区块写入 disk。如果这个区块不在区块缓存里，那么 flush_block 不应该做任何事情。我们会使用 VM 硬件来追踪一个硬件在上次读取之后是否被修改过，以及上次写入磁盘后是否被修改过。为了查看一个区块是否需要写入，我们只要看`PTE_D`的“dirty”位是否设置了，通过 uvpt 项来查看这个。（PTE_D 位由处理器设置，以响应对该页的写操作；请参考 386 参考手册第 5 章的 5.2.4.3。）将该块写入磁盘后，`flush_block`应该使用`sys_page_map`清除`PTE_D`位。

Use make grade to test your code. Your code should pass "check_bc", "check_super", and "check_bitmap".

`fs/fs.c`里的`fs_init`函数是使用块缓存的一个主要例子。在初始化块缓存之后，它就简单地把指针存储在磁盘映射区域（在`super`结构体里）。在此之后，我们可以简单地从`super`结构体读取，好像它们就在内存里一样，而且我们的页错误处理程序也会把它们从磁盘里读出来。

Challenge! The block cache has no eviction policy. Once a block gets faulted in to it, it never gets removed and will remain in memory forevermore. Add eviction to the buffer cache. Using the PTE_A "accessed" bits in the page tables, which the hardware sets on any access to a page, you can track approximate usage of disk blocks without the need to modify every place in the code that accesses the disk map region. Be careful with dirty blocks.

### 区块位图

在`fs_init`设置位图指针，我们可以把位图当作一组比特数组，每一个区块用一个位代表。看，例如，`block_is_free`，它只是检查一个给定的块在位图中是否被标记为空闲。

练习三：使用`free_block`作为一个模型去实现`fs/fs.c`里的`allo_block`，应该到位图里找到一个空闲的磁盘块，并标记为已使用，以及返回该区块的数字。当你申请一个区块时，你应该立刻用`flush_block`把更改后的位图区块刷新到磁盘里，来保持文件系统的一致性。

### 文件操作

我们已经在`fs/fs.c`里提供了一些基础函数，你可以用这些函数来实现管理`File`结构体、扫描和管理目录文件的项，以及遍历整个文件系统来解析绝对路径。请先把`fs/fs.c`里的代码通读一遍，确保你理解了每个函数的功能。

练习 4：实现`file_block_walk`和`file_get_block`，file_block_walk maps from a block offset within a file to the pointer for that block in the struct File or the indirect block，非常像我们在`pgdir_walk`里为页表所做的事情。`file_get_block`再进一步地映射到了特定的磁盘块上，如果有必要就申请一个新的。

Use make grade to test your code. Your code should pass "file_open", "file_get_block", and "file_flush/file_truncated/file rewrite", and "testfile".

文件系统由`file_block_walk`和`file_get_block`负责。For example, file_read and file_write are little more than the bookkeeping atop file_get_block necessary to copy bytes between scattered blocks and a sequential buffer.

Challenge! The file system is likely to be corrupted if it gets interrupted in the middle of an operation (for example, by a crash or a reboot). Implement soft updates or journalling to make the file system crash-resilient and demonstrate some situation where the old file system would get corrupted, but yours doesn't.

### 文件系统接口

现在，文件系统用户环境已经有它必要的功能了，我们必须让其他用户环境可以使用文件系统。因为其他的用户环境不可以直接调用文件系统的函数，我们会通过 RPC 把访问暴露出去，这个是建立在 JOS 的 IPC 机制之上的。调用文件系统的过程如下：

```
      Regular env           FS env
   +---------------+   +---------------+
   |      read     |   |   file_read   |
   |   (lib/fd.c)  |   |   (fs/fs.c)   |
...|.......|.......|...|.......^.......|...............
   |       v       |   |       |       | RPC mechanism
   |  devfile_read |   |  serve_read   |
   |  (lib/file.c) |   |  (fs/serv.c)  |
   |       |       |   |       ^       |
   |       v       |   |       |       |
   |     fsipc     |   |     serve     |
   |  (lib/file.c) |   |  (fs/serv.c)  |
   |       |       |   |       ^       |
   |       v       |   |       |       |
   |   ipc_send    |   |   ipc_recv    |
   |       |       |   |       ^       |
   +-------|-------+   +-------|-------+
           |                   |
           +-------------------+
```

所有在点划线之下就是从一个文件系统环境读取普通用户环境的一个读请求。在开头出发，read 简单地文件描述符分发到适当的设备读功能上，在这种情况下是`devfile_read`（我们可以有更多种类的设备）。`devfile_read`实现了磁盘上的`read`。这个和`lib/file.c`里其他的`devfile_*`都实现了客户端的文件系统操作，运作方式都差不多，都是绑定把参数绑定到请求结构体里，调用`fsipc`去发送 IPC 请求，解包数据并返回结果。`fsipc`简单地处理向服务器发送请求并接收应答的常见细节。

文件系统服务器代码可以在`fs/server.c`中找到。它在`serve`函数中循环，不断地通过 IPC 接收请求，将该请求发送给适当的处理程序函数，并通过 IPC 将结果发送回来。在`read`的例子中，`serve`将分派给`serve_read`, `serve_read`将处理特定于读请求的 IPC 细节，比如解包请求结构，最后调用`file_read`来实际执行文件读操作。

回想一下 JOS 的 IPC 机制，它允许环境发送一个 32 位数字，并可以选择共享一个页。为了从客户端向服务器发送请求，我们使用 32 位的数字作为请求类型(文件系统服务器 rpc 被编号了，就像系统调用编号一样)，并将请求的参数存储在通过 IPC 共享的页面上的一个联合`Fsipc`中。在客户端，我们总是在`fsipcbuf`共享页面;在服务器端，我们将传入的请求页面映射到`fsreq (0x0ffff000)`。

服务器也通过 IPC 发送响应。我们使用 32 位数字作为函数的返回码。对于大多数 rpc 来说，这是它们返回的全部内容。`FSREQ_READ`和`FSREQ_STAT`也返回数据，它们只是简单地写到客户端请求发送的页上。不需要在响应 IPC 中发送此页，因为客户机首先将其共享给文件系统服务器。同样，在它的响应中，`FSREQ_OPEN`与客户端共享一个新的“Fd 页”。我们将快速返回到文件描述符页。

练习 5：实现 fs/serv.c 里的 serve_read。

serve_read 繁重的工作由已经实现的 file_read（fs/fs.c）来完成（反过来说，file_read 也只是一堆对 file_get_block 的调用）。serve_read 只是为文件读取提供了 RPC 接口。阅读一下 serve_set_size 里的注释和代码，来大致了解 server 功能应该如何组织的。

练习 6：实现`fs/serv.c`里的`serve_write`，以及`lib/file.c`里的`devfile_write`。

## Spawning Processes

我们已经为 `spawn` (`lib/spawn.c`)写了一些代码，会创建一个新的用户环境，加载文件系统里的程序镜像，然后在子用户环境里开始运行这个程序。父进程接着独立于子进程继续运行。`spawn`就像是 UNIX 里的`fork`，之后在子进程里调用`exec`。

我们实现了 `spawn` 而不是 UNXI 那样的 `exec`，因为这样更简单一些。想想你在用户空间里实现 `exec`，你会怎么做？确保你理解了为什么这样更难实现。

练习 7：`spawn`依赖于新的系统调用`sys_env_set_trapframe`，来初始化新创建用户环境的状态。实现`sys_env_set_trapframe`(位于`kern/syscall.c`)。

挑战！实现 Unix-style 的 exec。

挑战！实现 mmap-style 的 memory-mapped 文件，修改 spawn，来尽可能从 ELF 映像直接映射内存页。

### 在 fork 和 spawn 之间共享库状态

UNIX 文件描述符是一个通用的概念，它还包括了管道、console I/O 等。在 JOS 里，这些设备类型都有一个对应的结构体 Dev，带着一个指向为该设备类型实现的 read/write 等函数的指针。lib/fd.c 此基础上实现了通用的类 UNIX 文件描述符接口。每个结构体 Fd 指示了设备的类型，lib/fd.c 的大部分功能就是简单地把操作分发到合适的结构体 Dev 所指向的函数。

lib/fd.c 也维护了在各个环境的地址空间里的文件描述符表区域，这个区域从 FDTABLE 开始。该区域为应用程序可以同时打开的 MAXFD(目前为 32)个文件描述符保留一个页面的(4KB)地址空间。在任何时刻，当且仅当对应的文件描述符正在使用时，才映射特定的文件描述符表页。每个文件描述符在 FILEDATA 开始的区域中也有一个可选的 "data page（数据页）" ，设备可以选择使用它。

我们希望在 fork 和 spawn 之间共享文件描述符状态，但是文件描述符状态保存在用户空间内存中。现在，在 fork 上，内存会被标记为 copy-on-write，所以状态会被复制而不是共享。（这意味着环境无法在它们没有打开的文件中查找，这样管道就在 fork 上就无法工作。）在 spawn 上，内存会留下，根本不会被复制。（spawn 的环境一开始是没有文件描述符的。）

我们将改变 fork，使其知道某些内存区域是由“库操作系统”使用的，并且应该总是被共享的。我们不硬编码一个内存区域的列表，而是在页表目录里设置一个未使用的位（就像我们在 fork 里用的 PTE_COW 位一样）。

我们在 inc/lib.h 里定义一个新的 PTE_SHARE 位。这个位是三个 PTE 位之一，根据 Intel 和 AMD 手册，这些 PTE 位是用来标记“可供软件使用”的。我们将建立一个约定，如果一个页表条目设置了这个位，PTE 应该在 fork 和 spawn 中直接从父环境复制到子环境。注意这个和标记 copy-on-write 的区别：如第一个段落所述，我们希望确把更新也共享给这个页。

练习 8。修改 lib/fork.c 里的 duppage，使其遵循新的约定。如果这个页表项的 PTE_SHARE 位被设置了，那就直接复制映射。（你应该使用 PTE_SYSCALL，而不是 0xfff，来掩掉页表项相关的位。0xfff 也会获取被访问的位和脏位。）

同样地，实现 lib/spawn.c 里的 copy_shared_pages。 它应该遍历当前进程中的所有页表条目(就像 fork 所做的那样)，将设置了 PTE_SHARE 位的任何页映射复制到子进程中。

Use make run-testpteshare to check that your code is behaving properly. You should see lines that say "fork handles PTE_SHARE right" and "spawn handles PTE_SHARE right".

Use make run-testfdsharing to check that file descriptors are shared properly. You should see lines that say "read in child succeeded" and "read in parent succeeded".

## 键盘接口

For the shell to work, we need a way to type at it. QEMU has been displaying output we write to the CGA display and the serial port, but so far we've only taken input while in the kernel monitor. In QEMU, input typed in the graphical window appear as input from the keyboard to JOS, while input typed to the console appear as characters on the serial port. kern/console.c already contains the keyboard and serial drivers that have been used by the kernel monitor since lab 1, but now you need to attach these to the rest of the system.

Exercise 9. In your kern/trap.c, call kbd_intr to handle trap IRQ_OFFSET+IRQ_KBD and serial_intr to handle trap IRQ_OFFSET+IRQ_SERIAL.

We implemented the console input/output file type for you, in lib/console.c. kbd_intr and serial_intr fill a buffer with the recently read input while the console file type drains the buffer (the console file type is used for stdin/stdout by default unless the user redirects them).

Test your code by running make run-testkbd and type a few lines. The system should echo your lines back to you as you finish them. Try typing in both the console and the graphical window, if you have both available.

## The Shell

Run make run-icode or make run-icode-nox. This will run your kernel and start user/icode. icode execs init, which will set up the console as file descriptors 0 and 1 (standard input and standard output). It will then spawn sh, the shell. You should be able to run the following commands:

```
   echo hello world | cat
	cat lorem |cat
	cat lorem |num
	cat lorem |num |num |num |num |num
	lsfd
```

Note that the user library routine cprintf prints straight to the console, without using the file descriptor code. This is great for debugging but not great for piping into other programs. To print output to a particular file descriptor (for example, 1, standard output), use fprintf(1, "...", ...). printf("...", ...) is a short-cut for printing to FD 1. See user/lsfd.c for examples.

Exercise 10.

The shell doesn't support I/O redirection. It would be nice to run `sh < script` instead of having to type in all the commands in the script by hand, as you did above. Add I/O redirection for < to user/sh.c.

Test your implementation by typing `sh < script` into your shell

Run `make run-testshell` to test your shell. testshell simply feeds the above commands (also found in fs/testshell.sh) into the shell and then checks that the output matches fs/testshell.key.

Challenge! Add more features to the shell. Possibilities include (a few require changes to the file system too):

-   backgrounding commands (ls &)
-   multiple commands per line (ls; echo hi)
-   command grouping ((ls; echo hi) | cat > out)
-   environment variable expansion (echo $hello)
-   quoting (echo "a | b")
-   command-line history and/or editing
-   tab completion
-   directories, cd, and a PATH for command-lookup.
-   file creation
-   ctl-c to kill the running environment

but feel free to do something not on this list.

Your code should pass all tests at this point. As usual, you can grade your submission with `make grade` and hand it in with `make handin`.

This completes the lab. As usual, don't forget to run `make grade` and to write up your answers and a description of your challenge exercise solution. Before handing in, use `git status` and `git diff` to examine your changes and don't forget to `git add answers-lab5.txt`. When you're ready, commit your changes with `git commit -am 'my solutions to lab 5'`, then `make handin` to submit your solution.
