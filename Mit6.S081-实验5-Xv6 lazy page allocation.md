# Mit6.S081-实验5-Xv6 lazy page allocation

开启实验之前，需要切换本次实验分支：

```c++
git checkout lazy
```

## 基础概念

### 页面错误异常

RISC-V有三种不同的页面错误：

* 加载页面错误：加载指令访问无效的虚拟地址时触发
* 存储页面错误：存储指令访问无效的虚拟地址时触发
* 指令页面错误：取指令时访问无效的虚拟地址时触发

这些页面错误信息保存在RISC-V的两个寄存器中：

1. `scause`：指示页面错误的类型（加载页面错误、存储页面错误、指令页面错误）
2. `stval`：保存无法转换的虚拟地址

页面错误可以让地址映射关系变得动态起来。通过页面错误，内核可以更新页表，当发生页面错误的时候，内核需要什么样的信息才能够响应页面错误呢？

* 出错的虚拟地址

  * 作用：记录触发页面错误的 **无效虚拟内存地址**。

  * 硬件行为：当页面错误发生时，RISC-V自动将出错的虚拟地址存入 **STVAL（Supervisor Trap Value）寄存器**。

当一个用户应用程序触发了页面错误的时候，页面错误会使用中断trap机制，将程序运行切换到内核态，同时也会将出错的地址存放在STVAL寄存器中。

* 出错的原因（SCAUSE寄存器）
  * 作用：区分page fault的具体类型，决定内核如何响应。
    * 12：指令执行（如`jump`到未映射地址）
    * **13**：加载操作（如`load`指令读取无效地址）
    * **15**：存储操作（如`store`指令写入无效地址）
  * 内核响应：系统调用ECALL触发trap的时候，SCAUSE值为8，page fault也使用相同的trap机制切换到内核态。
* 触发指令地址
  * 硬件方面
    * RISC-V自动将触发page fault的 **指令地址** 保存到 **SEPC（Supervisor Exception PC）寄存器**，确保异常处理后能返回到原指令。
  * 软件方面
    * XV6内核将SEPC的值备份到当前进程的 **trapframe->epc** 中。

因此，从硬件和XV6的角度来说，当出现了page fault时，有三个极具价值的信息，分别是：

1. 引起page fault的内存地址
2. 引起page fault的原因类型
3. 引起page fault的时候的SEPC当中的值，这表明了page fault在用户程序中导致错误的代码位置，同时操作系统还可以通过PC值判断是否需要重新执行该指令或者终止程序。

### 写时复制（在Lab6中实现）

传统的fork问题：标准fork操作会完整拷贝父进程内存空间给子进程，导致两个独立但内容相同的地址空间。可是这种方法存在一个致命的问题，需要分配大量的内存、耗费时间。如果子进程第一件事情就是执行exec，则会马上丢弃这个地址空间，浪费了刚刚分配的大量的内存以及时间。

COW fork的解决方案：

* 初始阶段父子进程共享同一物理内存
* 共享页面被标记为只读（Read-Only）
* 通过延迟拷贝实现优化：
  - 只有实际发生写入时才进行物理内存复制
  - 避免不必要的内存拷贝

当父进程或者子进程修改页面内容的时候，就会触发页面错误异常，内核捕获异常之后，将会执行如下操作：

1. 为子进程分配一个新的物理内存副本，为父进程分配一个新的物理内存副本。
2. 将新页面的物理地址映射到父进程/子进程中产生页面错误的虚拟地址中，并且更新页表中的权限为可读/写。
3. 返回到引发异常的指令位置，重新执行导致页面错误的写操作。

### 惰性分配

1. 初始请求阶段
   1. 当应用进程通过`sbrk`等系统调用请求扩大内存空间时
   2. 内核仅记录地址空间的扩展范围（调整堆指针）
   3. 新地址范围内的页表项（PTE）被标记为"无效"，暂不分配实际物理内存
2. 实际访问阶段
   1. 当程序首次访问这些被标记的虚拟地址的时候
   2. CPU发现页表项无效，触发页面错误异常
   3. 内核捕获异常之后执行以下操作：
      1. 确认故障地址属于惰性分配区域
      2. 分配新的物理页帧
      3. 建立虚拟地址到物理页的映射
      4. 更新页表项状态为"有效"

3. 执行恢复
   1. 内核返回触发异常的指令重新执行
   2. 此时地址已经有效映射，访问可正常完成


### 页面换出

当进程所需内存超过物理内存容量时，操作系统会启动页面换出机制，将部分不活跃的内存页面写入磁盘交换区，腾出物理内存空间。

被换出的页面会在页表项（PTE）中被标记为"无效"。当进程再次访问这些页面时，会触发页面错误（page fault）异常。

