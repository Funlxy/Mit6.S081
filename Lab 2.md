# 笔记

整个操作系统都驻留在内核中，这样所有系统调用的实现都以管理模式运行。这种组织被称为**宏内核（monolithic kernel）**。

宏组织的一个缺点是操作系统不同部分之间的接口通常很复杂（正如我们将在本文的其余部分中看到的），因此操作系统开发人员很容易犯错误。在宏内核中，一个错误就可能是致命的，因为管理模式中的错误经常会导致内核失败。如果内核失败，计算机停止工作，因此所有应用程序也会失败。计算机必须重启才能再次使用。

为了降低内核出错的风险，操作系统设计者可以最大限度地减少在管理模式下运行的操作系统代码量，并在用户模式下执行大部分操作系统。这种内核组织被称为**微内核（microkernel）**。

![img](https://typora-picgo-picbed.oss-cn-shenzhen.aliyuncs.com/img/202305120107024.png)

图2.1说明了这种微内核设计。在图中，文件系统作为用户级进程运行。作为进程运行的操作系统服务被称为服务器。为了允许应用程序与文件服务器交互，内核提供了允许从一个用户态进程向另一个用户态进程发送消息的进程间通信机制。例如，如果像shell这样的应用程序想要读取或写入文件，它会向文件服务器发送消息并等待响应。



在微内核中，内核接口由一些用于启动应用程序、发送消息、访问设备硬件等的低级功能组成。这种组织允许内核相对简单，因为大多数操作系统驻留在用户级服务器中。

像大多数Unix操作系统一样，Xv6是作为一个宏内核实现的。因此，xv6内核接口对应于操作系统接口，内核实现了完整的操作系统。由于xv6不提供太多服务，它的内核可以比一些微内核还小，但从概念上说xv6属于宏内核

Xv6为每个进程维护一个单独的页表，定义了该进程的地址空间。如图2.3所示，以虚拟内存地址0开始的进程的用户内存地址空间。首先是指令，然后是全局变量，然后是栈区，最后是一个堆区域（用于`malloc`）以供进程根据需要进行扩展。有许多因素限制了进程地址空间的最大范围： RISC-V上的指针有64位宽；硬件在页表中查找虚拟地址时只使用低39位；xv6只使用这39位中的38位。因此，最大地址是2^38-1=0x3fffffffff，即`MAXVA`（定义在**kernel/riscv.h**:348）。在地址空间的顶部，xv6为`trampoline`（用于在用户和内核之间切换）和映射进程切换到内核的`trapframe`分别保留了一个页面，正如我们将在第4章中解释的那样。

![img](https://typora-picgo-picbed.oss-cn-shenzhen.aliyuncs.com/img/202305120149164.png)

xv6内核为每个进程维护许多状态片段，并将它们聚集到一个`proc`(**kernel/proc.h**:86)结构体中。一个进程最重要的内核状态片段是它的页表、内核栈区和运行状态。我们将使用符号`p->xxx`来引用`proc`结构体的元素；例如，`p->pagetable`是一个指向该进程页表的指针。

每个进程都有一个执行线程（或简称线程）来执行进程的指令。一个线程可以挂起并且稍后再恢复。为了透明地在进程之间切换，内核挂起当前运行的线程，并恢复另一个进程的线程。线程的大部分状态（本地变量、函数调用返回地址）存储在线程的栈区上。每个进程有两个栈区：一个用户栈区和一个内核栈区（`p->kstack`）。当进程执行用户指令时，只有它的用户栈在使用，它的内核栈是空的。当进程进入内核（由于系统调用或中断）时，内核代码在进程的内核堆栈上执行；当一个进程在内核中时，它的用户堆栈仍然包含保存的数据，只是不处于活动状态。进程的线程在主动使用它的用户栈和内核栈之间交替。内核栈是独立的（并且不受用户代码的保护），因此即使一个进程破坏了它的用户栈，内核依然可以正常运行。

一个进程可以通过执行RISC-V的`ecall`指令进行系统调用，该指令提升硬件特权级别，并将程序计数器（PC）更改为内核定义的入口点，入口点的代码切换到内核栈，执行实现系统调用的内核指令，当系统调用完成时，内核切换回用户栈，并通过调用`sret`指令返回用户空间，该指令降低了硬件特权级别，并在系统调用指令刚结束时恢复执行用户指令。进程的线程可以在内核中“阻塞”等待I/O，并在I/O完成后恢复到中断的位置。

`p->state`表明进程是已分配、就绪态、运行态、等待I/O中（阻塞态）还是退出。

`p->pagetable`以RISC-V硬件所期望的格式保存进程的页表。当在用户空间执行进程时，Xv6让分页硬件使用进程的`p->pagetable`。一个进程的页表也可以作为已分配给该进程用于存储进程内存的物理页面地址的记录。

# 实验

### System call tracing

#### 1.修改用户代码

1.在Makefile中的UPROGS中添加$U/_trace

<img src="https://typora-picgo-picbed.oss-cn-shenzhen.aliyuncs.com/img/202305162121359.png" alt="image-20230516212139121" style="zoom:67%;" />

2.在user/user.h中添加函数声明

<img src="https://typora-picgo-picbed.oss-cn-shenzhen.aliyuncs.com/img/202305162123036.png" alt="image-20230516212343996" style="zoom:67%;" />

3.在user/usys.pl中添加入口

<img src="https://typora-picgo-picbed.oss-cn-shenzhen.aliyuncs.com/img/202305162352181.png" alt="image-20230516235232001" style="zoom:67%;" />

在make时会调用usys.pl生成user/usys.S汇编程序。在该汇编程序中，每个函数有五行，其中有三条指令，其功能是将系统调用号通过li(load imm)存入a7寄存器，之后使用ecall指令进入内核态，最后返回。

make后user/usys.S中将出现下图中.global trace开始的5行代码。

![image-20230516235440027](https://typora-picgo-picbed.oss-cn-shenzhen.aliyuncs.com/img/202305162354058.png)

#### 2.修改内核代码

1. `kernel/syscall.h` 中添加 trace 系统调用的号码

<img src="https://typora-picgo-picbed.oss-cn-shenzhen.aliyuncs.com/img/202305162357919.png" alt="image-20230516235717877" style="zoom:67%;" />

2.在kernel/syscall.c的syscalls函数指针数组中添加对应的函数。

<img src="https://typora-picgo-picbed.oss-cn-shenzhen.aliyuncs.com/img/202305180207055.png" alt="image-20230518020648569" style="zoom:67%;" />

3.kernel/proc.h中在proc结构体中添加一个trace_mask字段

<img src="https://typora-picgo-picbed.oss-cn-shenzhen.aliyuncs.com/img/202305180208664.png" alt="image-20230518020810615" style="zoom:67%;" />

4.kernel/proc.c中在fork函数中添加以复制该字段给子进程

<img src="https://typora-picgo-picbed.oss-cn-shenzhen.aliyuncs.com/img/202305180222234.png" alt="image-20230518022230193" style="zoom:67%;" />

5.系统调用sys_trace的实现。在kernel/sysproc.c中添加函数uint64 sys_trace(void)，该函数通过argint函数读取参数赋值给mask变量

<img src="https://typora-picgo-picbed.oss-cn-shenzhen.aliyuncs.com/img/202305180233074.png" alt="image-20230518023347040" style="zoom:67%;" />

6.在kernel/syscall.c中首先添加系统调用号与系统调用名的映射数组,然后修改syscall.c函数,添加打印输出

<img src="https://typora-picgo-picbed.oss-cn-shenzhen.aliyuncs.com/img/202305180235547.png" alt="image-20230518023538504" style="zoom:67%;" />

<img src="https://typora-picgo-picbed.oss-cn-shenzhen.aliyuncs.com/img/202305180235720.png" alt="image-20230518023552683" style="zoom:67%;" />

  