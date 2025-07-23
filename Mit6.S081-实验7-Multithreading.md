# Mit6.S081-实验7-Multithreading

开启实验之前，需要切换本次实验分支：

```c++
git checkout thread
```

## 基础概念

> 进程和线程的区别

* 基本概念
  * 进程：操作系统进行资源分配和调度的基本单位，每个进程拥有独立的地址空间、文件描述符、堆栈等资源。
  * 线程：进程的执行单元，多个线程可以共享同一个进程的地址空间和其他资源，包括堆、全局变量、打开的文件等。
* 调度和切换
  * 进程之间的切换需要保存和恢复完整的上下文信息，而线程之间的切换只需要保存和恢复部分上下文信息。
* 并发性
  * 进程之间的并发性是依赖于操作系统的时间片轮转调度实现的，进程之间的通信需要使用进程间通信机制（如管道、消息队列、共享内存等）。
  * 同一进程内的多个线程可以并发执行，线程之间可以直接共享同一进程的地址空间和其他资源，从而实现统一进程的线程间的通信。
* 资源开销
  * 创建和销毁进程的开销比较大，因为需要分配和释放独立的地址空间、文件描述符等资源。
  * 创建和销毁线程的开销相对较小，因为线程共享了进程的地址空间和其他资源，只需要分配和释放线程的栈空间、线程控制块等少量资源。

> 用户级线程和内核级线程

* 用户级线程
  * 由用户空间的线程库管理，内核无感知，调度由应用程序控制。
* 内核级线程
  * 由操作系统内核直接管理，线程是内核调度的基本单位。

![image-20250701090944210](./assets\image-20250701090944210.png)

> 进程间通信的方式

* 管道（`pipe`）：半双工通信方式，其设计依赖于文件描述符的继承（如`fork()`）用于在父进程和子进程之间的通信。
* 命名管道（`mkfifo`）：命名管道是一种特殊的管道，它允许不相关的进程之间进行通信。与普通管道不同，命名管道在文件系统中有一个**路径名**，并且可以在多个进程之间共享数据。
* 消息队列：消息队列本质上是数据结构中队列的一种实现。
  * 单工：一方仅发送，另一方仅接收。
  * 半双工：双队列的模式。
    * 进程A通过 `queue_AtoB` 发送消息，进程B从该队列接收。
    * 进程B通过 `queue_BtoA` 回复消息，进程A从该队列接收。
  * 全双工：双方可同时发送和接收（Linux默认不支持）
    * 双消息队列的组合
    * 消息队列（通知对方数据已写入共享内存）+共享内存（用于双向数据交换）
* 信号量：用于进程间同步与互斥的核心机制之一，主要用于解决多进程/多线程的`race-condition`和数据不一致的问题。
  * 二值信号量：用于互斥锁（`mutex`）
  * 计数信号量：用于控制有限资源的访问
* 套接字：不仅支持进程间通信，也支持网络通信
  * 服务端
    * `socket()`：创建套接字
    * `bind()`：绑定IP和端口
    * `listen()`：监听连接
    * `accept()`：接受连接
    * `close()`：关闭套接字
  * 客户端
    * `socket()`：创建套接字
    * `connect()`：连接服务端
    * `send()`：发送数据
    * `recv()`：接收数据
    * `close()`：关闭套接字
* 信号：进程间异步通知机制
  * 异步通知
    * 信号由内核或其他进程发送，**强制中断**目标进程的执行流程，跳转到注册的信号处理函数。
  * 不可靠性
    * 同类信号可能丢失（如连续发送多个 `SIGINT` 只能捕获一次）
  * 常见的信号
    * `SIGKILL`（强制终止进程）
    * `SIGTERM`（允许进程清理资源后再退出）
    * `SIGSEGV`（段错误）

> 线程通信的方式

* 共享内存：共享同一块内存区域，通过读写共享内存来进行线程间数据共享和同步，但是需要使用互斥锁、读写锁等同步机制来保证。
* 互斥锁（`mutex`）：其防止多个线程同时访问共享资源导致的`race-condition`和数据不一致问题。线程在访问共享资源之前需要先获取互斥锁，访问完后及时释放互斥锁。
* 条件变量：用于让线程在某个条件不满足时主动挂起，并在条件可能满足时被唤醒。
  * 核心
    * 不直接存储条件值
    * 必须与互斥锁配合使用
  * 条件变量的基本操作
    * 等待（`wait`）
    * 通知（`signal`）
    * 广播（`broadcast`）
