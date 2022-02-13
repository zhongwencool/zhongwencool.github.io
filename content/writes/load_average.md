---
title: "Linux负载"
tags:
- linux
---

通过top或uptime都可以查看到操作系统的CPU的1分钟/5分钟/15分钟的负载情况。
```bash
uptime
22:08:52 up 171 days, 12:03,  1 user,  load average: 0.00, 0.01, 0.05
```
CPU load average的平均值是如何被计算出来的？
1. Load average不包括任何等待 I/O、网络、数据库或其他不需要 CPU 的进程或线程。
2. 不要与 CPU Percentage混淆。CPU%位于top命令下的其中的列，如果它显示你的进程占比为45时，只是说明top在抽取的样本中发现45%都是你的进程在CPU上活动。其余的时间则是由其它进程占用。
    - 先使用stress 命令让并发2个进程系统不停的计算平方根。
```bash
stress -c 2
stress: info: [18480] dispatching hogs: 2 cpu, 0 io, 0 vm, 0 hdd
```
   
   -  使用top查看,可以看到2个stress子进程分别占用了48.9%的CPU，且机器使用的CPU率为99.3%， 说明在每次top取样时，stress子进程分别耗尽了近一半的CPU时间（PS：本系统只有一个CPU核，所以最大只能是100%）。
   
```bash
top - 22:21:11 up 171 days, 12:15,  2 users,  load average: 6.58, 3.90, 1.67
Tasks: 113 total,   4 running, 109 sleeping,   0 stopped,   0 zombie
%Cpu(s): 99.3 us,  0.7 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  1881996 total,    89888 free,   546700 used,  1245408 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  1149268 avail Mem

PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
18481 root      20   0    7312    100      0 R 48.9  0.0   1:56.50 stress
18482 root      20   0    7312    100      0 R 48.9  0.0   1:56.50 stress
 1623 grafana+  20   0  936596  56488   7216 S  0.7  3.0   1354:05 grafana-agent
 ...
``` 
   -  CPU是一个离散的状态机，它执行指令是处于100%的状态，等待时则为0%，不存在只使用了45%CPU的中间状态。所以top上的%CPU是一个时间函数，它是CPU某时间段（瞬间）指令集的快照。
   - 当你的应用程序处于等待（抢占）CPU时，也会是卡住，但是无法体现在top的%Cpu指标。

CPU抢占过程可类比于高速公路排队，四核都相当于四条高速跑道，车辆则代表着执行的指令。
因为CPU切换指令速度特别快，所以近似于CPU（非常小的时间段）可以同时跑多个指令，即：每条跑道为1公里，上面可同时跑多辆车，%CPU代表的是同时在这一公里跑道上车有多少车是属于你应用程序的。这无法反映对于跑道的潜在需求-在入口处排队的车辆情况。

为了更好的衡量在入口处排队的车辆情况，引入了CPU的负载，它不只是在CPU上跑的指令集，还需要统计等待CPU的指今情况。它包含了一种CPU利用率的趋势。CPU Load average可以告诉我们跑道的堵塞情况。

最理想的状态是保持CPU很忙，但是又没有进程需要等待，如果机器有4个CPU，而1分钟的Load Average为4.00，则说明在过去的60秒内一直在完美的利用CPU，4个跑道上一直保持4辆车在跑，同时没有等待。虽然计算大体如此，但是具体计算方法没有这么简单。

Linux内核中，每个可调度的进程在CPU上每次调度都有一个固定的时间量。默认为10毫秒，在这时间段内，进程可以被分配到一个物理CPU上运行指令，并允许接管CPU。但是10毫秒对于进程来说已经很长了（在英特尔2.6GHz处理器上，**10毫秒的时间足以发生大约5000万条指令**。这对于大多数应用程序周期来说，时间是绰绰有余的），所以更多的情况是进程会在10毫秒内通过套接字调用、I/O调用或回调内核而放弃控制。如果进程用完了分配给它的10ms的CPU时间，硬件就会发出一个中断，内核就会从进程中重新获得控制权。内核会立即惩罚这个（慢）进程。

全局的Load Average计算涉及到如何高效（分布式，异步）统计指标，过程非常复杂。如果想了解，可以查看`kernel/sched/loadavg.c`.

>
> kernel/sched/loadavg.c 
> This file contains the magic bits required to compute the global loadavg
>  figure. Its a silly number but people think its important. We go through
>  great pains to make it work on big machines and tickless kernels.
>

大概可以理解为每隔一段时间就会计算一次Load Average，且在计算的过程中不会发生中断（切换）。但是CPU为4核，Load Average为4时，只是近似可以理解为还有1个进程还在排队。平均负载的计算是一个非常复杂的过程，无法做到非常精确。

**总结**：`%cpu`反映的是当前CPU上跑本进程的指令的占比，`Load Average`除了当前CPU的繁忙程度外，还额外加上了在CPU外等待排队的情况，反映的是CPU的总体繁忙程度，包含着对未来负载的一种趋势。