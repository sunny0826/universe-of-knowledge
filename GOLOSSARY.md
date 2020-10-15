# 概念

## 页表

页表是一种特殊的数据结构，放在系统空间的页表区，存放逻辑页与物理页帧的对应关系。 每一个进程都拥有一个自己的页表，PCB表中有指针指向页表。

## 用户态（User mode）

控制计算机的硬件资源，并提供上层应用程序运行的环境。

![Unix和Linux的体系架构](https://tva4.sinaimg.cn/large/ad5fbf65ly1gj2owxr8grj20a609rmxg.jpg)

## 内核态（kernel mode）

上层应用程序的活动空间，应用程序的执行必须依托于内核提供的资源。

## 系统调用

为了使上层应用能够访问到这些资源，内核为上层应用提供访问的接口。

![三者关系](https://tvax1.sinaimg.cn/large/ad5fbf65ly1gj2p5pcphxj21580b877a.jpg)

## lseek 函数

lseek 是一个用于改变读写一个文件时读写指针位置的一个系统调用。指针位置可以是绝对的或者相对的。

## CAP原则

CAP 原则又称 CAP 定理，指的是在一个分布式系统中，一致性（Consistency）、可用性（Availability）、分区容错性（Partition tolerance）。CAP 原则指的是，这三个要素最多只能同时实现两点，不可能三者兼顾。