* 信号量：信号量是一种计数器，用于多线程中的同步和互斥。当信号量计数大于0的时候，线程可以继续执行，否则线程将被阻塞等待。它可以控制同时访问共享资源的线程数量，防止资源的过度使用和竞争。
* 同步屏障：同步屏障是一种同步机制，用于在多线程编程中实现多个线程的同步点。当所有线程达到屏障点时，它们将会被阻塞，直到所有线程都到达后才能继续执行。

* 消息队列：消息队列是一种线程间通信的方式，用于在多个线程之间发送和接受消息。每个线程都可以通过消息队列发送消息，其他线程可以通过消息队列接受消息。

> 同步 vs. 异步、阻塞 vs. 非阻塞

* 同步：调用者发出一个请求，被调用者进行处理，处理完毕后返回给过给调用者，期间调用者会一直等待。
* 异步：调用者发出一个请求却不等待，而是继续执行其他操作，被调用者处理完后**通知调用者**或**调用者通过回调函数**来处理结果。
* 阻塞：调用者在等待结果时会被挂起，不能执行其他操作。
* 非阻塞：调用者在等待结果时，仍然可以执行其他操作，不被挂起。

> 进程的状态

* 创建态
* 就绪态
* 运行态
* 阻塞态
* 终止态



## 具体实验细节

启动实验前，需要切换到`thread`分支：

```shell
$ git fetch
$ git checkout thread
$ make clean
```

### Uthread: switching between threads (moderate)

在本练习中，你将为用户级线程系统设计context切换机制，然后实现它。
开始前，xv6有两个文件user/uthread.c和user/uthread_switch.S，和一个在Makefile中的规则，来构建一个uthread程序。
uthread.c包含绝大多数用户级线程包，以及3个简单的测试线程代码。
线程包缺少一些代码来创建一个线程和在线程之间切换。

你的工作是提出一个计划：**创建线程、存储/恢复寄存器来进行多线程切换，然后实现该计划。**
当你做完时，make grade应该说你的方案通过uthread test。

一旦你已经结束了，当你在xv6上执行uthread时，你应该看到下面输出（3个线程可能以不同顺序启动）：

<img src="./assets\image-20250722224713022.png" alt="image-20250722224713022" style="zoom:50%;" />

这个输出来自3个测试线程，每个测试线程有一个循环（打印一行然后让出CPU到其他线程）。
基于这点，如果没有context切换代码，你将看不到输出。
你将需要添加代码到user/uthread.c中的thread_creat()和thread_schedule()，user/uthread_switch.S的thread_switch。
目标一是确保当thread_schedule()首次运行一个给定线程时，线程执行传到thread_create()中的函数，在它自己的栈上。
另外目标是确保thread_switch保存切换前线程的寄存器，恢复要切换线程的寄存器，返回到切换后的线程上次离开时的指令。
你将不得不决定哪里存储/恢复寄存器；更改struct thread来保存寄存器是一个好计划。
你将需要在thread_schedule添加thread_switch调用；你能传递任何需要的参数到thread_switch，但目的是从一个线程切换到下个线程。

#### 具体实现

1. 在user/uthread.c中引入头文件。

```c++
#include "kernel/riscv.h"   // RISC-V 架构相关定义
#include "kernel/spinlock.h"       // 自旋锁
#include "kernel/param.h"          // 系统参数
#include "kernel/proc.h"    // 进程控制块和调度相关函数
```

2. 在user/uthread.c中，给struct thread增加成员struct context，用于线程切换时保存/恢复寄存器信息。

```c++
struct thread {
  char       stack[STACK_SIZE]; /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
  struct context threadContext;     // 结构体：用于线程切换时保存/恢复寄存器信息
};
```

3. 在user/uthread.c中，修改thread_switch的函数定义。

