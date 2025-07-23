# Mit6.S081-实验10-mmap

开启实验之前，需要切换本次实验分支：

```bash
git checkout mmap
```

## 基础概念

> Linux的内存布局 vs. XV6的内存布局

![image-20250714233617622](C:\Users\lione\Documents\Notes\CppNote\WilesNote\MIT_OS_6.S081\assets\image-20250714233617622.png)

上述图片是XV6操作系统的内存布局，可以看到从上至下，分别是trampoline、trapframe、heap、stack、guard page、data、text。

* trampoline：一段小型代码，在系统调用或者中断处理的时候，CPU通过trampoline代码安全地进入内核，避免直接暴露内核地址。
* trapframe：发生中断、异常或系统调用的时候，内核将寄存器状态保存到trapframe中。
* heap：动态内存分配的区域，供程序运行时申请/释放内存（如 `malloc()`/`free()`）。
* stack：存储函数调用的局部变量、返回地址、参数等。
* guard page：内存中的特殊页面，用于检测栈溢出或堆越界访问。
* data：存储程序的全局变量和静态变量。
* Text：存储程序的机器指令（即代码）。

接下来，我们再看Linux操作系统的内存布局是怎样的。

![image-20250614113403113](C:\Users\lione\Documents\Notes\CppNote\WilesNote\MIT_OS_6.S081\assets\image-20250614113403113.png)

从高地址到低地址可以分为五个区域：

- 栈区：操作系统控制，由高地址向低地址生长，编译器做了优化，显示地址时栈区和其他区域保持一致的方向。(同一个区域，先定义的内容在较低的地址),存放栈变量、栈对象。局部变量、函数的参数等，编译器会进行自动分配与释放。

- 堆区：程序员分配，由低地址向高地址生长，堆区与栈区没有明确的界限。`malloc/new`等操作申请的空间。

- 全局/静态区：读写段（数据段），存放全局变量、静态变量。

- 文字常量区：只读段，存放程序中直接使用的文字常量和全局的常量。

- 程序代码区：只读段，存放函数体的二进制代码。

可以看到，Linux的内存布局和XV6的内存布局有一点明显的差异，在于Linux的内存布局中，从上到下是stack再到heap，而XV6从上到下是heap到stack。

> 本次实验需要实现的MMAP是什么呢？

在了解MMAP的时候，我们可以拓展了解什么是零拷贝技术，而本文则不再赘述，零拷贝技术是什么，详细请参考本文最后的内容《什么是内存映射》 。

MMAP（内存映射）是Unix/Linux系统中一种机制，允许进程将文件或者设备直接映射到其虚拟地址空间中，从而可以像访问内存一样读写文件，而无需显式调用`read()`或者`write()`等系统调用。

本次实验中，实现文件系统的MMAP的核心目标在于：页表机制 + 按需分配

1. 页表机制：动态管理虚拟地址到物理内存/文件的映射。
2. 按需分配：仅在访问时分配物理页（触发缺页异常后再加载数据）。



## 具体实验细节（Hard）

### 定义VMA

VMA记录了进程通过mmap申请的虚拟内存区域的各种信息，仅仅描述虚拟内存区域的逻辑属性。

页表存储在内核的页表数据结构中，（如 xv6 的 `pagetable_t`），通过 `walk` 函数查询。当进程访问 VMA 描述的虚拟地址时，内核会：

1. 查页表，若缺失则触发缺页中断
2. 根据 VMA 的信息分配物理页，并更新页表。

因此，我们可以梳理出VMA和页表的关系：VMA 看不到页表，但页表能感知 VMA。

在定义VMA数据结构之前，我们需要先关心MMAP使用的地址空间放在哪里呢？

从XV6的内存布局中可以看到，进程所使用的内存空间是从低地址到高地址生长的，范围就是从stack到trapframe，为了不和进程使用的内存空间发生冲突，则需要将mmap使用的地址空间映射到trapframe下的页，从上往下生长（即高地址到低地址），我们先定义mmap的最后一页的结束地址为trapframe的起始地址。

```c++
// kernel/memlayout.h

// MMAP进程映射文件内存最后一个页
#define MMAPEND TRAPFRAME
```

完成之后，我们再开始定义VMA数据结构，用于记录mmap创建的虚拟内存地址的范围、长度、权限、文件等。

