# TCP常见的拥塞控制算法

### 1. Reno

Reno被许多教材所介绍，适用于低延时、低带宽的网络环境。它将拥塞控制的过程分为了四个阶段：**慢启动、拥塞避免、快重传和快恢复**，对应的状态如下所示：

![PaMaPb](http://image.skyjerry.com/uPic/PaMaPb.jpg)

- 慢启动：慢启动阶段思路是不要一开始就发送大量的数据，先探测一下网络的拥塞程度，也就是说由小到大逐渐增加拥塞窗口的大小，在没有出现丢包时每收到一个 ACK 就将拥塞窗口大小加一（单位是 MSS，最大单个报文段长度），每轮次发送窗口增加一倍，呈指数增长，若出现丢包，则将拥塞窗口减半，进入拥塞避免阶段；
- 拥塞避免：为了防止网络拥塞，当窗口达到慢启动阈值或出现丢包时，进入拥塞避免阶段，窗口每轮次加一，呈线性增长；
- 快重传：当收到对一个报文的三个重复的 ACK 时，认为这个报文的下一个报文丢失了，进入快重传阶段，要求接收方在收到一个失序的报文段后就立即发出重复确认（为的是使发送方及早知道有报文段没有到达对方，可提高网络吞吐量约20%）而不要等到自己发送数据时捎带确认；
- 快重传完成后进入快恢复阶段，将慢启动阈值修改为当前拥塞窗口值的一半，同时拥塞窗口值等于慢启动阈值，然后进入拥塞避免阶段，重复上述过程。



### 2. BBR

> BBR 是谷歌在 2016 年提出的一种新的拥塞控制算法，已经在 Youtube 服务器和谷歌跨数据中心广域网上部署，据 Youtube 官方数据称，部署 BBR 后，在全球范围内访问 Youtube 的延迟降低了 53%，在时延较高的发展中国家，延迟降低了 80%。

BBR 算法不将出现丢包或时延增加作为拥塞的信号，而是认为当网络上的数据包总量大于瓶颈链路带宽和时延的乘积时才出现了拥塞，所以 BBR 也称为基于拥塞的拥塞控制算法（Congestion-Based Congestion Control），其适用网络为高带宽、高时延、有一定丢包率的长肥网络，可以有效降低传输时延，并保证较高的吞吐量，与其他两个常见算法发包速率对比如下：

![P4zPEN](http://image.skyjerry.com/uPic/P4zPEN.jpg)

BBR 算法周期性地探测网络的容量，交替测量一段时间内的带宽极大值和时延极小值，将其乘积作为作为拥塞窗口大小，使得拥塞窗口始的值始终与网络的容量保持一致。

所以 BBR 算法解决了两个比较主要的问题：

- 在有一定丢包率的网络链路上充分利用带宽。
  适合高延迟、高带宽的网络链路。
- 降低网络链路上的 buffer 占用率，从而降低延迟。
  适合慢速接入网络的用户。



### 3. Others

其实目前有非常多的 TCP 的拥塞控制协议，例如：

- **基于丢包的拥塞控制**：将丢包视为出现拥塞，采取缓慢探测的方式，逐渐增大拥塞窗口，当出现丢包时，将拥塞窗口减小，如 Reno、Cubic 等。
- **基于时延的拥塞控制**：将时延增加视为出现拥塞，延时增加时增大拥塞窗口，延时减小时减小拥塞窗口，如 Vegas、FastTCP 等。
- **基于链路容量的拥塞控制**：实时测量网络带宽和时延，认为网络上报文总量大于带宽时延乘积时出现了拥塞，如 BBR。
- **基于学习的拥塞控制**：没有特定的拥塞信号，而是借助评价函数，基于训练数据，使用机器学习的方法形成一个控制策略，如 Remy。