```c++
struct thread {
  char       stack[STACK_SIZE]; /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
  struct context threadContext;     // 结构体：用于线程切换时保存/恢复寄存器信息
};
struct thread all_thread[MAX_THREAD];
struct thread *current_thread;
extern void thread_switch(struct context*, struct context*);
```

4. 在user/uthread.c中，修改thread_create函数。

```c++
void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  // YOUR CODE HERE
  t->threadContext.ra = (uint64)func;   // ra保证初次切换到线程时从何处执行
  t->threadContext.sp = (uint64)(t->stack) + STACK_SIZE;    // 修改sp保证栈指针位于栈顶
}
```

`t->threadContext.ra = (uint64)func`：为了让线程第一次被调用时，能跳转到目标函数（`func`），我们**手动将 `ra` 设置为 `func`**，欺骗 CPU 让它以为线程是从 `func`“返回”的，此时此刻仅仅只是准备好了"跳转地址"。当某个线程调用调度器的时候，调度器选择回了这个线程，执行thread_switch，切换回这个线程的上下文，CPU回自动从ra开始执行。

` t->threadContext.sp = (uint64)(t->stack) + STACK_SIZE`：栈是从高地址往低地址生长的，其图示如下：

```markdown
高地址（栈底）: t->stack + STACK_SIZE
...
低地址（栈顶）: t->stack
```

5. 在user/uthread.c中，修改thread_schedule函数，调用thread_switch调用，保存当前线程的上下文，而恢复新线程的上下文。

```c++
void 
thread_schedule(void)
{
	// ...........
    /* YOUR CODE HERE
     * Invoke thread_switch to switch from t to next_thread:
     * thread_switch(??, ??);
     */
    thread_switch(&t->threadContext, &current_thread->threadContext);
  } else
    next_thread = 0;
}
```

6. 修改user/uthread_switch.S，实现保存/恢复寄存器。

```c++
thread_switch:
	/* YOUR CODE HERE */
    
    // 保存当前线程状态
    sd ra, 0(a0)
    sd sp, 8(a0)
    sd s0, 16(a0)
    sd s1, 24(a0)
    sd s2, 32(a0)
    sd s3, 40(a0)
    sd s4, 48(a0)
    sd s5, 56(a0)
    sd s6, 64(a0)
    sd s7, 72(a0)
    sd s8, 80(a0)
    sd s9, 88(a0)
    sd s10, 96(a0)
    sd s11, 104(a0)

    // 恢复目标线程状态
    ld ra, 0(a1)
    ld sp, 8(a1)
    ld s0, 16(a1)
    ld s1, 24(a1)
    ld s2, 32(a1)
    ld s3, 40(a1)
    ld s4, 48(a1)
    ld s5, 56(a1)
    ld s6, 64(a1)
    ld s7, 72(a1)
    ld s8, 80(a1)
    ld s9, 88(a1)
    ld s10, 96(a1)
    ld s11, 104(a1)

	ret    /* return to ra */
```



### Using threads(moderate)

问题发生过程：

1. 线程1执行put操作时，检测到key1不存在，准备执行insert到哈希桶的某个位置
2. 在线程1执行insert前，发生了线程切换，CPU控制权转给线程2
3. 线程2恰好也需要向同一个哈希桶的同一位置插入key2，并完整完成了插入操作
4. 当切换回线程1时，线程1继续执行之前未完成的insert操作
5. 最终结果是线程1的插入覆盖了线程2已经插入的key2，导致数据丢失

因此，我们自然而然想到的办法是在并发写共享数据的地方加锁。

#### 具体实现

1. 在notxv6/ph.c中，声明一个互斥锁。

```c++
pthread_mutex_t lock;   // 定义一个锁
```

2. 在notxv6/ph.c中，main函数中初始化锁。

```c++
pthread_mutex_init(&lock, NULL);  // 初始化锁
```

3. 在notxv6/ph.c中，put函数执行的时候加上锁。

```c++
static 
void put(int key, int value)
{
  pthread_mutex_lock(&lock);

  // is the key already present?
  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    insert(key, value, &table[i], table[i]);
  }
  pthread_mutex_unlock(&lock);
}
```