```c++
// kernel/proc.h

// 定义一个虚拟内存区域结构体，用来记录mmap创建的虚拟内存地址的范围、长度、权限、文件等
struct vma{
  int valid;  // 该虚拟内存区域是否已经被映射
  uint64 vastart; // 该虚拟内存区域开始地址
  uint64 sz;  // 该虚拟内存区域大小
  struct file* f; // 该虚拟内存区域映射的文件
  int prot; // 该虚拟内存区域权限
  int flags;  // 标记映射内存的修改是否写回文件
  uint64 offset;  // 映射文件的起点
};
```

由于每个文件映射需要独占一个VMA条目，所以我们还需要定义VMA数组，这个VMA数组，直接在proc中添加即可。

```c++
// kernel/proc.h

#define NVMA 16 // VMA数组大小

// .........

// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
  struct vma vmas[NVMA];       // mmap虚拟内存映射地址数组
};
```

完成了上述操作之后，即可开始mmap系统调用的实现了。



### 实现mmap系统调用

参考Linux的mmap函数，我们需要在进程的vmas数组中遍历寻找空闲的vma。

> `sys_mmap(void)`：创建新的虚拟内存映射

1. 读取用户建议的映射起始虚拟地址addr、映射长度sz、权限标志prot、映射类型flag、文件描述符fd、指向文件的指针f。
2. 若文件不可读但请求的是PROT_READ，则拒绝映射。
3. 若我呢见不可写但请求的是PROT_WRITE，且非私有映射MAP_PRIVATE，则拒绝映射。
4. 由于操作系统都是按页分配内存的，所以我们需要让用户建议的映射长度sz进行一个向上取整操作，以达到页面大小的整数倍。
5. 获取当前进程并且初始化VMA。
6. 遍历VMA数组，寻找空闲条目并计算可用地址。
   1. 找到第一个未使用的条目，并且标记为已用。
   2. 计算最低可用地址，确保新映射不会和现有的VMA重叠。
7. 若VMA数组已满，则需`panic("mmap: no free vma")`。
8. 能走到这一步，已经说明获取到了空闲的VMA条目，此时此刻就需要设置VMA的属性。
9. 最后，还需要增加文件引用计数以及返回映射的起始地址。

```c++
// kernel/sysfile.c

#include "memlayout.h"

// 创建新的虚拟内存映射
uint64 sys_mmap(void){
  uint64 addr, sz, offset;
  int prot, flag, fd;
  struct file* f;

  // 读取传入参数
  if(argaddr(0, &addr) < 0 || argaddr(1, &sz) < 0 || argint(2, &prot) < 0 
  || argint(3, &flag) < 0 || argfd(4, &fd, &f) < 0 || argaddr(5, &offset) < 0 || sz ==0){
    return -1;
  }

  if((!f->readable && ((prot & (PROT_READ))))
    || (!f->writable && (prot & PROT_WRITE) && !(flag & MAP_PRIVATE))){
    return -1;    
  }

  sz = PGROUNDUP(sz);
  struct proc* p = myproc();  // 获取当前进程控制块
  struct vma* v = 0;          // 初始化空闲vma指针
  uint64 vaend = MMAPEND;     // 初始化mmap区域结束地址（高地址起点）

  // 遍历查找空闲vma并计算最低可用地址
  for(int i = 0; i < NVMA; i++){
    struct vma* vv = &p->vmas[i];
    if(vv->valid == 0){
      if(v == 0){
        v = &p->vmas[i];
        v->valid = 1;
      }
    }
    else if(vv->vastart < vaend){
      vaend = PGROUNDDOWN(vv->vastart);
    }
  }

  // 没找到空闲的vma
  if(v == 0){
    panic("mmap: no free vma");
  }

  // 设置vma属性
  v->vastart = vaend - sz;
  v->sz = sz;
  v->f = f;
  v->prot = prot;
  v->flags= flag;
  v->offset = offset;

  // 增加源文件引用数
  filedup(v->f);

  return v->vastart;
}
```

> 按需分配的实现

上述`sys_mmap(void)`的实现，仅仅只是找到一个空闲的VMA并初始化它，但是实际上，按需分配这个操作仍未实现。

那按需分配的实现，需要依赖两个操作：查询任意虚拟地址所属的VMA + 给虚拟地址分配物理页并且建立映射。

当发生缺页中断的时候（`trap.c`），调用`findvma`和`vmaalloc`来完成按需分配。

