#OS Lab6 Report

##练习1:使用 Round Robin 调度算法
####1.请理解并分析sched_calss中各个函数指针的用法，并接合Round Robin 调度算法描ucore的调度执行过程
>sched_class是一个用C模拟的抽象调度类，它采用函数指针的方式使得调度类的结构和具体实现相分离，类结构如下

```
struct sched_class {
    // the name of sched_class
    const char *name;
    // Init the run queue
    void (*init)(struct run_queue *rq);
    // put the proc into runqueue, and this function must be called with rq_lock
    void (*enqueue)(struct run_queue *rq, struct proc_struct *proc);
    // get the proc out runqueue, and this function must be called with rq_lock
    void (*dequeue)(struct run_queue *rq, struct proc_struct *proc);
    // choose the next runnable task
    struct proc_struct *(*pick_next)(struct run_queue *rq);
    // dealer of the time-tick
    void (*proc_tick)(struct run_queue *rq, struct proc_struct *proc);
};
```
init函数主要用于调度类的初始化，需要将一个具体实现的调度类赋值给这个统一的接口，在练习一里这个调度类是Round Robin调度类，此外它还需要完成就绪队列的初始化，以及时间片等一些相关参数的设置工作；enqueue函数负责就绪队列的入队，当一个进程因为时间片用完或者其他原因进入就绪队列；dequeue正好相反；pick_next用来在调度的时候从就绪队列中选出下一个应该执行的进程；proc_tick函数用于调度类对时钟中断的响应，有一些算法会根据进程的执行时间来调整参数；
`对于Round Robin算法，就绪队列是按照FIFO的顺序组织的，enqueue只是简单地将进程加入队列的尾部，每一个进程都维护了一个时间片；RR调度器维护当前runnable进程的有序运行队列。当前进程的时间片用完之后，调度器将当前进程放置到运行队列的尾部(enqueue)，再从其头部取出进程进行调度(pick_next),响应时钟中断的函数proc_tick在每次中断时会将当前进程的时间片减一，如果到0了就会触发调度`
####2.请在实验报告中简要说明如何设计实现”多级反馈队列调度算法“
>设置多个就绪队列，并为各个队列赋予不同的优先级。第一个队列的优先级最高，第二队次之，其余队列优先级依次降低。仅当第1～i-1个队列均为空时，操作系统调度器才会调度第i个队列中的进程运行。赋予各个队列中进程执行时间片的大小也各不相同。在优先级越高的队列中，每个进程的执行时间片就越小或越大。
当一个就绪进程需要链入就绪队列时，操作系统首先将它放入第一队列的末尾，按FCFS的原则排队等待调度。若轮到该进程执行且在一个时间片结束时尚未完成，则操作系统调度器便将该进程转入第二队列的末尾，再同样按先来先服务原则等待调度执行。如此下去，当一个长进程从第一队列降到最后一个队列后，在最后一个队列中，可使用FCFS或RR调度算法来运行处于此队列中的进程。
如果处理机正在第i（i>1）队列中为某进程服务时，又有新进程进入第k（k<i）的队列，则新进程将抢占正在运行进程的处理机，即由调度程序把正在执行进程放回第i队列末尾，重新将处理机分配给处于第k队列的新进程。

---
##练习2：实现 Stride Scheduling 调度算法
####1.设计思路
>首先为每个进程多维护两个数据域stride和priority，stride可以大致描述为进程当前执行的长度，priority是每个进程的优先级；SS调度器的核心思想就是选取stride最小的进程继续执行，每个执行的进程都会在当前的stride上加上一个步进值pass，pass与priority成反比，也就是说优先级越高的进程它执行的机会就越多，初始化过程和RR基本一致（除了在lab6中使用了斜堆而不是简单的队列），enqueue函数根据当前进程的stride把它插入斜堆，使斜堆保持小根堆的性质（根节点最小），proc_tick函数还是简单的将当前进程的时间片减一，若减为0就触发调度，pick_next简单地将小根堆的堆顶取出作为下一个执行的进程。

####2.实现过程  
>首先把RR调度器替换为SS调度器，初始化时加入斜堆的初始化

```
list_init(&(rq->run_list));
rq->lab6_run_pool = NULL;
rq->proc_num = 0;
```
>入队函数enqueue调用斜堆的插入，把当前的进程插入斜堆之中，并对进程池计数