从上述测试返回的数据可以看到，加锁之后，多线程已经不会再丢失键了，保证了线程安全，但是两个线程的性能比单线程的还要低，这就违背了多线程提升性能的初衷。原因在于上面的代码给整个put函数都加上了锁，每一时刻只有一个线程执行put操作，这将退化成了单线程，同时由于加锁、释放锁、竞争锁都是会有开销的，所以会比单线程性能更低。

为了解决这一问题，多线程提升效率通常的做法就是降低锁的粒度，即减少加锁的范围。其实在这段代码中，不同散列桶的put操作是不会相互影响的，同一时刻操作不同的散列桶是不会导致线程安全问题，因此我们只需要给散列桶加锁，保证不同的线程不会同时操作同一个散列桶就可以降低锁的粒度了。

4. 在notxv6/ph.c中，给每一个散列桶声明一把锁。

```c++
pthread_mutex_t lock[NBUCKET];
```

5. 在notxv6/ph.c中，在main函数中初始化锁

```c++
int
main(int argc, char *argv[])
{
  // ............

  // pthread_mutex_init(&lock, NULL);  // 初始化锁

  // 初始化锁
  for(int i = 0; i < NBUCKET; i++){
      pthread_mutex_init(&lock[i], NULL);
  }

  //
  // first the puts
  //
  // .............
}
```

6. 执行put操作的时候，针对散列桶上锁。

```c++
static 
void put(int key, int value)
{
  int i = key % NBUCKET;
 
  pthread_mutex_lock(&lock[i]);

  // is the key already present?
  // .......
}
```

重新编译测试之后，发现不仅保证了线程安全，同时多线程在性能上也有很大的提升。



### Barrier(moderate)

- **线程调用屏障后的操作流程**

  - 线程调用 `barrier` 后，首先递增屏障状态结构体 `bstate` 中的线程计数器 `nthread`。
  - 判断当前进入屏障的线程数是否达到预设的全局总线程数：
    - **若未达到**：调用 `pthread_cond_wait` 使当前线程进入休眠，等待其他线程到达。
    - 若已达到：
      1. 将 `bstate` 中的轮次计数器 `round` 加1（标记新的一轮开始）。
      2. 重置 `bstate` 的线程计数器 `nthread` 为0。
      3. 通过 `pthread_cond_broadcast` 唤醒所有休眠的线程。

  1. **锁的关键作用**
     - **竞态条件风险**：
       若线程1递增 `nthread` 后未立即休眠，此时线程2到达并满足总线程数条件，会先执行唤醒操作，而之后切换回线程1之后，线程1继续执行休眠，而线程2的唤醒操作已经被线程2消费了，这将导致线程1永远无法唤醒。
     - 解决方案：
       - 在 `nthread` 递增到调用 `pthread_cond_wait` 的**整个过程中必须持有锁**。
       - `pthread_cond_wait` 内部会**自动释放锁**（避免死锁），并在被唤醒后重新获取锁。

#### 具体实现

在在notxv6/barrier中，修改barrier函数

```c++
static void 
barrier()
{
  // YOUR CODE HERE
  //
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  //
  pthread_mutex_lock(&(bstate.barrier_mutex));
  bstate.nthread++;     // 当前到达屏障的线程数+1
  int current_round = bstate.round;     // 保存当前轮次
  if(bstate.nthread < nthread){
    // 非最后一个进程：等待轮次变化
    while(bstate.round == current_round){
          pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
    }
  }else{
      // 最后一个线程：重置状态并进入下一轮
      bstate.nthread = 0;
      bstate.round++;
      pthread_cond_broadcast(&bstate.barrier_cond);
  }
  pthread_mutex_unlock(&(bstate.barrier_mutex));
}
```

值得注意的是：此处有一个段有意思的代码，其作用是防止虚假唤醒。

```c++
while(bstate.round == current_round) {
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
}
```

由于POSIX条件变量（`pthread_cond_t`）允许线程**无明确原因**被唤醒（如系统信号干扰），即使没有其他线程调用 `pthread_cond_broadcast/signal`。

当线程被唤醒时候，会重新检查条件`（bstate.round == current_round）`，倘若真的是最后一个线程的时候，将不会进入`if(bstate.nthread < nthread)`这个逻辑中，从而确保只响应下一轮的唤醒信号。

**至此，这个实验所有的任务都已经完成！**