```c++
// kernel/sysfile.c

// 通过虚拟地址va查找进程所属的虚拟内存区域vma
// 查询任意虚拟地址所属的现有vma
struct vma* findvma(struct proc* p, uint64 va){
  for(int i = 0; i < NVMA; i++){
    struct vma* vv = &p->vmas[i];
    // 如果va地址在某一个vma范围内，则返回这个vma
    if(vv->valid == 1 && va >= vv->vastart && va < vv->vastart + vv->sz){
      return vv;
    }
  }
  return 0;
}

// 给虚拟地址分配物理页并建立映射
int vmaalloc(uint64 va){
  struct proc* p = myproc();
  struct vma* v = findvma(p, va);
  if(v == 0){
    return 0;
  }

  // 分配物理页
  void *pa = kalloc();
  if(pa == 0){
    panic("vmaalloc: kalloc");
  }
  memset(pa, 0, PGSIZE);

  // 从磁盘中读取文件的那一页
  begin_op();
  ilock(v->f->ip);
  readi(v->f->ip, 0, (uint64)pa, v->offset + PGROUNDDOWN(va - v->vastart), PGSIZE);
  iunlock(v->f->ip);
  end_op();

  // 建立映射
  if(mappages(p->pagetable, va, PGSIZE, (uint64)pa, PTE_R | PTE_W |PTE_U) < 0){
    panic("vmaalloc: mappages");
  }
  return 1;
}
```

* 实现`findvma`函数
  * 遍历vmas数组，如果要查找的va地址在某一个vma范围内，则返回该vma。
* 实现`vmaalloc`函数
  * 通过`myproc()`找到当前的进程。
  * 通过`findvma`函数找到需要映射的虚拟地址空间。
  * 调用`kalloc()`函数，分配物理页。
  * 从磁盘中读取文件的内容
    * `v->offset`：表示文件从哪个字节开始映射到内存。
    * `PGROUNDDOWN(va - v->vastart)`：确定用户访问的地址va在映射区域内页对齐偏移。
    * 两者相加，就能得到文件在该页的实际位置。
  * 通过`mappages`函数，将分配的物理页和虚拟地址形成映射。

> 缺页中断时，调用`vmaalloc`函数

```c++
// kernel/trap.c

void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call

    if(p->killed)
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sstatus &c registers,
    // so don't enable until done with those registers.
    intr_on();

    syscall();
  } else if((which_dev = devintr()) != 0){
    // ok
  } else if(r_scause() == 13 || r_scause() == 15) {
    uint64 va = r_stval();  // 读取当前发生页面错误的地址
    if(vmaalloc(va) == 0){
      panic("usertrap: wrong va");
    }
  } else {
 	// ....................
}
```

至此，我们就实现了mmap的系统调用，但是距离实验结束还有一段距离，我们还需要实现munmap的系统调用，释放所有的vma。



### 实现munmap的系统调用

> `sys_munmap(void)`：释放所有的VMA

1. 读取需要释放映射的地址addr和释放地址的范围大小sz。
2. 通过`findvma`函数找到对应的VMA。
3. 防止"打洞"检查。（XV6是允许从区域的起始位置、区域的结束位置、整个区域进行释放，但是不可以在VMA的中间"打洞"）
4. 地址对齐处理 + 计算实际释放的字节数
   1. 地址对齐处理：保证释放的起始地址从页大小的整数倍的地方开始。
   2. 计算实际释放的字节数时，虽然`sz - (addr_alinged - addr)`，这会导致实际释放大小减少，但是能够保证不跨页部分释放。
5. 执行`vmaunmap`函数来解除映射。
6. 检查当前释放的区域是否覆盖了VMA的起始部分（若条件成立的时候）
   1. 文件偏移量offset指的是文件被映射部分的起始偏移量，所以此时需要将文件偏移量offset往后增加"解除了映射的部分"。
   2. vastart指的是VMA的起始地址，此时需要将VMA的起始地址往后移动，跳过已经释放的区域。
   3. 最后还需要将VMA的范围大小sz缩小。
7. 完全释放VMA的时候，需要关闭关联的文件以及更改VMA标记为无效

