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

### Sysinfo

​	`argint,argaddr,argfd`分别获取系统调用中的整数、地址、文件描述符  

同上,先添加sysinfo系统调用。

#### 1.实现获取空闲内存量

##### 1.1 看代码学习

首先看看**kernel/kalloc.c**文件。

kalloc.c是XV6系统用来实现物理内存分配的,采用**分页**管理,每页4096字节。

首先看下面这段代码

```c
struct run {
  struct run *next;
};
```

这段代码定义了一个结构体类型run，其中只包含一个成员变量next，它是一个指向结构体类型run的指针。

这个结构体类型通常用于实现内存空闲链表，即用于管理已经被释放的内存页面。具体来说，每个结构体实例代表着一个已经被释放的内存页面，而next成员变量则指向下一个空闲页面的结构体实例。通过这种方式，可以将所有的空闲页面组织成一个链表，方便对它们进行管理和分配。

当需要分配一个新的内存页面时，可以从这个链表中取出一个空闲页面，将它标记为已经被占用，并返回其指针给调用者。

当需要释放一个已经被占用的内存页面时，可以将它转换成一个run结构体指针，并将其添加到内存空闲链表中。这样，下一次需要分配内存时，就可以从这个链表中取出一个空闲页面，并将其标记为已经被占用。

------

然后是

```c
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem;
```

这段代码定义了一个结构体类型，它包含了两个成员变量：一个spinlock类型的lock，和一个指向run类型的指针**freelist**。这个结构体类型被命名为**kmem**。

具体来说，这个结构体类型通常用于内核内存管理器（kernel memory manager）。其中，spinlock类型的lock用于实现对内存空闲列表的互斥访问，以确保多个线程访问内存空闲列表时不会产生竞争条件（race condition）和数据一致性问题。而指针freelist则指向一个run类型的链表，其中每个run结构体实例代表着一个已经被释放的内存页面。

------

xv6采用空闲链表法来记录空闲页。

其**分配空闲页的函数**如下:

```c
void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r)
    kmem.freelist = r->next;
  release(&kmem.lock);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

1. 首先获得空闲链表头,如果链表头即`r`不为空,说明存在空闲页,同时奖链表头指向下一个页面即`r->next`
2. 接着，如果r不为空，则使用memset函数将这个页面填充为5，以便在后续使用时能够检测到**悬空引用**的情况。

> 悬空引用（dangling reference）是指一个指针指向的内存已经被释放掉了，但是指针本身仍然有效，继续被使用。这种情况下，使用指针指向的内存会导致不可预测的行为，可能会造成严重的安全问题。
>
> 例如，当一个程序释放一个内存页面之后，这个页面中的内容就不再受到保护。如果在释放后的程序中仍然使用了这个页面，就可能会产生悬空引用。在这种情况下，程序可能会读取到无效的数据，或者修改无关的内存，导致程序崩溃或者出现其他严重问题。
>
> 例如这段程序代码中已经将所有字节填充为5,那么别的程序之前引用这块内存的指针继续使用就会发现内存中的数据已经被修改了,从而停止继续使用。

3. 这段代码最后要返回`void*`类型的指针，是为了能够适应不同类型的数据。因为动态分配的内存空间可以用来存储各种不同类型的数据，而不同类型的数据占用的字节数是不同的。

   如果这个函数只返回一个特定类型的指针，那么在使用这个函数时，就需要为每种类型的数据编写一个对应的函数，这样会增加代码的复杂性和维护成本。

   而使用`void*`类型的指针，则可以将其指向任意类型的数据，从而使得函数更加灵活和通用。在函数返回一个void*类型的指针后，调用者可以将其强制转换成特定类型的指针，以便在使用这个内存空间时，可以正确地访问其中的数据。但是需要注意的是，在转换指针类型时，需要确保转换后的指针类型与实际存储在这段内存空间中的数据类型相同，否则可能会导致访问内存越界或其他类型错误。

其**释放内存页的函数**如下:

```c
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  release(&kmem.lock);
}
```

这段代码是一段用于释放内核空间中动态分配的页面的代码。具体来说，它将一个指向要释放页面的指针作为参数传入函数中，然后进行如下操作：

1. 首先，它检查传入的指针是否指向一个页面的开头，以及是否位于内核空间的有效范围内。如果不是，则会调用panic函数，导致系统崩溃。
2. 接着，它使用memset函数将要释放的页面填充为1，以便在后续使用时能够检测到悬空引用的情况。
3. 然后，它将指向要释放页面的指针转换为一个结构体指针，以便可以将该页面添加到内存空闲列表中。
4. 最后，它获取内核内存管理器的锁，将要释放的页面添加到内存空闲列表中，并释放锁，使得其他线程可以访问该内存空闲列表。

##### 1.2 实现

```c
void
acquire_freebytes(uint64 *n)
{
  struct run *r;
  *n = 0;
  acquire(&kmem.lock);
  r = kmem.freelist;
  while(r){
    r = r->next;
    *n += PGSIZE;
  }
  release(&kmem.lock);
}
```

实现则通过不断遍历freelist,当r不为NULL,则说明存在一个空闲页,则空闲字节数加上一个页面的大小,并往下继续进行遍历。

#### 2.实现获取`state`字段不为`UNUSED`的进程数

注意到`kernel/proc.c`中的 `struct proc proc[NPROC]`

![image-20230519023657404](https://typora-picgo-picbed.oss-cn-shenzhen.aliyuncs.com/img/202305190237580.png)

它存储了每个进程的进程状态,所以遍历这个数组进行判断就可以了

```c
void
acquire_freeprocess(uint64 *n)
{  
    struct proc *p;
    *n = 0;
    for(p = proc; p < &proc[NPROC]; p++){
      if(p->state!=UNUSED)
        *n += 1;
    }
}
```

#### 3.实现系统调用

```c
uint64
sys_sysinfo(void)
{

  struct sysinfo sys_info;
  acquire_freebytes(&sys_info.freemem);
  acquire_freeprocess(&sys_info.nproc);

  // 虚拟地址
  uint64 dstaddr;
  if(argaddr(0, &dstaddr) < 0)
    return -1;
  struct proc *p = myproc();
  
  if(copyout(p->pagetable, dstaddr, (char *)&sys_info, sizeof(sys_info)) < 0)
      return -1;
  return 0;
  
}
```

1. 首先获得相应的信息
2. `argaddr`从用户态的系统调用的参数中获得对应的指针赋值给dstaddr
3. `copyout`将sys_info复制到`dstaddr`所指的用户空间中

### 测试结果

![image-20230519025003704](https://typora-picgo-picbed.oss-cn-shenzhen.aliyuncs.com/img/202305190250746.png)