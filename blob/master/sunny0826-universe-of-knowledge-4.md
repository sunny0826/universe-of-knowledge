# sunny0826/universe-of-knowledge

[Permalink](https://github.com/sunny0826/universe-of-knowledge/blob/a065dd75bbe1a75c388662bd4362ddf94bcc2526/concepts.md)

Cannot retrieve contributors at this time

 27 lines \(14 sloc\) 952 Bytes

## 概念

### 页表

页表是一种特殊的数据结构，放在系统空间的页表区，存放逻辑页与物理页帧的对应关系。 每一个进程都拥有一个自己的页表，PCB表中有指针指向页表。

### Linux 操作系统概念

#### 用户态（User mode）

控制计算机的硬件资源，并提供上层应用程序运行的环境。

[![Unix&#x548C;Linux&#x7684;&#x4F53;&#x7CFB;&#x67B6;&#x6784;](https://camo.githubusercontent.com/ef098375c0b0f6e3e8b66102bd842978059c8b86/68747470733a2f2f747661342e73696e61696d672e636e2f6c617267652f61643566626636356c7931676a326f7778723867726a323061363039726d78672e6a7067)](https://camo.githubusercontent.com/ef098375c0b0f6e3e8b66102bd842978059c8b86/68747470733a2f2f747661342e73696e61696d672e636e2f6c617267652f61643566626636356c7931676a326f7778723867726a323061363039726d78672e6a7067)

#### 内核态（kernel mode）

上层应用程序的活动空间，应用程序的执行必须依托于内核提供的资源。

#### 系统调用

为了使上层应用能够访问到这些资源，内核为上层应用提供访问的接口。

[![&#x4E09;&#x8005;&#x5173;&#x7CFB;](https://camo.githubusercontent.com/2d6d337612d9e85e65a9524e9407bdbe99d3294c/68747470733a2f2f74766178312e73696e61696d672e636e2f6c617267652f61643566626636356c7931676a32703570637068786a323135383062383737612e6a7067)](https://camo.githubusercontent.com/2d6d337612d9e85e65a9524e9407bdbe99d3294c/68747470733a2f2f74766178312e73696e61696d672e636e2f6c617267652f61643566626636356c7931676a32703570637068786a323135383062383737612e6a7067)

### lseek 函数

lseek 是一个用于改变读写一个文件时读写指针位置的一个系统调用。指针位置可以是绝对的或者相对的。

