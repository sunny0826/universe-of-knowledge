# 进程与线程

进程与线程是面试中计算机系统方面问的比较多的一种知识，本文力求以最简单语言描述这两个知识点。Let's go！

## 背景知识

>处理速度：CPU > 内存 > 磁盘

>容量：CPU < 内存 < 磁盘

### CPU 的上下文切换

操作系统事先帮 CPU 设置好 **CPU 寄存器和程序计数器**。

**CPU 寄存器**是 CPU 内部的一个速度极快的小缓存；**程序计数器**则用来存储 CPU 正在执行的指令位置、或者即将执行的下一条指令的位置。

CPU 寄存器和程序计数器是 CPU 在运行任何任务前，都必须依赖的环境，这些环境就叫做 **CPU 上下文**。

CPU 上下文切换就是把前一个任务的 CPU 上下文（CPU 寄存器和程序计数器）保存起来，然后加载新任务的上下文，最后在跳转到新程序计数器所指的新位置，运行新任务。这样做的目的，就是保证任务在重新加载后，原来的任务状态不受影响，看起来还是连续运行的。

上面的「任务」主要包括：进程、线程和中断。根据任务的不同，把 CPU 上下文切换分为：**进程上下文切换**、**线程上下文切换**、**中断上下文切换**。

<!-- TODO 协程 -->
## 协程

协程属于用户态，大量的协程实际上只占用了内核态的一个线程。
协程数量和内核线程处理不一致时，需要有调度器来维护所有协程，尽可能让他们公平地使用 CPU。
在 go runtime 有自己的调度器，主要使用了 “work stealing” 算法。

<!-- Linux 系统的进程与线程 -->
<!-- TODO 编程语言对于调度算法的应用 -->

## 参考

[进程和线程基础知识全家桶，30 张图一套带走 - 小林coding](https://mp.weixin.qq.com/s/YXl6WZVzRKCfxzerJWyfrg)