# 进程 

## 为什么会有进程出现 
&emsp;&emsp;早期的计算机的处理器处理能力非常有限，一次只能运行一个程序。但是就三级的输入和输出却非常慢。当计算机正在输入程序的时候，处理器就处于“空闲”状态。而计算机的成本是十分昂贵的，为了让处理器一直都在工作，当前执行的程序就被挂起，切换到另外一个程序。**进程的概念就此诞生**。

## 什么是进程 
经典定义：一个执行中的程序的实例。应用程序的关键抽象是：<br/>
1.一个独立的逻辑控制流，它提供一个假象，好像我们的应用程序独占地使用处理器。<br/>
2.一个私有的地址空间，它提供一个假象，好像我们的程序独占地使用存储器系统。

## 进程到底占有哪些资源呢　
&emsp;&emsp;进程不仅仅局限于一段可执行程序代码。通常进程还要包含其他资源。像有打开的文件，挂起的信号，内核内部数据，处理器状态，一个或多个具有内存映射的内存地址空间及一个或多个执行线程，当然还包括用来存放全局变量的数据段等等。

## linux用一个task_struct结构体来描述进程
linux中task_struct定义在/include/linux/sched.h头文件中。
task_struct部分成员如下所示
```
struct task_struct {
	volatile long state;	/* -1 unrunnable, 0 runnable, >0 stopped */
	void *stack;
	atomic_t usage;
	unsigned int flags;	/* per process flags, defined below */
	unsigned int ptrace;

	int lock_depth;
#ifdef CONFIG_SMP
#ifdef __ARCH_WANT_UNLOCKED_CTXSW
	int oncpu;
#endif
#endif

	int prio, static_prio, normal_prio;
	unsigned int rt_priority;
	const struct sched_class *sched_class;
	struct sched_entity se;
	struct sched_rt_entity rt;
```
**重要成员说明**
### 进程ID
`pid_t pid;`<br/>
每个进程都有一个唯一的数字标示符，称为进程ID，它总是非负整数。为了与老版本的unix和linux兼容，pid的最大值默认设置为32768。
用ps，top，htop等命令可以看到操作系统当前的进程列表。
### 进程状态
`volatile long state;`<br/>
进程描述符中的state字段描述了进程当前所处的状态。
**1.可运行状态（TASK_RUNNING）**<br/>
 &emsp;进程要么在CPU上执行，要么准备执行。<br/>
**2.可中断的等待状态（TASK_INTERRUPTIBLE）**
     &emsp;进程被挂起（睡眠），直到某个条件变为真。产生一个硬件中断，释放进程正等待的系统资源，或传递一个信号都是可以唤醒进程的条件（把进程的状态放回到TASK_RUNNING）<br/>
**3.不可中断的等待状态（TASK_UNINTERRUPTIBLE）**
      &emsp;与可中断的等待状态类似，但有一个例外，把信号传递到睡眠进程不能改变它的状态。这种状态很少用到，但在一些特定的情况下（进程必须等待，直到一个不能被中断的事件发生），这种状态是很有用的。例如，当进程打开一个设备文件，其相应的设备驱动程序开始探测相应的硬件设备时会用到这种状态。探测完成以前，设备驱动程序不能被中断，否则，硬件设备会处于不可预知的状态。<br/>
**4.暂停状态（TASK_STOPPED）**
     &emsp; 进程的执行被暂停。当进程收到SIGSTOP、SIGTOP、SIGTIN、SIGTTOU信号后，进入暂停状态。<br/>
**5.跟踪状态（TASK_TRACED）**
     &emsp; 进程的执行已由debugger程序暂停。当一个进程被另一个进程监控时（strace命令，例如debugger执行ptrace()系统调用监控一个测试程序），任何信号都可以把这个进程置于TASK_TRACED状态。<br/>
## 创建一个进程
```
调用一次返回两次，父进程中返回子进程的pid，子进程返回0
#include <unistd.h>
pid_t fork(void);
```
fork()函数为创建一个子进程的系统调用，底层调用clone()系统调用，clone()调用do_fork()，do_fork()调用copy_process()。<br/>
下面介绍copy_process函数所做的工作。<br/>
1.调用dup_task_struct()为新进程创建一个内核栈、thread_info结构和task_struct，这些值与当前进程的值相同。此时，子进程和父进程的描述符是完全相同的。<br/>
2.检查并确保新创建这个子进程后，当前用户所拥有的进程数目没有超出给它分配的资源的限制。<br/>
3.子进程着手使自己与父进程区别开来。进程描述符内的许多成员都要被清0或设为初始值。那些不是继承而来的进程描述符成员，主要是统计信息。task_struct中的大多数数据都依然未被修改。<br/>
4.子进程的状态被设置为TASK_UNINTERRUPTIBLE，以保证它不会投入运行。<br/>
5.copy_process()调用copy_flags()以更新task_struct的flags成员。表明进程是否拥有超级用户权限的PF_SUPERPRIV标志被清0。表明进程还没有调用exec()函数的PF_FORKNOEXEC标志被设置。<br/>
6.调用alloc_pid()为新进程分配一个有效的PID。<br/>
7.根据传递给clone()的参数标志，copy_process()拷贝或共享打开的文件、文件系统信息、信号处理函数、进程地址空间等。在一般情况下，这些资源会被给定进程的所有线程共享；否则，这些资源对每个进程是不同的，因此被拷贝到这里。<br/>
- 8.copy_process()做扫尾工作并返回一个指向子进程的指针。

**写时拷贝**
&emsp;fork()使用写时拷贝（copy-on-write）页实现。写时拷贝是一种可以推迟甚至免除拷贝数据的技术。内核此时并不复制整个进程地址空间，而是让父进程和子进程共享同一个拷贝。<br/>
&emsp;只有在需要写入的时候，数据才会被复制，从而使各个进程拥有各自的拷贝。也就是说，资源的复制只有在需要写入的时候才进行，在此之前，只是以只读方式共享。

**创建一个进程实例**
```
#include <stdio.h>
#include <unistd.h>

int main(int argc, char* argv[])
{
  pid_t pid;

  pid = fork();
  if(pid > 0)
  {
    printf("(father pid)=%d\n", getpid());
  }
  else if(pid == 0)
  {
    printf("(child pid)=%d\n", getpid());
  }
  return 0;
}
```
## 进程的终止
8种方式终止进程
**5种正常的**
- 1.从main返回
- 2.调用exit
- 3.调用_exit或_exit
- 4.最后一个线程从其启动例程返回
- 5.从最后一个线程调用pthread_exit
**3种异常终止**
- 1.调用abort
- 2.接到一个信号
- 3.最后一个线程对取消请求做出响应
## 专用进程
 * ID为0的进程通常是调度进程，该进程是内核的一部分。常常被称为交换进程（swapper）
* ID为1的进程init进程，它是Unix和类Unix系统中用来产生其它所有进程的程序。它以守护进程的方式存在。