```c++
// kernel/sysfile.c

// 释放vma映射的页
uint64 sys_munmap(void){
  uint64 addr, sz;
  
  if(argaddr(0, &addr) < 0 || argaddr(1, &sz) < 0 || sz == 0){
    return -1;
  }

  struct proc* p = myproc();
  struct vma* v = findvma(p, addr);
  if(v == 0){
    return -1;
  }

  // 禁止在中间打洞
  if(addr > v->vastart && addr + sz < v->vastart + v->sz){
    return -1;
  }

  // 确保释放的起始地址addr_alinged是页大小的整数倍
  uint64 addr_alinged = addr;
  if(addr > v->vastart){
    addr_alinged = PGROUNDUP(addr);
  }
  
  int nunmap = sz - (addr_alinged - addr);  // 计算要释放的字节数
  
  if(nunmap < 0){
    nunmap = 0;
  }

  vmaunmap(p->pagetable, addr_alinged, nunmap, v);

  if(addr <= v->vastart && addr + sz > v->vastart){
    v->offset += addr + sz - v->vastart;
    v->vastart = addr + sz;
  }
  v->sz -= sz;

  if(v->sz <= 0){
    fileclose(v->f);
    v->valid = 0;
  }
  return 0;
}
```

在`sys_munmap(void)`中，我们用到了`vmaunmap`函数，用于释放mmap映射的页，并且需要根据PTE_D和MAP_SHARED判断是否应该将修改写回磁盘。

> `vmaunmap`函数的实现

在实现`vmaunmap`函数之间，我们需要添加`PTE_D`标志位，表示页面是否被修改过。

```c++
// kernel/riscv.h

#define PTE_V (1L << 0) // valid
#define PTE_R (1L << 1)
#define PTE_W (1L << 2)
#define PTE_X (1L << 3)
#define PTE_U (1L << 4) // 1 -> user can access
#define PTE_G (1L << 5)
#define PTE_A (1L << 6)
#define PTE_D (1L << 7) // 页表被修改过
```

接下来，就可以实现`vmaunmap`函数，这个函数的作用是释放mmap映射的页，并且需要根据PTE_D和MAP_SHARED判断是否应该将修改写回磁盘。

1. 添加必要的头文件。
2. 逐页处理 `[va, va + nbytes)` 范围内的虚拟内存。
3. 查找页表项PTE，并且检查PTE是否有效。
4. 如果页表项PTE有效
   1. 获取物理地址。
   2. 该页被修改过且共享映射的话，需要将脏页写回文件。
   3. 如何写回文件呢？
      1. 当aoff < 0 的时候，即页的起始地址低于VMA的起始地址，则只回写页中属于VMA的部分到文件中。
      2. 当aoff + PGSIZE > v->sz的时候，即页的结束地址超过了VMA的结束地址，则只回写页中属于VMA的部分到文件中。
      3. 其他情况，整页都在VMA内，回写完整的一页。
   4. 释放物理内存（一页） + 清空页表项

```c++
// kernel/vm.c

// 添加必要的头文件
#include "fcntl.h"
#include "spinlock.h"
#include "sleeplock.h"
#include "file.h"
#include "proc.h"

// 释放mmap映射的页
void vmaunmap(pagetable_t pagetable, uint64 va, uint64 nbytes, struct vma* v){
  uint64 a;
  pte_t* pte;

  for(a = va; a < va + nbytes; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0){
      continue;
    }

    if(PTE_FLAGS(*pte) == PTE_V){
      panic("sys_munmap: not a leaf");
    }

    if(*pte & PTE_V){
      uint64 pa = PTE2PA(*pte);
      if((*pte & PTE_D) && (v->flags & MAP_SHARED)){
        begin_op();
        ilock(v->f->ip);
        uint64 aoff = a - v->vastart;
        if(aoff < 0){
          writei(v->f->ip, 0, pa + (-aoff), v->offset, PGSIZE + aoff);
        }else if(aoff + PGSIZE > v->sz){
          writei(v->f->ip, 0, pa, v->offset + aoff, v->sz - aoff);
        }else{
          writei(v->f->ip, 0, pa, v->offset + aoff, PGSIZE);
        }
        iunlock(v->f->ip);
        end_op();
      }
      kfree((void*)pa);
      *pte = 0;
    }
  }
}
```

> 在proc.c中添加堆vma的处理

1. 初始化进程的时候，需要初始化一个进程的vmas数组

```c++
// kernel/proc.c

static struct proc*
allocproc(void)
{
  // ........

  // 初始化时清空vmas数组
  for(int i = 0; i < NVMA; i++){
    p->vmas[i].valid = 0;
  }
  
  return p;
}
```

2. 释放进程的时候，需要释放页表前清空vmas数组

