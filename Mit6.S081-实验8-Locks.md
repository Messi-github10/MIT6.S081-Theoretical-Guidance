# Mit6.S081-实验8-Locks

开启实验之前，需要切换本次实验分支：

```c++
git checkout lock
```

## 具体实验细节

### Memory allocator(moderate)

首先我们通过user/kalloctest中的压力测试一下XV6中的Memory allocator看看发生了什么？

```bash
$ kalloctest
start test1
test1 results:
--- lock kmem/bcache stats
lock: kmem: #fetch-and-add 185630 #acquire() 433016
lock: bcache: #fetch-and-add 0 #acquire() 1242
--- top 5 contended locks:
lock: kmem: #fetch-and-add 185630 #acquire() 433016
lock: proc: #fetch-and-add 90080 #acquire() 679353
lock: proc: #fetch-and-add 72960 #acquire() 679352
lock: proc: #fetch-and-add 37202 #acquire() 679368
lock: proc: #fetch-and-add 31566 #acquire() 679352
tot= 185630
test1 FAIL
start test2
total free number of pages: 32499 (out of 32768)
.....
test2 OK
```

在`lock: kmem: #fetch-and-add 185630 #acquire() 433016`中，发现`keme.lock`在获取的时候，发生了185630次自旋等待，锁被其他CPU持有，当前CPU必须忙等待。

在`lock: kmem: #fetch-and-add 185630 #acquire() 433016`中，发现`kmem.lock`被成功获取了433016次，即kalloc和kfree总共调用433016次。

这说明了平均每次获取锁的时候，大概需要自旋等待0.43次`（185630/433016）`，说明锁的争用率很高。

所以`kmem.lock`是系统的主要瓶颈，需要优化。

1. 先看一下在kernel/kalloc.c中的kalloc的代码发生了什么。

```c++
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem;

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

代码中定义了一个结构体kmem，将将空闲物理页的指针存放到freelist当中的`struct run* freelist`当中，使空闲页形成一个链表。分配物理页就是把freelist从链表移除，释放物理页就是把要释放的页重新连回链表。

分配和释放物理页都是操作共享数据，修改freelist链表，因此为了线程安全，这些操作都加上了锁。这样就导致了同一时刻只能有一个线程申请分配或者释放内存，多线程无法并行执行这些操作，限制了并发操作。

因此，我们可以通过降低锁的粒度，为每一个CPU都声明一个freelist锁，CPU申请分配或释放内存，不会影响到另一个CPU执行相同的操作。

在kernel/kalloc.c中，修改一下kmem结构体和每一个CPU都声明一个freelist锁。

```c++
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem[NCPU]; // 每一个CPU都声明一个freelist锁，多个CPU并发分配物理内存不会相互竞争

char* kmem_lock_names[] = {
  "kmem_cpu_0",
  "kmem_cpu_1",
  "kmem_cpu_2",
  "kmem_cpu_3",
  "kmem_cpu_4",
  "kmem_cpu_5",
  "kmem_cpu_6",
  "kmem_cpu_7",
};