```
rq->lab6_run_pool = skew_heap_insert(rq->lab6_run_pool, &(proc->lab6_run_pool), proc_stride_comp_f);
rq->proc_num++;
```
>出对函数dequeue调用斜堆的删除，把当前进程从斜堆中删除，同样维护进程池计数

```
rq->lab6_run_pool = skew_heap_remove(rq->lab6_run_pool, &(proc->lab6_run_pool), proc_stride_comp_f);
rq->proc_num--;
```
>选择下一个运行的进程时，只要从斜堆的顶取出进程即可（由小根堆的性质保证），然后更新进程的stride值

```
struct proc_struct *next_proc = le2proc(rq->lab6_run_pool, lab6_run_pool);
if (next_proc->lab6_priority == 0 ) next_proc->lab6_stride += BIG_STRIDE;
else{
	next_proc->lab6_stride += BIG_STRIDE/next_proc->lab6_priority;
}
```
>在处理时钟中断时，SS的处理方式与RR是一样的，只是对当前进程的时间片进行修改不再赘述。

---
##与参考答案实现上的区别
####练习二
>没有太大的区别

---
##实验原理（重要部分简析）
###实验中涉及的原理
####1.进程的状态变化
```
1.进程首先在 cpu 初始化或者 sys_fork 的时候被创建，当为该进程分配了一个进程控制块之后，该进程进入 uninit态(在proc.c 中 alloc_proc)。
2.当进程完全完成初始化之后，该进程转为runnable态。
3.当到达调度点时，由调度器 sched_class 根据运行队列rq的内容来判断一个进程是否应该被运行，即把处于runnable态的进程转换成running状态，从而占用CPU执行。
4.running态的进程通过wait等系统调用被阻塞，进入sleeping态。
5.sleeping态的进程被wakeup变成runnable态的进程。
6.running态的进程主动 exit 变成zombie态，然后由其父进程完成对其资源的最后释放，子进程的进程控制块成为unused。
7.所有从runnable态变成其他状态的进程都要出运行队列，反之，被放入某个运行队列中。
```
####2.进程切换过程
```
首先在执行某进程A的用户代码时，出现了一个 trap (例如是一个外设产生的中断)，这个时候就会从进程A的用户态切换到内核态(过程(1))，并且保存好进程A的trapframe；当内核态处理中断时发现需要进行进程切换时，ucore要通过schedule函数选择下一个将占用CPU执行的进程（即进程B），然后会调用proc_run函数，proc_run函数进一步调用switch_to函数，切换到进程B的内核态(过程(2))，继续进程B上一次在内核态的操作，并通过iret指令，最终将执行权转交给进程B的用户空间(过程(3))。

当进程B由于某种原因发生中断之后(过程(4))，会从进程B的用户态切换到内核态，并且保存好进程B的trapframe；当内核态处理中断时发现需要进行进程切换时，即需要切换到进程A，ucore再次切换到进程A(过程(5))，会执行进程A上一次在内核调用schedule (具体还要跟踪到 switch_to 函数)函数返回后的下一行代码，这行代码当然还是在进程A的上一次中断处理流程中。最后当进程A的中断处理完毕的时候，执行权又会反交给进程A的用户代码(过程(6))。这就是在只有两个进程的情况下，进程切换间的大体流程。
```

####3.进程调度的时机
```
正在执行的进程执行完毕，需要选择新的就绪进程执行。
正在执行的进程调用相关系统调用（包括与I/O操作，同步互斥操作等相关的系统调用）导致需等待某事件发生或等待资源可用，从而将白己阻塞起来进入阻塞状态。
正在执行的进程主动调用放弃CPU的系统调用，导致自己的状态为就绪态，且把自己重新放到就绪队列中。
等待事件发生或资源可用的进程等待队列，从而导致进程从阻塞态回到就绪态，并可参与到调度中。
正在执行的进程的时间片已经用完，致自己的状态为就绪态，且把自己重新放到就绪队列中。
在执行完系统调用后准备返回用户进程前的时刻，可调度选择一新用户进程执行
就绪队列中某进程的优先级变得高于当前执行进程的优先级，从而也将引发进程调度。
```