```c++
// kernel/pro.c

static void
freeproc(struct proc *p)
{
  // .......

  // 释放页表前把vmas数组清空
  for(int i = 0; i < NVMA; i++){
    struct vma* v = &p->vmas[i];
    vmaunmap(p->pagetable, v->vastart, v->sz, v);
  }

  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  // ........
}
```

3. fork创建子进程的时，子进程复制父进程的vmas数组，不复制物理页（COW）

```c++
// kernel/pro.c

int
fork(void)
{
  // ..........

  // 父进程vmas复制到子进程中，实际内存页和pte不会被复制
  // 写时复制
  for(int i = 0; i < NVMA; i++){
    struct vma* v = &p->vmas[i];
    if(v->valid){
      np->vmas[i] = *v;
      filedup(v->f);
    }
  }


  safestrcpy(np->name, p->name, sizeof(p->name));

  pid = np->pid;

  np->state = RUNNABLE;

  release(&np->lock);

  return pid;
}
```

到这里为止，就完成了`munmap`的系统调用的实现。

可是要想结束这个实验还要完成最后一步，那就是添加上`mmap`和`munmap`两个系统调用。



### 添加`mmap`和`munmap`两个系统调用

1. 在kernel/defs.h中

```c++
struct buf;
struct context;
struct file;
struct inode;
struct pipe;
struct proc;
struct spinlock;
struct sleeplock;
struct stat;
struct superblock;
struct vma;			// 添加数据结构vma

// sysfile.c
int             vmaalloc(uint64 va);

// fs.c
int             copyinstr(pagetable_t, char *, uint64, uint64);
void            vmaunmap(pagetable_t pagetable, uint64 va, uint64 nbytes, struct vma* v);
```

2. 在kernel/syscall.c中

```c++
extern uint64 sys_mmap(void);
extern uint64 sys_munmap(void);

static uint64 (*syscalls[])(void) = {
[SYS_fork]    sys_fork,
[SYS_exit]    sys_exit,
[SYS_wait]    sys_wait,
[SYS_pipe]    sys_pipe,
[SYS_read]    sys_read,
[SYS_kill]    sys_kill,
[SYS_exec]    sys_exec,
[SYS_fstat]   sys_fstat,
[SYS_chdir]   sys_chdir,
[SYS_dup]     sys_dup,
[SYS_getpid]  sys_getpid,
[SYS_sbrk]    sys_sbrk,
[SYS_sleep]   sys_sleep,
[SYS_uptime]  sys_uptime,
[SYS_open]    sys_open,
[SYS_write]   sys_write,
[SYS_mknod]   sys_mknod,
[SYS_unlink]  sys_unlink,
[SYS_link]    sys_link,
[SYS_mkdir]   sys_mkdir,
[SYS_close]   sys_close,
[SYS_mmap]    sys_mmap,		// add
[SYS_munmap]  sys_munmap,	// add
};
```

3. 在kernel/syscall.h中

```c++
#define SYS_mmap   22
#define SYS_munmap 23
```

4. 在user/user.h中

```c++
void* mmap(void* addr, uint sz, int prot, int flag, int fd, uint offset);
int munmap(void* addr, uint sz);
```

5. 在user/usys.pl中

```c++
entry("mmap");
entry("munmap");
```

6. 修改Makefile

```c++
$U/_wc\
$U/_zombie\
$U/_mmaptest\
```



## 什么是内存映射

核心定义：将**虚拟地址空间**和**物理资源**建立映射关系的机制。

> 文件映射 vs. 用户进程与内核的地址映射

* 文件映射
  * 作用：将文件的内容直接映射到进程的虚拟地址空间中。
  * 实现：
    * 进程访问文件时，内核通过内存映射将文件的一部分（如`mmap`）关联到虚拟地址。
    * 访问该地址时，若数据不在物理内存中，触发缺页异常，内核从磁盘加载文件内容到内存（按需加载）。
* 用户进程与内核的地址映射
  * 作用：内核需要访问用户进程的地址空间。
  * 实现：
    * 内核通过内存映射机制，将**用户空间的虚拟地址**映射到**内核的虚拟地址空间。
    * 用户进程不能直接要求内核与自己进行“文件映射”，但用户进程可以通过系统调用（`mmap`）间接请求内核将文件映射到自己的用户空间。

> 为什么上述两种操作都说使用到内存映射机制？

内存映射就是将某种资源（文件、物理内存等）映射到进程的虚拟地址空间中。

内存映射的核心在于：通过页表将虚拟地址与物理资源关联。