void
kinit()
{
  // initlock(&kmem.lock, "kmem");
  for(int i = 0; i < NCPU; i++){
    initlock(&kmem[i].lock, kmem_lock_names[i]);
  }
  freerange(end, (void*)PHYSTOP);
}
```

```c++
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  push_off(); // 关闭中断

  int cpu = cpuid();  // 获取cpu编号，中断关闭时调用cpuid才是安全的

  // 将释放的页插入当前CPU的freelist中
  acquire(&kmem[cpu].lock);   // 上锁
  r->next = kmem[cpu].freelist;
  kmem[cpu].freelist = r;
  release(&kmem[cpu].lock);   // 释放锁

  pop_off();  // 重新打开中断
}
```

先通过`r->next = kmem[cpu].freelist`将释放的页`r`指向原链表头，再通过`kmem[cpu].freelist = r`将链表头指针更新为刚释放的页`r`。

```c++
void *
kalloc(void)
{
  struct run *r;

  push_off(); // 关闭中断

  int cpu = cpuid();

  acquire(&kmem[cpu].lock);

  // 当前CPU的空闲链表已经为空的情况下，需要从其他CPU的空闲链表中偷取内存页
  if(!kmem[cpu].freelist){
    int steal_left = 64;  // 指定偷取64个内存页
    for(int i = 0; i < NCPU; i++){
      if(i == cpu){
        continue;
      }
      acquire(&kmem[i].lock);
      // 如果在想要偷页的CPU也没有空闲链表，则释放锁跳过
      if(!kmem[i].freelist){
        release(&kmem[i].lock);
        continue;
      }
      struct run* rr = kmem[i].freelist;
      while(rr && steal_left){
        // 循环将kmem[i]的freelist移动到kmem[cpu]中
        kmem[i].freelist = rr->next;  // 从目标CPU链表移除一页
        rr->next = kmem[cpu].freelist;  // 将偷来的页插入当前CPU链表头部
        kmem[cpu].freelist = rr;
        rr = kmem[i].freelist;  // 移动到下一页
        steal_left--; // 减少剩余可偷页数
      }
      release(&kmem[i].lock);

      if(steal_left == 0){
        // 偷到指定页数后退出循环
        break;
      }
    }
  }

  r = kmem[cpu].freelist;
  if(r)
    kmem[cpu].freelist = r->next;
  release(&kmem[cpu].lock);

  pop_off();  // 打开中断

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}

```

执行`make qemu`启动XV6，执行`kalloctest`验证：

```bash
$ kalloctest
start test1
test1 results:
--- lock kmem/bcache stats
lock: kmem_cpu_0: #fetch-and-add 0 #acquire() 119406
lock: kmem_cpu_1: #fetch-and-add 0 #acquire() 165249
lock: kmem_cpu_2: #fetch-and-add 0 #acquire() 148363
lock: bcache: #fetch-and-add 0 #acquire() 334
--- top 5 contended locks:
lock: proc: #fetch-and-add 39320 #acquire() 501665
lock: proc: #fetch-and-add 29435 #acquire() 501570
lock: proc: #fetch-and-add 29230 #acquire() 501570
lock: proc: #fetch-and-add 24110 #acquire() 501572
lock: proc: #fetch-and-add 21695 #acquire() 501585
tot= 0
test1 OK
start test2
total free number of pages: 32499 (out of 32768)
.....
test2 OK
```

可以发现kmem的锁竞争已经得到了改善，通过了测试。

但其实此处还有一个潜在问题：当CPU1在持有自身的锁的时候区偷CPU2的页，此时CPU2也持有自身锁的时候去CPU1中偷页，回造成死锁问题。

不过一个CPU内存页不足的情况很少见，两个CPU内存页不足还互相偷取对方内存页的情况就更少了。



### Buffer cache (hard)

Buffer cache是由多个固定大小的缓存块（Buffer）组成的缓存区（Cache）。

每个缓存块（Buffer）中的引用计数`refcnt`用于记录当前有多少个内核线程正在使用该缓存块，但是同一时间内只能有一个线程持有该缓存块。

我们通过user/bcachetest中的压力测试一下XV6中的Buffer cache看看发生了什么？

从`lock: bcache: #fetch-and-add 47412 #acquire() 65022`中可以发现`bcache.lock`的争用，尝试获取锁时发生 47,412 次自旋等待（其他线程正持有锁）。`47412 / 65022 ≈ 73%`，说明 73% 的锁获取操作需要等待，性能极差。

我们先在kernel/bio.c中看看关于获取缓存块的代码是怎样的？

