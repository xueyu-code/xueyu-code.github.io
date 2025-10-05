# Index


## 整体框架
xenomai3的整体结构如下
![](../../attachment/xenomai3整体结构.png)
![](../../attachment/Xenomai3整体结构2.png)

---
**在内核空间，在标准linux基础上添加一个实时内核Cobalt，得益于基于ADEOS（Adaptive Domain Environment for Operating System)，使Cobalt在内核空间与linux内核并存，并把标准的Linux内核作为实时内核中的一个idle进程在实时内核上调度**

ADEOS ,其核心思想是Domain,也就是范围的意思，linux内核有linux内核的范围，cobalt内核有cobalt内核的范围。
- 两个内核管理各自范围内的应用、驱动、中断
- 两个domain之间有优先级之分，cobalt内核优先级高于linux内核
- I-pipe优先处理高优先级域的中断，来保证高优先级域的实时性
- 高优先级域可以通过I-pipe 向低优先级域发送各类事件等
**在用户空间**，添加针对实时应用优化的库--libcobalt，libcobalt提供POSIX接口给应用空间实时任务使用，应用通过libcobalt让实时内核cobalt提供服务。
**驱动方面**，xenomai提供实时驱动框架模型RTDM（Real-Time Driver Model）,专门用于Cobalt内核，基于RTDM进行实时设备驱动开发，为实时应用提供实时驱动。RTDM将驱动分为2类：
- 字符设备 如串口等
- 协议设备 如UDP/TCP CAN
**中断方面**，I-Pipe(interrupt Pipeline)分发Linux和Xenomai之间的中断，并以Domain优先级顺序传递中断。I-Pipe传递中断如下图所示，对于实时内核注册的中断，中断产生后能够直接得到处理，保证实时性。对于linux的中断，先将中断记录在i-log，等实时任务让出CPU后，linux得到运行，该中断才得到处理。
![](../../attachment/Xenomai中断1.png)
&gt; 更多信息可参考如下链接 https://www.cnblogs.com/wsg1100/p/12833126.html

## 其他Linux实时化方案

![](../../attachment/Linux实时化方案.png)
PREEMPT-RT方案主要缺点：通过修改Linux内核，难以保证实时进程的执行不会遭到非实时进程所进行的不可预测活动的干扰
![](../../attachment/Linux实时化方案双内核.png)

![](../../attachment/Linux实时化方案对比.png)
**Linux实时化如此艰难的主要原因是Linux中有非常明显的内核态和用户态的区分（本身从用户态陷入内核态就是个实时性很差的机制），内核态中各种临界区是不能抢占的，哪怕打完RT补丁，还是有很多地方不能抢占，比如spinlock默认禁止抢占（整个内核态有10万&#43;地方使用），同时硬中断的执行时间不确定，软中断总是抢占应用上下文等等影响任务调度时间的地方。**













































---

> Author:   
> URL: https://xueyu-code.github.io/posts/xenomai/72c6f94/  

