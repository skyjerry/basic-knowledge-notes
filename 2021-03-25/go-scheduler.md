# 简述Goroutine调度流程

## 基本流程

1. go func() 创建 G
2. 放入 P 本地队列(或平衡到全局队列)
3. 唤醒或新建 M 执行任务
4. 进入调度循环 schedule
5. 带着M的P竭力获取待执行 G 任务并执行
6. 清理执行现场，重新进入调度循环

## 基本概念

基本上，Go进程内的一切都在以 Goroutine 方式运行，包括 Runtime和 main.main 入口函数。G 结构并非执行体，它仅保存并发任务状态，为任务执行提供所需栈内存空间。

实际执行体是系统线程(M)，它和 P 绑定，以调度循环的方式不停执行 G 并发任务。M 通过修改寄存器，将执行栈指向 G 自带的栈内存，并在此空间内分配栈帧，执行任务函数。当需要中途切换时，只需要将相关执行寄存器值保存回 G 空间即可维持状态，任何 M 都可据此恢复执行。线程仅负责执行，不再持有状态，这是并发任务跨线程调度，实现多路复用的根本所在。

尽管 P/M 构成执行组合体，但 M 数量一般会大于 P 的数量。通常情况下，P 的数量相对恒定，默认与 CPU 核数量相同。而 M 则是由调度器按需创建的，当 M 因陷入系统调用而长时间阻塞时，P 就会被监控线程抢回，去新建或唤醒一个 M 执行其他任务，这样 M 的数量就会增长。

因为 G 初始栈仅有 2 KB，且创建操作只是在用户空间简单的分配对象，远比进入内核态分配线程要简单得多。调度器让多个 M 进入调度循环，不停获取并执行任务，所以我们才能创建成千上万个并发任务。

## local queue

每个 P 维护一个 G 本地队列(runq)，并依次将 G 调度 到 M 中执行。同时，每个 P 会周期性地查看全局队列中是否有 G 待运行并将其调度到 M 中执行，全局队列中的 G 主要来自从系统调用中恢复的 G和某个P产生放不下的G。之所以 P 会周期性地查看全局队列，也是为了防止全局队列中的 G 长时间得不到调度机会被 "饿死"。

## work-stealing

当一个 P 执行完本地所有的 G 之后，会尝试挑选一个受害者 P，从它的 G 队列中窃取一半的 G。当尝试若干次窃取都失败之后，会从全局队列中获取(当前个数/GOMAXPROCS)个 G。

为了保证公平性，从随机位置上的 *P* 开始，而且遍历的顺序也随机化了*(*选择一个小于 *GOMAXPROCS*，且和它互为质数的步长*)*，保证遍历的顺序也随机化了。

光窃取失败时获取是不够的，可能会导致全局队列饥饿。P 的调度算法中还会每个 N 轮调度之后就去全局队列拿一个 G。新建 G 时 P 的本地 G 队列已满(达到256个)的时候会放半数 G 到全局队列去，阻塞的系统调用返回时找不到空闲 P 也会放到全局队列。

```go
runtime.schedule() {
  // only 1/61 of the time, check the global runnable queue for a G.
  // if not found, check the local queue.
  // if not found,
  //     try to steal from other Ps.
  //     if not, check the global runnable queue.
  //     if not found, poll network.
}
```



## 

协程发起系统调用时，对应 M 就可能会被阻塞，此时 M 绑定的 P 的 runq 中的G们将得不到调度，相当于里面所有 G 都被阻塞。

实际上，G 即将进入系统调用时，M 会解绑 P，然后M 和 G 进入阻塞，P 此时状态就是系统调用状态，不能跟其他M绑定，如果短时间内阻塞的 M 唤醒，那么M会优先获取这个P，能获取到就继续绑回去，这样有利于数据局部性。

sysmon线程会定时扫描系统调用超时的P，如果时间超过10ms，就会把P设为空闲，重新调度给其他M继续执行P上面的G。



## spinning thread

线程自旋是相对于线程阻塞而言的，表象就是循环执行一个指定逻辑(调度逻辑，目的是不停地寻找 G)。这样做的问题显而易见，如果 G 迟迟不来，CPU 会白白浪费在这无意义的计算上。但好处也很明显，降低了 M 的上下文切换成本，提高了性能。在两个地方引入自旋：

