# 什么情况下会发生死锁，如何解决死锁？

## 死锁产生的四个必要条件

1. 资源互斥。至少一个资源是被排他性共享的，其他线程/进程必须处于等待状态，直到资源被释放
2. 持有和等待。某些线程持有资源，同时在等待其他线程持有的资源；
3. 不可剥夺。资源只能由持有它的线程释放；
4. 环路等待。A等B，B等C，C等A。

## 死锁解决

1. 死锁预防。破坏导致死锁必要条件之一的任意一个就可以预防死锁。如要求用户申请资源时一次性申请全部资源，即**破坏持有和等待条件**；将资源分层，得到上一层资源后才能申请下一层资源，即**破坏环路等待条件**。**预防通常会降低系统的效率或提升系统的复杂性**。
2. 死锁避免。线程在申请资源时判断操作的安全性。如使用银行家算法。
3. 死锁检测。判断系统是否处于死锁状态(资源分配图有环路则可能发生死锁)。
4. 死锁解除。将某线程持有的资源强制收回，解除环路等待。