内核捕获到page fault后，首先检查故障地址属性，若确认属于已换出的页面，则执行如下操作：

* 分配新的物理页面
* 从磁盘交换区读回原页面的内容
* 更新页表项（PTE），将其重新标记为"有效"。

最后恢复进程执行。

## 具体实验实现

### Eliminate allocation from sbrk() (easy)

这个实验的目标是 **消除 `sbrk()` 系统调用中的内存分配操作**。

1. 传统的`sbrk()`会直接分配或释放物理内存，而本实验要求改为仅修改进程的地址空间大小，而不实际分配物理内存。
2. 实际内存分配推迟到进程首次访问该内存的时候，才触发缺页异常后再分配。

首先需要做的就是，在kernel/sysproc.c中，改动系统调用`sys_sbrk`，不再调用分配内存的`growproc`，仅仅让进程的`sz`加上新增的内存：

```c++
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  // if(growproc(n) < 0)
  //   return -1;

  struct proc* p = myproc();

  if(n > 0){
    p->sz += n;
  }else if(p->sz + n > 0){  // 当 n < 0 (缩小内存)
    // 请求缩小内存且结果合法，则立即释放物理页
    p->sz = uvmdealloc(p->pagetable, p->sz, p->sz+n);
  }else{
    // 请求缩减到负数（非法请求）
    return -1;
  }

  return addr;
}
```

值得注意的是：当缩小内存的时候，应该立即释放，必须确保释放的内存能被其他进程复用，且原进程无法再访问该区域。

```bash
$ echo hi
usertrap(): unexpected scause 0x000000000000000f pid=3
            sepc=0x00000000000012a4 stval=0x0000000000004008
panic: uvmunmap: not mapped
```

此时可以启动XV6执行一个简单的命令，可以发现会报错`panic: uvmunmap: not mapped`，即内核已经通过`sbrk()`扩展了进程的虚拟地址空间，但是未实际分配物理页，所以当进程首次访问新内存的时候，会触发缺页异常。

### Lazy allocation(moderate)

当进程首次访问新内存的时候，会触发缺页异常，但是内核未能正确处理该异常。所以此时此刻，我们需要修改kernel/trap.c中的`usertrap`函数，处理缺页异常。

从上述的"基础概念"中可以知道，`r_scause`可以获取异常原因，其中13表示`page load fault`，15表示`page write fault`，`stval`表示引发缺页异常的虚拟地址，缺页异常就可以由`(r_scause() == 13 || r_scause() == 15)`表示。

我们需要先判断发生错误的虚拟地址是否位于栈空间之上，进程大小之下，然后再为其分配物理内存并且添加映射：

```c++
// kernel/trap.c

void
usertrap(void)
{
  // .....
  } else if((which_dev = devintr()) != 0){
    // ok
  } else if(r_scause() == 13 || r_scause() == 15){  // 惰性分配导致的缺页异常
    uint64 fault_va = r_stval();  // 获取引发缺页异常的虚拟地址
    char* pa = 0; // 分配的物理地址
    // 判断fault_va是否在进程栈空间之中
    if(PGROUNDUP(p->trapframe->sp) - 1 < fault_va && fault_va < p->sz && (pa = kalloc()) != 0){
      memset(pa, 0, PGSIZE);
      // 建立物理内存映射
      if(mappages(p->pagetable, PGROUNDDOWN(fault_va), PGSIZE, (uint64)pa, PTE_R | PTE_W | PTE_X | PTE_U) != 0){
        printf("lazy alloc: failed to map page\n");
        kfree(pa);
        p->killed = 1;
      }
    }
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

  // .....
}
```

`uvmunmap()` 的作用是 **解除一段虚拟地址空间的页表映射**，并选择性释放对应的物理内存。

惰性分配刚开始并未实际分配内存，解除映射关系的时候应该直接跳过这个检查，不然会导致系统崩溃。

而这个检查指的就是`walk()`以及`PTE_V`检查：惰性分配仅承诺虚拟地址空间，而页表项是物理实现的细节，页表项的生成需要有两个条件，中间页表存在（如L2、L1页表已分配）以及最终页表项（PTE）已映射到物理页，所以当我们尝试`walk()`找到该虚拟地址对应的页表项的时候可能会失败，同时可能因为竞争条件或错误，最终PTE未被有效映射（PTE_V = 0）。

```c++
// kernel/vm.c

void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      // panic("uvmunmap: walk");
      continue;
    if((*pte & PTE_V) == 0)
      // panic("uvmunmap: not mapped");
      continue;
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
```

这个时候我们再次启动XV6，执行`echo hi`，即可正常运行了。

```bash
$ echo hi
hi
```

### Lazytests and Usertests(moderate)

`sbrk()`参数为负的情况在上面已经处理了。