```c++
// 如果该块已经分配了缓存块，则直接返回
// 如果该块未分配缓存块，从缓存中分配一个空闲块（LRU策略）
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;

  acquire(&bcache.lock);

  // Is the block already cached?
  // 查找是否已经缓存
  for(b = bcache.head.next; b != &bcache.head; b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;  // 增加引用计数
      release(&bcache.lock);  // 释放全局锁
      acquiresleep(&b->lock); // 获取该块的睡眠锁
      return b; // 返回缓存块
    }
  }

  // Not cached.
  // Recycle the least recently used (LRU) unused buffer.
  // 未命中时的缓存替换（LRU）
  // 反向遍历链表​：从尾部开始（LRU 策略，尾部是最久未使用的）
  for(b = bcache.head.prev; b != &bcache.head; b = b->prev){
    if(b->refcnt == 0) {  // 找到未被引用的块
      b->dev = dev;
      b->blockno = blockno;
      b->valid = 0;
      b->refcnt = 1;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }
  panic("bget: no buffers");
}
```

在原本的设计当中，当获取Buffer Cache（缓冲区）时需要全局加锁，根据块号遍历链表查找缓存块（Buffer），如果命中则缓存块（Buffer）的引用计数 + 1，释放锁返回缓存块（Buffer）的指针。如果未命中，则从LRU链表选择引用数为0的缓存块（Buffer）复用。

可是这个设计在多进程/多CPU并发访问的时候，全局锁`bcache.lock`竞争激烈。

Buffer Cache（缓冲区）中的Buffer（缓存块）是会被多个进程、多个CPU共享的，所以无法为每个CPU/线程分配一个专属的块。

改进的方案：哈希分桶

* 哈希表设计
  * 键：（dev、blockno）唯一标识磁盘块
  * 桶：通过哈希函数分散到多个桶中，每个桶中含有若干缓存块（Buffer）
  * 锁粒度：每个哈希桶一个锁（并非全局锁）
* 并发访问的场景
  * 进程A访问块1，进程B访问块2，哈希到不同桶 → **无锁竞争**。
  * 进程A和B同时访问块1（同桶） → **仅竞争该桶锁**。

在kernel/boi.c中，先定义哈希表：

```c++
// 哈希表中的桶号索引，设置质数个桶可以降低哈希冲突的可能性
// 定义哈希表（BUFMAP）中 ​桶（Bucket）的总数量为 13
#define NBUFMAP_BUCKET 13

// 哈希索引
// ​哈希函数​：根据设备号 dev 和块号 blockno 计算目标桶的索引（0 ~ NBUFMAP_BUCKET-1）
#define BUFMAP_HASH(dev, blockno) ((((dev) << 27) | (blockno)) % NBUFMAP_BUCKET)

struct {
  // struct spinlock lock;
  struct buf buf[NBUF];

  // Linked list of all buffers, through prev/next.
  // Sorted by how recently the buffer was used.
  // head.next is most recent, head.prev is least.
  // struct buf head;

  struct buf bufmap[NBUFMAP_BUCKET];  // 哈希表
  struct spinlock bufmap_locks[NBUFMAP_BUCKET]; // 桶锁

} bcache;
```

* `NBUFMAP_BUCKET`：桶的个数
* `BUFMAP_HASH`：哈希函数
* `bcache`：缓存区
  * `struct buf buf[NBUF]`：NBUF表示缓存块的总数，`buf[NBUF]`表示Buffer Cache中所有缓存块的集合
  * `struct buf bufmap[NBUFMAP_BUCKET]`：哈希表
  * `struct spinlock bufmap_locks[NBUFMAP_BUCKET]`：表示每个桶的锁

> 睡眠锁的作用

所有尝试获取**已经被占用的缓存块数据锁**的线程都会被睡眠锁阻塞。

当线程A已经通过 `acquiresleep(&b->lock)` 持有某个缓存块（如 `buf1`）的锁时：

* 线程B若再调用 `acquiresleep(&buf1->lock)`，会**主动睡眠**（让出CPU）。
* 直到线程A调用 `releasesleep(&buf1->lock)`（通常在 `brelse()` 中）唤醒线程B。

