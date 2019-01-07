---
layout: post
title: Linux time 和 Java gc 中的 real,user,sys
categories: blog
description: Linux time 和 Java gc 中的 real,user,sys
keywords: Linux time 和 Java gc 中的 real,user,sys
---

#### 1. Linux time 中的 real,user,sys
在 Linux 服务器上输入命令 `time sleep 2` 可以得到类似如下的输出：
![linux-time-sleep](/images/blog/linux-time-sleep.png)

其中最后三行输出了3个耗时：real, user, sys。解释如下：
* real：表示程序从开始到结束的耗时。也就是程序开始执行和程序结束时你分别看两次钟表时间，二者时间差就是 real 的值。这个耗时包括了该程序在运
行过程中等待 cpu 时间片的耗时（此时 cpu 时间片被其他程序占用）以及本程序阻塞的耗时（比如 IO 等阻塞）。

* user：表示程序运行过程中在*用户态*的 cpu 耗时。有2个注意点：
    * 不包括等待 cpu 时间片的耗时和阻塞的耗时，而是实实在在的本程序在*用户态*下占用的 cpu 的耗时
    * 如果是多核 cpu，要把每个 cpu 在*用户态*上服务这个程序的耗时累加起来

* sys：表示程序运行过程中在*内核态*的 cpu 耗时。同样的，如果是多核 cpu，要把每个 cpu 在*内核态*上服务这个程序的耗时累加起来。

从命令 `time sleep 2` 的输出可以看到，该程序（只需要单个 cpu 运行）此次执行的 real 耗时为 2.001s，sys 耗时为 0.001s。注意：这里的
real 刚好等于 user + sys，这只是巧合，不能误以为所有地方都有这个等式关系。

#### 2. Java gc 中的 real,user,sys
在 Java gc 日志中我们也能看到每个 gc 事件有这样的输出：
```
[Times: user=1.51 sys=0.03, real=0.21 secs]
```
这里的 user,sys,real 耗时含义解释同上。在对 Java gc 日志分析时，我们主要关心的是 real 耗时。

#### 3. real,user,sys 的关系
一般在程序使用了多核的情况下，real 会小于 user + sys 的总和，因为 user + sys 是累加了这个程序在各个 cpu 上的时间消耗，但是，
这只是一般性的判断。如下分析几个非一般的情况：

* real > user + sys
    如果程序使用了多核，那么根据 real 的含义，则重点注意如下情况：
    - CPU 资源是否紧张
        如果多核中很多核被其他进程使用，本进程就不得不等待 CPU 的时间片，这个时间会体现在 real 上。
    - IO 是否过重
        IO 可能是本程序引起的，也可能是别的进程导致的

* sys > user
    在 Java gc 中较罕见，因为一般地，在一个 gc 事件中大多数事件是花费在 jvm 代码上（用户态），而只有少量时间是在内核态。但是如果多次注意到
    sys > user 的情况，重点注意如下情况：
    - 操作系统的问题
        排查操作系统的 cpu/mem 使用情况，有无 page faults, cpu 过高占用等情况。
    - 虚拟机相关的问题
        虚拟机所在宿主机的资源隔离可能没有做好，导致其他其他进程影响了本进程。

#### 4. 参考：
* [what-do-real-user-and-sys-mean-in-the-output-of-time1](https://stackoverflow.com/questions/556405/what-do-real-user-and-sys-mean-in-the-output-of-time1?lq=1)