* 虚拟地址空间：进程看到的地址是虚拟的，需要通过页表转换为物理地址。
* 映射的多样性：被映射的资源可以是物理内存、文件、设备内存，甚至内核数据结构。

其实质是都依赖CPU的MMU和页表机制。

* 文件映射：将磁盘资源映射到虚拟地址。
* 用户-内核映射：将用户物理页（内存）映射到内核虚拟地址中。

两者最终都是通过修改页表完成映射，只是资源来源不同（文件和物理内存）。

### 零拷贝技术

在介绍完内存映射机制之后，我们来看零拷贝技术究竟是什么？

>  在传统的读写（read/write）流程下

![image-20250618210218214](C:\Users\lione\Documents\Notes\CppNote\WilesNote\MIT_OS_6.S081\assets\image-20250618210218214.png)

1. 磁盘文件通过DMA拷贝，将文件内容复制到内核态的缓冲区中。
2. 内核缓冲区通过CPU拷贝，将内核缓冲区的内容复制到用户缓冲区中。
3. 用户缓冲区通过CPU拷贝，将内核缓冲区的内容复制到socket缓冲区中。
4. socket缓冲区通过DMA拷贝，将socket缓冲区的内容复制到网卡中。

CPU在用户态的时候，发生`read`系统调用，从用户态变成了内核态，然后在内核态下，实现了从内核缓冲区读取数据到用户缓冲区，读取完成后从内核态切换回用户态；同理`write`系统调用也是一样的，所以发生了4次的上下文切换。

整个流程中，一共发生了4次用户态到内核态的上下文切换，同时还发生了4次数据拷贝。

若想要提高文件传输的性能，就需要减少**用户态与内核态的上下文切换**和**内存拷贝**的次数。

> 如何减少用户态与内核态的上下文切换呢？

我们会发生用户态与内核态的上下文切换，实际上是源于用户态没有对磁盘和网卡等物理资源的操作权限，此时此刻就需要引入内存映射机制。

> 如何减少内存拷贝的次数？

因为此处要完成的功能是文件传输，那就意味在用户态的时候，我们是不需要对数据进行加工的，所以实际上用户态的缓冲区是不需要存在的。

![image-20250618210117789](C:\Users\lione\Documents\Notes\CppNote\WilesNote\MIT_OS_6.S081\assets\image-20250618210117789.png)

> 引入了内存映射机制之后，我们将使用系统调用函数`mmap()`代替`read()`

`mmap()`系统调用函数会直接把内核缓冲区的数据“映射”到用户空间，这样就省去了内核态和用户态之间的数据拷贝操作。

具体的流程：

1. 磁盘文件通过DMA拷贝将数据拷贝到内核的缓冲区中，以后应用进程和操作系统内核都“共享”这个来自于内核区的缓冲区。
2. 应用进程通过调用`write()`，将共享缓冲区的数据拷贝到socket缓冲区中，而这一切都发生在内核态，由CPU来搬运数据。
3. 内核态的socket缓冲区里的数据通过DMA拷贝，拷贝到网卡里面。

引入了`mmap()`代替`read()`之后，能节省一次数据拷贝过程，数据拷贝次数是3次，而需要4次上下文切换。

> 在Linux内核版本2.1中，引用了`sendfile`系统调用函数

`sendfile`系统调用函数替代了前面的`read`和`write`两个系统调用函数。此时可以减少一次系统调用，意味着减少了2次上下文切换的开销。

![image-20250618213952779](C:\Users\lione\Documents\Notes\CppNote\WilesNote\MIT_OS_6.S081\assets\image-20250618213952779.png)

此时此刻，只有2次上下文切换和3次数据拷贝。

> 在Linux内核2.4版本中，`sendfile`系统调用函数发生了一点微弱的变化

![image-20250618214217539](C:\Users\lione\Documents\Notes\CppNote\WilesNote\MIT_OS_6.S081\assets\image-20250618214217539.png)

具体的流程：

* 磁盘文件通过DMA拷贝，将磁盘上的数据拷贝到内核缓冲区中。
* 文件描述符和数据长度传输到socket缓冲区中，网卡中的SG—DMA就可以直接将内核缓冲区中的数据拷贝到网卡中，节省了内核缓冲区拷贝数据到socket缓冲区的过程。

此时此刻，只有2次数据拷贝和2次上下文切换。

因此，零拷贝技术指的是我们没有通过CPU来搬运数据，所有的数据都是通过DMA来进行传输的。