设计思路：

![Lab8_Locks.drawio](C:\Users\lione\Documents\Notes\CppNote\WilesNote\MIT_OS_6.S081\assets\Lab8_Locks.drawio.png)

1. 查询时刻的BUF与驱逐时刻的BUF状态不一致

线程A查询时发现某缓存块（`buf`）符合条件，但释放桶锁后，其他线程可能修改该块状态。

解决方案：每一次查询桶都记录该桶的桶号，查询完该桶，如果该桶中查找到新的LRU-BUF就不释放该桶锁，直到给为LRU-BUF设置完dev、blockno、refcnt、valid后再解除所有的锁。

2. #### **环路死锁**

CPU1持有桶1锁并请求桶2锁，CPU2持有桶2锁并请求桶1锁 → 死锁。

解决方案：线程在查询其他桶前，先释放自己的桶锁，避免持有多个锁。

可是这样又会导致一个新问题，因提前释放桶锁，线程A和线程B同时为同一个磁盘块创建不同的缓存块，这就违背了一个磁盘块对应一个缓存块的目的。

解决方案：添加一个新的锁`eviction_lock`（驱逐锁）。释放桶锁之后，加上驱逐锁，然后马上再次检查磁盘块号是否已经分配了缓存块。若还未分配缓存块，则开始执行查询操作，最后把缓存块添加到桶（对应磁盘块）之后才释放驱逐锁。

这样，即使多个线程同时用同一个blockno访问同一个桶（磁盘块），并都检查到磁盘块还未分配缓存块，只会有一个线程能竞争得到驱逐锁，然后去其他桶中拿到缓存块，把缓存块关联自己的桶（磁盘块）后，才释放锁。其他线程则会被驱逐锁阻塞，最终得到驱逐锁之后，会先检查磁盘块是否关联了缓存块，这个时候发现该磁盘块已经关联了缓存块了，直接释放锁返回了。

好处：保证了查找过程中不会出现环路死锁，并且不会出现极端情况下一个块产生多个缓存的情况。

坏处：驱逐锁相当于全局锁，使得原本可并发的遍历在驱逐过程中的并行性降低，并且每一次缓存中无某一个磁盘块的时候，从磁盘中读取，并分配一个空闲的缓存块的时候，都需要多一次的额外的桶遍历开销。

在kernel/buf.h中，修改struct buf：

```c++
struct buf {
  int valid;   // has data been read from disk?
  int disk;    // does disk "own" buf?
  uint dev;
  uint blockno;
  struct sleeplock lock;
  uint refcnt;
  // struct buf *prev; // LRU cache list
  struct buf *next;
  uchar data[BSIZE];
  uint lastuse; // 用于跟踪LRU-buf，记录缓存块buf最后一次被访问的时间戳
};
```

在kernel/bio.c中，进行大刀阔斧的改革！