此时需要实现两个函数：`uvmshouldallocate`和`uvmlazyallocate`。

在kernel/vm.c中，实现`uvmshouldallocate`函数：

```c++
// 判断页面是否是之前惰性分配的地址
int uvmshouldallocate(uint64 va){
  pte_t* pte;
  struct proc* p =myproc();
  return va < p->sz   // 确保地址在进程的内存大小范围内
      && PGROUNDDOWN(va) != r_sp()  // 确保地址不在 guard page中
      && (((pte = walk(p->pagetable, va, 0)) == 0) || ((*pte & PTE_V) == 0)); // 确保页表项确实不存在
}
```

我们需要检测三个条件：

1. 地址是否是处于进程的合法范围内
2. 地址不是栈保护页
   1. XV6 在用户栈下方预留了一个未映射的 **Guard Page**，用于检测栈溢出（如无限递归）。
   2. 如果错误地为 Guard Page 分配物理页，栈溢出检测会失效。
3. 页表项不存在或者无效

在kernel/trap.c中的uvmunmap代码中，以前未严格检查Guard Page、未检查页表项状态就直接调用`kalloc()`和`mappages`，若页表项已经存在，则会重复映射到同一个物理页，所以现在需要修改的是，增加Guard Page检查以及先通过`walik()`检查页表项是否存在。

在kernel/vm.c中，实现`uvmlazyallocate`函数：

```c++
// 给惰性分配的页面分配并映射物理地址
void uvmlazyallocate(uint64 va){
  struct proc* p = myproc();
  char* pa = kalloc();  // 分配物理地址
  if(pa == 0){
    printf("lazy alloc: out of memory\n");
    p->killed = 1;
  }else{
    memset(pa, 0, PGSIZE);
    if(mappages(p->pagetable, PGROUNDDOWN(va), PGSIZE, (uint64)pa, PTE_W | PTE_X | PTE_R | PTE_U) != 0){
      // 映射物理地址
      printf("lazy alloc: failed to map page\n");
      kfree(pa);
      p->killed = 1;
    }
  }
}
```

实现了上述两个函数之后，就可以在kernel/trap.c中优化之前的usertrap函数了。

```c++
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
  } else {
    uint64 va = r_stval();  // 获取引发缺页异常的虚拟地址
    if((r_scause() == 13 || r_scause() == 15) && uvmshouldallocate(va)){
      // 缺页异常且发生异常的地址经过惰性分配
      uvmlazyallocate(va);
    }else{
      // 如果不是缺页异常或者是在非惰性分配地址上发生的异常，则抛出错误并杀死进程
      printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
      printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
      p->killed = 1;
    }
  }

  if(p->killed)
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```

在kernel/vm.c中修改uvmunmap函数：

惰性分配刚开始并未实际分配内存，解除映射关系的时候应该直接跳过这个检查，不然会导致系统崩溃。

```c++
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      // panic("uvmunmap: walk");
      continue;
    if((*pte & PTE_V) == 0)
      // panic("uvmunmap: not mapped");
      continue;
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
```

在kernel/vm.c中修改uvmcopy函数：

```c++
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  char *mem;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      // panic("uvmcopy: pte should exist");
      continue;
    if((*pte & PTE_V) == 0)
      // panic("uvmcopy: page not present");
      continue;
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    if((mem = kalloc()) == 0)
      goto err;
    memmove(mem, (char*)pa, PGSIZE);
    if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
      kfree(mem);
      goto err;
    }
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```

在kernel/vm.c中修改copyin函数和copyout函数：

`copyin()`：从用户态复制数据到内核态。

若用户缓冲区是 `sbrk()` 刚扩展的惰性内存（未分配物理页），直接访问会导致 `walkaddr()` 失败。所以需要在 `copyin` 中调用 `uvmshouldallocate` 和 `uvmlazyallocate`，动态分配物理页。

`copyout()`：从内核态复制数据到用户态。

用户程序可能通过 `sbrk()` 扩展了虚拟地址空间（增大 `p->sz`），但尚未实际使用这部分内存（未触发缺页异常分配物理页）。此时 `dstva` 是合法的虚拟地址，但 **没有对应的物理页**。

```c++
int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  uint64 n, va0, pa0;

  if(uvmshouldallocate(srcva)){
    uvmlazyallocate(srcva);
  }

  while(len > 0){
    va0 = PGROUNDDOWN(srcva);
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (srcva - va0);
    if(n > len)
      n = len;
    memmove(dst, (void *)(pa0 + (srcva - va0)), n);

    len -= n;
    dst += n;
    srcva = va0 + PGSIZE;
  }
  return 0;
}
```

```c++
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;

  if(uvmshouldallocate(dstva)){
    uvmlazyallocate(dstva);
  }

  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n);

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}
```

至此，这个实验就完成了！