- 类型1：M 不带 P 的找 P 挂载（一有 P 释放就结合）
- 类型2：M 带 P 的找 G 运行（一有 runable 的 G 就执行）

为了避免过多浪费 *CPU* 资源，自旋的 *M* 最多只允许 GOMAXPROCS个。同时当有类型*1*的自旋 *M* 存在时，类型*2*的自旋 *M* 就不阻塞，阻塞会释放 *P*，一释放 *P* 就马上被类型*1*的自旋 *M* 抢走了，没必要。

在新 G 被创建、M 进入系统调用、M 从空闲被激活这三种状态变化前，调度器会确保至少有一个自旋 M 存在（唤醒或者创建一个 M），除非没有空闲的 P。

- 当新 *G* 创建，如果有可用 *P*，就意味着新 *G* 可以被立即执行，即便不在同一个 *P* 也无妨，所以我们保留一个自旋的 *M*（这时应该不存在类型*1*的自旋只有类型*2*的自旋）就可以保证新 *G* 很快被运行。
- 当 *M* 进入系统调用，意味着 *M* 不知道何时可以醒来，那么 *M* 对应的 *P* 中剩下的 *G* 就得有新的 *M* 来执行，所以我们保留一个自旋的 *M* 来执行剩下的 *G*（这时应该不存在类型*2*的自旋只有类型*1*的自旋）。
- 如果 *M* 从空闲变成活跃，意味着可能一个处于自旋状态的 *M* 进入工作状态了，这时要检查并确保还有一个自旋 *M* 存在，以防还有 *G* 或者还有 *P* 空着的。



## sysmon

sysmon 也叫监控线程，它无需 P 也可以运行，他是一个死循环，每20us~10ms循环一次，循环完一次就 sleep 一会，sleep周期是动态的，主要是避免空转，如果每次循环都没什么需要做的事，那么 sleep 的时间就会加大。

- 释放闲置超过*5*分钟的 *span* 物理内存；
- 如果超过*2*分钟没有垃圾回收，强制执行；
- 将长时间未处理的 *netpoll* 添加到全局队列；
- 向长时间运行的 *G* 任务发出抢占调度；
- 收回因 *syscall* 长时间阻塞的 *P。*

当 P 在 M 上执行时间超过10ms，sysmon 调用 preemptone 将 G 标记为 stackPreempt 。因此需要在某个地方触发检测逻辑，Go 当前是在检查栈是否溢出的地方判定(morestack())，M 会保存当前 G 的上下文，重新进入调度逻辑。



## Network poller

*G* 发起网络 *I/O* 操作也不会导致 *M* 被阻塞*(*仅阻塞*G)*，从而不会导致大量 *M* 被创建出来。将异步 *I/O* 转换为阻塞 *I/O* 的部分称为 *netpoller*。打开或接受连接都被设置为非阻塞模式。如果你试图对其进行 *I/O* 操作，并且文件描述符数据还没有准备好，*G* 会进入 *gopark* 函数，将当前正在执行的 *G* 状态保存起来，然后切换到新的堆栈上执行新的 *G*。

G 恢复执行的时机：

- *sysmon*
- *schedule()*：*M* 找 *G* 的调度函数
- *GC*：*start the world*



调用 *netpoll()* 在某一次调度 *G* 的过程中，处于就绪状态的 *fd* 对应的 *G* 就会被调度回来。

*G* 的 *gopark* 状态：*G* 置为 *waiting* 状态，等待显示*goready* 唤醒，在 *poller* 中用得较多，还有锁、*chan* 等。



## go 1.14 基于信号的真抢占式调度

golang在之前的版本中已经实现了抢占调度，不管是陷入到大量计算还是系统调用，大多可被sysmon扫描到并进行抢占。但有些场景是无法抢占成功的。比如轮询计算 for { i++ } 等，这类操作无法进行newstack、morestack、syscall，所以无法检测stackguard0 = stackpreempt。

go1.14中加入了基于信号的协程调度抢占。首先注册绑定 SIGURG 信号及处理方法runtime.doSigPreempt，sysmon会间隔性检测超时的p，然后发送信号，m收到信号后休眠执行的goroutine并且进行重新调度。

```go
package main

import (
    "runtime"
)

func main() {
    runtime.GOMAXPROCS(1)

    go func() {
        panic("already call")
    }()

    for {
    }
}
```

go1.14是可以执行到panic，而1.13版本一直挂在死循环上。