```c++
// 哈希表中的桶号索引，设置质数个桶可以降低哈希冲突的可能性
// 定义哈希表（BUFMAP）中 ​桶（Bucket）的总数量为 13
#define NBUFMAP_BUCKET 13

// 哈希索引
// ​哈希函数​：根据设备号 dev 和块号 blockno 计算目标桶的索引（0 ~ NBUFMAP_BUCKET-1）
#define BUFMAP_HASH(dev, blockno) ((((dev) << 27) | (blockno)) % NBUFMAP_BUCKET)

struct {
  // struct spinlock lock;
  struct buf buf[NBUF];
  struct spinlock eviction_lock;  // 驱逐锁
  // Linked list of all buffers, through prev/next.
  // Sorted by how recently the buffer was used.
  // head.next is most recent, head.prev is least.
  // struct buf head;

  struct buf bufmap[NBUFMAP_BUCKET];  // 哈希表
  struct spinlock bufmap_locks[NBUFMAP_BUCKET]; // 桶锁

} bcache;

void
binit(void)
{

  // 初始化桶锁
  for(int i = 0; i < NBUFMAP_BUCKET; i++){
    initlock(&bcache.bufmap_locks[i], "bcache_bufmap");
    bcache.bufmap[i].next = 0;
  }

  for(int i = 0; i < NBUF; i++){
    // 初始化缓存区块
    struct buf* b = &bcache.buf[i];
    initsleeplock(&b->lock, "buffer");
    b->lastuse = 0;
    b->refcnt = 0;
    
    // 将所有缓存区块添加到bufmap[0]中
    b->next = bcache.bufmap[0].next;
    bcache.bufmap[0].next = b;
  }

  initlock(&bcache.eviction_lock, "bcache_eviction");

  // struct buf *b;

  // initlock(&bcache.lock, "bcache");

  // // Create linked list of buffers
  // bcache.head.prev = &bcache.head;
  // bcache.head.next = &bcache.head;
  // for(b = bcache.buf; b < bcache.buf+NBUF; b++){
  //   b->next = bcache.head.next;
  //   b->prev = &bcache.head;
  //   initsleeplock(&b->lock, "buffer");
  //   bcache.head.next->prev = b;
  //   bcache.head.next = b;
  // }
}

// Look through buffer cache for block on device dev.
// If not found, allocate a buffer.
// In either case, return locked buffer.
// 如果该块已经分配了缓存块，则直接返回
// 如果该块未分配缓存块，从缓存中分配一个空闲块（LRU策略）
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;

  // 通过哈希函数获取桶号
  uint key = BUFMAP_HASH(dev, blockno);

  acquire(&bcache.bufmap_locks[key]);

  // Is the block already cached?
  // 是否已经给blockno分配了缓存块
  // for(b = bcache.head.next; b != &bcache.head; b = b->next){
  //   if(b->dev == dev && b->blockno == blockno){
  //     b->refcnt++;  // 增加引用计数
  //     release(&bcache.lock);  // 释放全局锁
  //     acquiresleep(&b->lock); // 获取该块的睡眠锁
  //     return b; // 返回缓存块
  //   }
  // }
  
  for(b = bcache.bufmap[key].next; b; b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      release(&bcache.bufmap_locks[key]);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // 

  // Not cached.
  // Recycle the least recently used (LRU) unused buffer.
  // 未命中时的缓存替换（LRU）
  // 反向遍历链表​：从尾部开始（LRU 策略，尾部是最久未使用的）
  // for(b = bcache.head.prev; b != &bcache.head; b = b->prev){
  //   if(b->refcnt == 0) {  // 找到未被引用的块
  //     b->dev = dev;
  //     b->blockno = blockno;
  //     b->valid = 0;
  //     b->refcnt = 1;
  //     release(&bcache.lock);
  //     acquiresleep(&b->lock);
  //     return b;
  //   }
  // }

  // 不在缓存区

  // 为了防止环路死锁，需要先释放当前桶锁
  release(&bcache.bufmap_locks[key]);

  // 为了防止blockno的缓存区块被重复创建，加上驱逐锁
  acquire(&bcache.eviction_lock);

  // 再次检查磁盘块号是否已经分配了缓存块，在释放桶锁->加驱逐锁的间隙可能分配了缓存块
  for(b = bcache.bufmap[key].next; b; b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      acquire(&bcache.bufmap_locks[key]); // 添加引用次数时必须添加桶锁
      b->refcnt++;
      release(&bcache.bufmap_locks[key]);
      release(&bcache.eviction_lock);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // 仍然不在缓存区内
  // 此时只持有驱逐锁，不持有任何桶锁。查询所有桶中的LRU-buf
  struct buf* before_least = 0; // LRU-buf的前一个块
  uint holding_bucket = -1; // 记录当前持有哪个桶锁

  // 循环查询所有桶
  for(int i =0; i < NBUFMAP_BUCKET; i++){
    acquire(&bcache.bufmap_locks[i]); // 获取当前遍历的桶锁

    int newfound = 0; // 是否在当前桶查找到新的LRU-buf

    // 全局LRU块查找算法
    for(b = &bcache.bufmap[i]; b->next; b = b->next){
      if(b->next->refcnt == 0 && (!before_least || b->next->lastuse < before_least->next->lastuse)){
        // 找到更旧的块且该块未被引用，同时还要检查是否是第一个遇到的候选块
        before_least = b;
        newfound = 1;
      }
    }
    if(!newfound){  // 当前桶无候选，则立即释放桶锁
      release(&bcache.bufmap_locks[i]);
    }else{  // 找到了新的LRU-BUF
      if(holding_bucket != -1){ // 如果当前找到的不是第一个LRU-BUF，之前肯定持有某个桶锁，需要释放
        release(&bcache.bufmap_locks[holding_bucket]);
      }
      holding_bucket = i; // 把标记 holding_bucket 更改成当前桶锁编号
    }
  }

  // 如果全局没有找到任何一个LRU-buf，表示没有空闲缓存块
  if(!before_least){
    panic("bget: no buffuers");
  }

  // 将该LRU-BUF摘下来分配出去
  b = before_least->next; // b = LRU-buf

  // LRU-BUF块 b 当前位于桶X，但需要迁移到桶Y
  if(holding_bucket != key){
    // 离开当前桶
    before_least->next = b->next;
    release(&bcache.bufmap_locks[holding_bucket]);

    // 将LRU-buf添加到目标桶
    acquire(&bcache.bufmap_locks[key]);
    b->next = bcache.bufmap[key].next;  // 头插法
    bcache.bufmap[key].next = b;
  }

  // 设置新buf的字段
  b->dev = dev;
  b->blockno = blockno;
  b->refcnt = 1;
  b->valid = 0;

  // 可以释放相关锁了
  release(&bcache.bufmap_locks[key]); // 释放迁移之后的桶锁
  release(&bcache.eviction_lock); // 释放驱逐锁
  acquiresleep(&b->lock); // 获取该块的睡眠锁
  return b;

  // panic("bget: no buffers");
}

// .......

// Release a locked buffer.
// Move to the head of the most-recently-used list.
// 该缓存块已经没有任何一个进程在使用
void
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  // acquire(&bcache.lock);

  // 使用dev和blockno通过哈希函数得到key
  uint key = BUFMAP_HASH(b->dev, b->blockno);

  // 获取桶锁
  acquire(&bcache.bufmap_locks[key]);

  // 减少该缓存块的引用计数
  b->refcnt--;
  if (b->refcnt == 0) {
    // no one is waiting for it.
    // b->next->prev = b->prev;
    // b->prev->next = b->next;
    // b->next = bcache.head.next;
    // b->prev = &bcache.head;
    // bcache.head.next->prev = b;
    // bcache.head.next = b;

    // 更新最后使用时间为当前ticks（时钟滴答数）
    // ticks只增不减，此时 lastuse 记录的是 ​最后一次被释放的时间戳
    // 值越大​ → 表示越晚被释放​ → ​越不适合被回收
    // 后续LRU算法将选择lastuse最小的块回收
    b->lastuse = ticks;
  }
  
  // 释放自旋锁
  release(&bcache.bufmap_locks[key]);
}

// 增加引用计数
void
bpin(struct buf *b) {

  uint key = BUFMAP_HASH(b->dev, b->blockno);

  acquire(&bcache.bufmap_locks[key]);
  b->refcnt++;
  release(&bcache.bufmap_locks[key]);
}

// 减少引用计数
void
bunpin(struct buf *b) {

  uint key = BUFMAP_HASH(b->dev, b->blockno);

  acquire(&bcache.bufmap_locks[key]);
  b->refcnt--;
  release(&bcache.bufmap_locks[key]);
}
```

至此，整个操作系统内核优化实验的第二座大山就完成了！
