---
title: 'Linux Load Average: Solving the Mystery'
date: 2020-01-16 20:33:34
category: 翻译
tags:
- linux
- load_average
---

本篇博客翻译自Brendan Gregg的技术考古文章：[Linux Load Average: Solving the Mystery](http://www.brendangregg.com/blog/2017-08-08/linux-load-averages.html)。翻阅这篇文章的原因是我在使用Prometheus做系统CPU使用量告警时，一个system_load的指标和自己预期的不太相符：总是在CPU余量还很大的情况下达到告警线。为此研究了一下Linux的Load Average指标。

<!--more-->

以下为原文翻译：

Load Average（以下译为平均负载）是工程中一个很重要的指标，我的公司使用该指标以及一些其它指标维持着数以百万计的云端实例进行自动扩容。但围绕着这个Linux指标一直以来都有一些谜团，比如这个指标不仅追踪正在运行的任务，也追踪处于uninterruptible sleep状态（通常是在等待IO）的任务。这到底是为什么呢？我之前从来没有找到过任何解释。因此这篇文章我将解决这个谜题，对平均负载指标做一些总结，供所有尝试理解这一指标的人作为参考。

Linux的平均负载指标，也即“system load average”，指的是系统一段时间内需要执行的线程（任务），也即正在运行加正在等待的线程数的平均数。这个指标度量的是系统需要处理的任务量，可以大于系统实际正在处理的线程数。大部分工具会展示1分钟，5分钟和15分钟的平均值。

```shell
$ uptime
 16:48:24 up  4:11,  1 user,  load average: 25.25, 23.40, 23.46

top - 16:48:42 up  4:12,  1 user,  load average: 25.25, 23.14, 23.37

$ cat /proc/loadavg 
25.72 23.19 23.35 42/3411 43603
```

简单做一些解释：
- 如果averages是0，表示你的系统处于空闲状态。
- 如果1分钟的数值高于5分钟或15分钟的数值，表示系统负载正在上升。
- 如果1分钟的数值低于5分钟或15分钟的数值，表示系统负载正在下降。
- 如果这些数值高于CPU数量，那么你可能面临着一个性能问题。（当然也要看具体情况）

通过一组三个数值，你可以看出系统负载是在上升还是下降，这对于你监测系统状况非常有用。而作为独立数值，这项指标也可以用作制定云端服务自动扩容的规则。但如果想要更细致地理解这些数值的含义，你还需要一些其它指标的帮助。一个单独的值，比如23-25，本身是没有任何意义的。但如果知道CPU的数量，这个值就能代表一个CPU-bound工作负载。

与其尝试对平均负载进行排错，我更习惯于观察其它几个指标。这些指标将在后面的“更好的指标（Better Metrics）”一章介绍。

# 历史

最初的平均负载指标只显示对CPU的需求：也即正在运行的程序数量加等待运行的程序数量。在1973年8月发表的名为“TENEX Load Average”[RFC546](https://tools.ietf.org/html/rfc546)文档中有很好的描述：

> [1] The TENEX load average is a measure of CPU demand. 
The load average is an average of the number of runnable processes over a given time period. 
For example, an hourly load average of 10 would mean that (for a single CPU system) at any time during that hour one could expect to see 1 process running and 9 others ready to run (i.e., not blocked for I/O) waiting for the CPU.

这篇文章还链向了一篇[PDF](https://tools.ietf.org/pdf/rfc546.pdf)文档，展示了一幅1973年7月手绘的平均负载图（如下所示），表明这个指标已经被使用了几十年。

如今，这些古老的操作系统源码仍然能在网上找到，以下代码片段节选自[TENEX](https://github.com/PDP-10/tenex)（1970年代早期）SCHED.MAC的宏观汇编程序：

```c
NRJAVS==3               ;NUMBER OF LOAD AVERAGES WE MAINTAIN
GS RJAV,NRJAVS          ;EXPONENTIAL AVERAGES OF NUMBER OF ACTIVE PROCESSES
[...]
;UPDATE RUNNABLE JOB AVERAGES

DORJAV: MOVEI 2,^D5000
        MOVEM 2,RJATIM          ;SET TIME OF NEXT UPDATE
        MOVE 4,RJTSUM           ;CURRENT INTEGRAL OF NBPROC+NGPROC
        SUBM 4,RJAVS1           ;DIFFERENCE FROM LAST UPDATE
        EXCH 4,RJAVS1
        FSC 4,233               ;FLOAT IT
        FDVR 4,[5000.0]         ;AVERAGE OVER LAST 5000 MS
[...]
;TABLE OF EXP(-T/C) FOR T = 5 SEC.

EXPFF:  EXP 0.920043902 ;C = 1 MIN
        EXP 0.983471344 ;C = 5 MIN
        EXP 0.994459811 ;C = 15 MIN
```

以下是当今[Linux](https://github.com/torvalds/linux/blob/master/include/linux/sched/loadavg.h)源码的一个片段（include/linux/sched/loadavg.h）:

```c
#define EXP_1           1884            /* 1/exp(5sec/1min) as fixed-point */
#define EXP_5           2014            /* 1/exp(5sec/5min) */
#define EXP_15          2037            /* 1/exp(5sec/15min) */
```

Linux也硬编码了1，5，15分钟这三个常量。

在更老的系统中也有类似的平均负载比如[Multics](http://web.mit.edu/Saltzer/www/publications/instrumentation.html)就有一个指数调度队列平均值（exponential scheduling queue average）。

# 三个数字

标题的三个数字指的是1分钟，5分钟，15分钟的平均负载。但要注意的是这三个数字并不是真正的“平均”，统计时间也不是真正的1分钟，5分钟和15分钟。从之前的汇编代码可以看出，1，5，15是等式中的一个常量，而这个等式实际计算的是平均每5s的指数衰减移动和（exponentially-damped moving sums）（译者：如果你和我一样对这个名词和公式一头雾水，本节随后有相关文章和代码链接）。这样计算出来的1，5，15分钟数值能更好地反应平均负载。

如果你拿一台空闲的机器，然后开启一个单线程CPU-bound的程序（例如一个单线程循环），那么60s后1min平均负载的值应该是多少？如果只是单纯的平均，那么这个值应该是1.0。但实际实验结果如下图所示：

![Load Average](https://hotlinkqiu-blog.oss-cn-shenzhen.aliyuncs.com/load-average/loadavg.png)

被称作“1分钟平均负载”的值在1分钟的点只达到了0.62。如果想要了解更多关于这个等式和类似的实验，Neil Gunther博士写了一篇文章：[How It Works](http://www.teamquest.com/import/pdfs/whitepaper/ldavg1.pdf)，而[loadavg.c](https://github.com/torvalds/linux/blob/master/kernel/sched/loadavg.c)这段linux源码也有很多相关计算的注释。

# Linux不可中断任务（Uninterruptible Tasks）

当平均负载指标第一次出现在linux中时，它们和其它操作系统一样，反映了对CPU的需求。但随后，Linux对它们做了修改，不仅包含了可运行的任务，也包含了处在不可中断（TASK_UNINTERRUPTIBLE or nr_uninterruptible）状态的任务。这个状态表示程序不想被信号量打断，例如正处于磁盘I/O或某些锁中的任务。你以前可能也通过ps或者top命令观察到过这些任务，它们的状态被标志为“D”。ps指令的man page对此这么解释：“uninterrupible sleep(usually IO)”。

加入了不可中断状态，意味着Linux的平均负载不仅会因为CPU使用上升，也会因为一次磁盘（或者NFS）负载而上升。如果你熟悉其它操作系统以及它们的CPU平均负载概念，那么包含不可中断状态的Linux的平均负载在一开始会让人难以理解。

为什么呢？为什么Linux要这么做？

有无数的关于平均负载的文章指出Linux加入了nr_uninterruptible，但我没有见过任何一篇解释过这么做的原因，甚至连对原因的大胆猜测都没有。我个人猜测这是为了让指标表示更广义的对资源的需求的概念，而不仅仅是对CPU资源的需求。

# 搜寻一个古老的Linux补丁

想了解Linux中一个东西为什么被改变了很简单：你可以带着问题找到这个文件的git提交历史，读一读它的改动说明。我查看了[loadavg.c](https://github.com/torvalds/linux/commits/master/kernel/sched/loadavg.c)的改动历史，但添加不可中断状态的代码是从一个更早的文件拷贝过来的。我又查看了那个更早的文件，但这条路也走不通：这段代码穿插在几个不同的文件中。我希望找到一条捷径，就使用git log -p下载了整个包含了4G文本文件的Linux github库，想回溯看这段代码第一次出现在什么时候，但这也是条死路：在整个Linux工程中，最老的改动要追溯到2005年，Linux引入了2.6.12-rc2版本的时候，但这个修改此时已经存在了。

在网上还有Linux的历史版本库（[这里](https://git.kernel.org/pub/scm/linux/kernel/git/tglx/history.git)和[这里](https://kernel.googlesource.com/pub/scm/linux/kernel/git/nico/archive/)），但在这些库中也没有关于这个改动的描述。为了最起码找到这个改动是什么时候产生的，我在[kernel.org](https://www.kernel.org/pub/linux/kernel/Historic/v0.99/)搜索了源码，发现这个改动在0.99.15已经有了，而0.99.13还没有，但是0.99.14版本丢失了。我又在其它地方找到了这个版本，并且确认改动是在1993年11月在Linux 0.99 patchlevel 14上实现的。寄希望于Linus在0.99.14的发布描述中会解释为什么改动，但结果也是死胡同：

> "Changes to the last official release (p13) are too numerous to mention (or even to remember)..." – Linus

他提到了很多主要改动，但并没有解释平均负载的修改。

基于这个时间点，我想在[关键邮件列表存档](http://lkml.iu.edu/hypermail/linux/kernel/index.html)中查找真正的补丁源头，但最老的一封邮件时间是1995年6月，系统管理员写下：

> "While working on a system to make these mailing archives scale more effecitvely I accidently destroyed the current set of archives (ah whoops)."

我的探寻之路仿佛被诅咒了。幸运的是，我找到了一些更老的从备份服务器中恢复出来的linux-devel邮件列表存档，它们用tar压缩包的形式存储着摘要。我搜索了6000个摘要，包含了98000封邮件，其中30000封来自1993年。但不知为何，这些邮件都已经遗失了。看来原始补丁的描述很可能已经永久丢失了，为什么这么做仍然是一个谜。

# “不可中断”的起源

谢天谢地，我最终在[oldlinux.org](http://oldlinux.org/Linux.old/mail-archive/)网站上一个来自于1993年的邮箱压缩文件中找到了这个改动，内容如下：

```text
From: Matthias Urlichs <urlichs@smurf.sub.org>
Subject: Load average broken ?
Date: Fri, 29 Oct 1993 11:37:23 +0200


The kernel only counts "runnable" processes when computing the load average.
I don't like that; the problem is that processes which are swapping or
waiting on "fast", i.e. noninterruptible, I/O, also consume resources.

It seems somewhat nonintuitive that the load average goes down when you
replace your fast swap disk with a slow swap disk...

Anyway, the following patch seems to make the load average much more
consistent WRT the subjective speed of the system. And, most important, the
load is still zero when nobody is doing anything. ;-)

--- kernel/sched.c.orig Fri Oct 29 10:31:11 1993
+++ kernel/sched.c  Fri Oct 29 10:32:51 1993
@@ -414,7 +414,9 @@
    unsigned long nr = 0;

    for(p = &LAST_TASK; p > &FIRST_TASK; --p)
-       if (*p && (*p)->state == TASK_RUNNING)
+       if (*p && ((*p)->state == TASK_RUNNING) ||
+                  (*p)->state == TASK_UNINTERRUPTIBLE) ||
+                  (*p)->state == TASK_SWAPPING))
            nr += FIXED_1;
    return nr;
 }
--
Matthias Urlichs        \ XLink-POP N|rnberg   | EMail: urlichs@smurf.sub.org
Schleiermacherstra_e 12  \  Unix+Linux+Mac     | Phone: ...please use email.
90491 N|rnberg (Germany)  \   Consulting+Networking+Programming+etc'ing      42
```

这种阅读24年前一个改动背后想法的感觉很奇妙。

这证实了关于平均负载的改动是有意为之，目的是为了反映对CPU以及对其它系统资源的需求。Linux的这项指标从“CPU load average”变为了“system load average”。

邮件中举的使用更慢的磁盘的例子很有道理：通过降低系统性能，对系统资源的需求应该增加。但当使用了更慢的磁盘时，平均负载指标实际上降低了。因为这些指标只跟踪了处在CPU运行状态的任务，没有考虑处在磁盘交换状态的任务。Matthias认为这不符合直觉，因此他做了相应的修改。

# “不可中断”的今日

一个问题是，今天如果你发现有时系统的平均负载过高，光靠disk I/O是否已经不足以解释？答案是肯定的，因为我会猜测Linux代码中加入了1993年时并不存在的设置TASK_UNINTERRUPTIBLE分支，进而导致平均负载过高。在Linux 0.99.14中，有13条代码路径将任务状态设置为TASK_UNINTERRUPIBLE或是TASK_SWAPPING（之后这个状态被从Linux中移除了）。时至今日，在Linux 4.12中，有接近400条代码分支设置了TASK_INTERRUPTIBLE状态，包括了一些加锁机制。很有可能其中的一些分支不应该被包括在平均负载统计中。下次如果我发现平均负载很高的情况，我会检查是否进入了不应被包含的分支，并看一下是否能进行一些修正。

我为此第一次给Matthias发了邮件，来问一问他当下对24年前改动的看法。他在一个小时内（就像我在twitter中说的一样）就回复了我，内容如下：

> "The point of "load average" is to arrive at a number relating how busy the system is from a human point of view. TASK_UNINTERRUPTIBLE means (meant?) that the process is waiting for something like a disk read which contributes to system load. A heavily disk-bound system might be extremely sluggish but only have a TASK_RUNNING average of 0.1, which doesn't help anybody."

（能这么快地收到回复，其实光是收到回复，就已经让我兴奋不已了，感谢！）

所以Matthias仍然认为这个指标是合理的，至少给出了原本TASK_UNINTERRUPTIBLE的含义。

但Linux衍化至今，TASK_UNINTERRUPIBLE代表了更多东西。我们是否应该把平均负载指标变为仅仅表征CPU和disk需求的指标呢？Scheduler的维护者Peter Zijstra已经给我发了一个取巧的方式：在平均负载中使用task_struct->in_iowait来代替TASK_UNINTERRUPTIBLE，用以更紧密地匹配磁盘I/O。这就引出了另外一个问题：到底什么才是我们想要的呢？我们想要的是度量系统的线程需求，还是想要分析系统的物理资源需求？如果是前者，那么等待不可中断锁的任务也应该包括在内，它们并不是空闲的。从这个角度考虑，平均负载指标当前的工作方式可能恰是我们所期望的。

为了更好地理解“不可中断”的代码分支，我更乐于做一些实际分析。我们可以检测不同的例子，量化执行时间，来看一下平均负载指标是否合理。

# 度量不可中断的任务

下面是一台生产环境服务器的[Off-CPU火焰图](http://www.brendangregg.com/blog/2016-01-20/ebpf-offcpu-flame-graph.html)，我过滤出了60秒内的内核栈中处于TASK_UNINTERRUPTIBLE状态的任务，这可以提供很多指向了uninterruptible代码分支的例子：

<embed src="http://www.brendangregg.com/blog/images/2017/out.offcputime_unint02.svg" />

如果你不熟悉Off-CPU火焰图：每一列是一个任务的完整塔形栈，它们组成了火焰的样子。你可以点击每个框来放大观察完整的栈。x轴大小与任务花费在off-CPU上的时间成正比，而从左到右的排序没有什么实际含义。off-CPU栈的颜色我使用蓝色（在on-CPU图上我使用暖色），颜色的饱和度是随机生成的，用以区分不同的框。

我使用我在[bcc](https://github.com/iovisor/bcc)工程下的offcputime工具来生成这张图，指令如下：

```shell
# ./bcc/tools/offcputime.py -K --state 2 -f 60 > out.stacks
# awk '{ print $1, $2 / 1000 }' out.stacks | ./FlameGraph/flamegraph.pl --color=io --countname=ms > out.offcpu.svgb>
```

awk命令将微秒输出为毫秒，--state 2表示TASK_UNINTERRUPTIBLE（参见sched.h文件），是我为这篇文章添加的一个可选参数。第一次这么做的人是Facebook的Josef Bacik，使用他的[kernelscope](https://github.com/josefbacik/kernelscope)工具，这个工具也使用了bcc和火焰图。在我的例子中，我只展示了内核栈，而offcputime.py也支持展示用户栈。

这幅图显示，在60s中uninterruptible睡眠只花费了926ms，这只让我们的平均负载增加了0.015。这些时间大部分花在cgroup相关代码上，disk I/O并没有花太多时间。

下面是一张更有趣的图，只覆盖了10s的时间：

<embed src="http://www.brendangregg.com/blog/images/2017/out.offcputime_unint01.svg" />

图形右侧比较宽的任务表示proc_pid_cmdline_read()（参考/proc/PID/cmdline）中的systemd-journal任务，被阻塞并对平均负载贡献了0.07。而左侧更宽的图形表示一个page_fault，同样以rwsem_down_read_failed()结束，对平均负载贡献了0.23。结合火焰图的搜索特性，我已经使用品红高亮了相关函数，这个函数的源码片段如下：

```c
    /* wait to be given the lock */
    while (true) {
        set_task_state(tsk, TASK_UNINTERRUPTIBLE);
        if (!waiter.task)
            break;
        schedule();
    }
```

这是一段使用TASK_UNINTERRUPTIBLE获取锁的代码。Linux对于互斥锁的获取有可中断和不可中断的实现方式（例如mutex_lock()和mutex_lock_interruptible()，以及对信号量的down()和down_interruptible()），可中断版本允许任务被信号中断，唤醒后继续处理。处在不可中断的锁中睡眠的时间通常不会对平均负载造成很大的影响。但在这个例子中，这类任务增加了0.30的平均负载。如果这个值再大一些，就值得分析是否有必要减少锁的竞争来优化性能，降低平均负载（例如我将开始研究systemd-journal和proc_pid_cmdline_read()）。

那么这些代码路径应该被包含在平均负载统计中吗？我认为应该。这些线程处在执行过程中，然后被锁阻塞。它们并不是空闲的，它们对系统有需要，尽管需求的是软件资源而非硬件资源。

# 分拆Linux Load Averages

那么Linux平均负载能否被完全拆成几个部件呢？下面是一个例子：在一台空闲的有8个CPU的系统上，我调用tar来打包一些未缓存的文件。这个过程会花费几分钟，大部分时间被阻塞在读磁盘上。下面是从三个终端窗口搜集的数据：

```shell
terma$ pidstat -p `pgrep -x tar` 60
Linux 4.9.0-rc5-virtual (bgregg-xenial-bpf-i-0b7296777a2585be1)     08/01/2017  _x86_64_    (8 CPU)

10:15:51 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
10:16:51 PM     0     18468    2.85   29.77    0.00   32.62     3  tar

termb$ iostat -x 60
[...]
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.54    0.00    4.03    8.24    0.09   87.10

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
xvdap1            0.00     0.05   30.83    0.18   638.33     0.93    41.22     0.06    1.84    1.83    3.64   0.39   1.21
xvdb            958.18  1333.83 2045.30  499.38 60965.27 63721.67    98.00     3.97    1.56    0.31    6.67   0.24  60.47
xvdc            957.63  1333.78 2054.55  499.38 61018.87 63722.13    97.69     4.21    1.65    0.33    7.08   0.24  61.65
md0               0.00     0.00 4383.73 1991.63 121984.13 127443.80    78.25     0.00    0.00    0.00    0.00   0.00   0.00

termc$ uptime
 22:15:50 up 154 days, 23:20,  5 users,  load average: 1.25, 1.19, 1.05
[...]
termc$ uptime
 22:17:14 up 154 days, 23:21,  5 users,  load average: 1.19, 1.17, 1.06
```

我也同样为不可中断状态的任务搜集了Off-CPU火焰图：

<embed src="http://www.brendangregg.com/blog/images/2017/out.offcputime_unint08.svg" />

最后一分钟的平均负载是1.19，让我们来分解一下：

* 0.33来自于tar的CPU时间（pidstat）
* 0.67来自于不可中断的磁盘读（off-CPU火焰图中显示的是0.69，我怀疑是因为脚本搜集数据稍晚了一些，造成了时间上的一些微小的误差）
* 0.04来自于其它CPU消费者（iostat user + system，减去pidstat中tar的CPU时间）
* 0.11来自于内核态处理不可中断的disk I/O的时间，向磁盘写入数据（通过off-CPU火焰图，左侧的两个塔）

这些加起来是1.15，还少了0.04。一部分可能源自于四舍五入，以及测量间隔的偏移造成的误差，但大部分应该还是因为平均负载使用的是“指数衰减偏移和”，而其它的平均数（pidstat，iostat）就是普通的平均。在1.19之前一分钟的平均负载是1.25，因此这一分钟的值会拉高下一分钟的平均负载。会拉高多少呢？根据之前的图，在我们统计的一分钟里，有62%来自于当前的一分钟时间。所以0.62 * 1.15 + 0.38 * 1.25 = 1.18，和报告中的1.19很接近了。

这个例子中，系统里有一个线程(tar)加上一小部分其它线程（也有一些内核态工作线程）在工作，因此Linux报告平均负载1.19是说得通的。如果只显示“CPU平均负载”，那么值会是0.37（根据mpstat的报告），这个值只针对CPU资源正确，但隐藏了系统上实际有超过一个线程需要维持工作的事实。

通过这个例子我想说明的是平均负载统计的数字（CPU+不可中断）的确是有意义的，而且你可以分解并计算出各个组成部分。

（作者在原文评论中说明了计算这些数值的方式：）

> tar: the off-CPU flame graph has 41,164 ms, and that's a sum over a 60 second trace. Normalizing that to 1 second = 41.164 / 60 = 0.69. The pidstat output has tar taking 32.62% average CPU (not a sum), and I know all its off-CPU time is in uninterruptible (by generating off-CPU graphs for the other states), so I can infer that 67.38% of its time is in uninterruptible. 0.67. I used that number instead, as the pidstat interval closely matched the other tools I was running.
  by mpstat I meant iostat sorry (I updated the text), but it's the same CPU summary. It's 0.54 + 4.03% for user + sys. That's 4.57% average across 8 CPUs, 4.57 x 8 = 36.56% in terms of one CPU. pidstat says that tar consumed 32.62%, so the remander is 36.56% - 32.62% = 3.94% of one CPU, which was used by things that weren't tar (other processes). That's the 0.04 added to load average.


# 理解Linux Load Averages

我成长在平均负载只表达CPU负载的操作系统环境中，因此Linux版的平均负载经常让我很困扰。或许根本原因是词语“平均负载”就像“I/O”一样意义不明：到底是什么I/O呢？磁盘I/O？文件系统I/O？网络I/O?...，同样的，到底是哪些负载呢？CPU负载？还是系统负载？用下面的方式解释能让我理解平均负载这个指标：

* 在Linux系统中，平均负载是（或希望是）“系统平均负载”，将系统作为一个整体，来度量所有工作中或等待（CPU，disk，不可中断锁）中的线程数量。换句话说，指标衡量的是所有不完全处在idle状态的线程数量。优点：囊括了不同种类资源的需求。
* 在其它操作系统中：平均负载是“CPU平均负载”，度量的是占用CPU运行中或等待CPU的线程数量。优点：理解起来，解释起来都很简单（因为只需要考虑CPU）。

请注意，还有另一种可能的平均负载，即“物理资源平均负载”，只囊括物理资源（CPU + disk）

或许有一天我们会为Linux添加不同的平均负载，让用户来选择使用哪一个：一个独立的“CPU平均负载”，“磁盘平均负载”和“网络平均负载”等等。或者简单的把所有不同指标都罗列出来。

# 什么是一个好的或坏的平均负载？

一些人找到了对他们的系统及工作负载有意义的值：当平均负载超过这个值X时，应用时延飙高，且用户会开始投诉。但如何得到这个值实际上并没有什么规律。

如果使用CPU平均负载，人们可以用数值除以CPU核数，然后说如果比值超过1.0，你的系统就处于饱和状态，可能会引起性能问题。但这也很模棱两可，因为一个长期的平均值（至少1分钟）也可能隐藏掉一些变化。比如比值1.5对于一个系统来说，可能工作地还不错，但对另一个系统而言，比值突升到1.5，这一分钟的性能表现可能就会很糟糕。

![Load averages measured in a modern tool](https://hotlinkqiu-blog.oss-cn-shenzhen.aliyuncs.com/load-average/vectorloadavg.png)

我曾经管理过一台双核邮件服务器，平常运行时CPU平均负载在11到16之间（比值就是5.5到8），时延还是可以接受的，也没有人抱怨。但这是一个极端的例子，大部分系统可能比值超过2对服务性能就有很大影响了。

而对于Linux的系统平均负载，情况就更复杂更模糊了，因为这个指标包含了各种不同的资源类型，因此你不能单纯地直接除以CPU核数。此时使用相对值比较更有效：如果你知道系统在平均负载为20时工作地很好，而现在平均负载已经达到40了，那么你就该结合其它指标看一看到底发生了什么。

# 更好的指标

当Linux的平均负载指标上升时，你知道你的系统需要更好的资源（CPU，disk以及一些锁），但你其实并不确定需要哪一个。那么你就可以用一些其它指标来区分。例如，对于CPU：

* per-CPU utilization：使用 mpstat -P ALL 1；
* per-process CPU utilization：使用 top，pidstat 1等等；
* per-thread run queue(scheduler) latency：使用in /proc/PID/schedstats，delaystats，pref sched；
* CPU run queue latency：使用in /proc/schedstat，perf sched，我的[runqlat](http://www.brendangregg.com/blog/2016-10-08/linux-bcc-runqlat.html) [bcc](https://github.com/iovisor/bcc)工具；
* CPU run queue length：使用vmstat 1，观察'r'列，或者使用我的runqlen bcc工具。

前两个指标评估的是利用率，后三个是饱和度指标。利用率指标用来描述工作负载，而饱和度指标则用来鉴别性能问题。表示CPU饱和度的最佳指标是run queue（或者scheduler）latency：任务或线程处在可运行状态但需要等待运行的时间。这些指标可以帮助你度量性能问题的严重程度，例如一个任务处在等待时间的百分比。度量run queue的长度也可以发现问题，不过难以度量严重程度。

schedstats组件在Linux 4.6中被设置成了内核可调整，并且改为了默认关闭。[cpustat](https://github.com/uber-common/cpustat)的延迟统计同样统计了scheduler latency指标，我也刚刚建议把它加到[htop](https://github.com/hishamhm/htop/issues/665)中去，这样可以大大简化大家的使用，要比从/proc/sched_debug的输出中抓取等待时间指标简单。

```shell
$ awk 'NF > 7 { if ($1 == "task") { if (h == 0) { print; h=1 } } else { print } }' /proc/sched_debug
            task   PID         tree-key  switches  prio     wait-time             sum-exec        sum-sleep
         systemd     1      5028.684564    306666   120        43.133899     48840.448980   2106893.162610 0 0 /init.scope
     ksoftirqd/0     3 99071232057.573051   1109494   120         5.682347     21846.967164   2096704.183312 0 0 /
    kworker/0:0H     5 99062732253.878471         9   100         0.014976         0.037737         0.000000 0 0 /
     migration/0     9         0.000000   1995690     0         0.000000     25020.580993         0.000000 0 0 /
   lru-add-drain    10        28.548203         2   100         0.000000         0.002620         0.000000 0 0 /
      watchdog/0    11         0.000000   3368570     0         0.000000     23989.957382         0.000000 0 0 /
         cpuhp/0    12      1216.569504         6   120         0.000000         0.010958         0.000000 0 0 /
          xenbus    58  72026342.961752       343   120         0.000000         1.471102         0.000000 0 0 /
      khungtaskd    59 99071124375.968195    111514   120         0.048912      5708.875023   2054143.190593 0 0 /
[...]
         dockerd 16014    247832.821522   2020884   120        95.016057    131987.990617   2298828.078531 0 0 /system.slice/docker.service
         dockerd 16015    106611.777737   2961407   120         0.000000    160704.014444         0.000000 0 0 /system.slice/docker.service
         dockerd 16024       101.600644        16   120         0.000000         0.915798         0.000000 0 0 /system.slice/
[...]
```

除了CPU指标，你也可以找到度量磁盘设备使用率和饱和度的指标。我主要使用[USE method](http://www.brendangregg.com/usemethod.html)中的指标，并且会参考它们的[Linux Checklist](http://www.brendangregg.com/USEmethod/use-linux.html)。

尽管有很多更加具体的指标，但这并不意味着平均负载指标没用。平均负载配合其它指标可以成功应用在云计算微服务的自动扩容策略中，能够帮助微服务应对CPU、磁盘等不同原因造成的负载上升。有了自动扩容策略，即使造成了错误的扩容（烧钱）也比不扩容（影响用户）要安全，因此人们会倾向于在自动扩容中加入更多的信号。如果某次自动扩容扩了太多，我们也能够到第二天进行debug。

促使我继续使用平均负载指标的另一个原因是它们（三个数字）能表示历史信息。如果我需要去检查云端一台实例为什么表现很差，然后登录到那台机器上，看到1分钟平均负载已经大大低于15分钟平均负载，那我就知道我错过了之前发生的性能问题。我只需几秒钟思考平均负载数值就能得到这个结论，而不需要去研究其它指标。

# 总结

在1993年，一位Linux工程师发现了一个平均负载体现不直观的问题，然后使用一个三行代码的补丁永久地把Load Average指标从“CPU平均负载”变成了，可能叫做“系统平均负载”更合适的指标。他的改动包含了处在不可中断状态的任务，因此平均负载反映了任务对CPU以及磁盘的需求。系统负载均衡指标计算了工作中以及等待工作中的线程数量，并且使用1，5，15这三个常数，通过一个特殊公式计算出了三个“指数衰减偏移和”。这三个数字让你了解你的系统负载是在增加还是减少，它们的最大值可能可以用来做相对比较，以确定系统是否有性能问题。

在Linux内核代码中，不可中断状态的情况越来越多，到今天不可中断状态也包含了获取锁的状态。如果平均负载是一个用来计算正在运行以及正在等待的线程数量（而不是严格表示线程等待硬件资源）的指标，那么它们的数值仍然符合预期。

在这篇文章中，我挖掘了这个来自1993年的补丁——寻找过程出乎意料地困难——并看到了作者最初的解释。我也在现代Linux系统上通过bcc/eBPF研究了处在不可中断状态的任务的堆栈和花费时间，并且把这些表示成了一幅off-CPU火焰图。在图中提供了不少处在不可中断状态的例子，可以随时用来解释为什么平均负载的值飙高。同时我也提出了一些其它指标来帮助你了解系统负载细节。

我会引用Linux源码中scheduler维护者Peter Zijlstra写在[kernel/sched/loadavg.c](https://github.com/torvalds/linux/blob/master/kernel/sched/loadavg.c)顶部的注释来结束这篇文章：

```c
  * This file contains the magic bits required to compute the global loadavg
  * figure. Its a silly number but people think its important. We go through
  * great pains to make it work on big machines and tickless kernels.
```

# 参考资料

[1] Saltzer, J., and J. Gintell. “[The Instrumentation of Multics](http://web.mit.edu/Saltzer/www/publications/instrumentation.html),” CACM, August 1970 （解释了指数）
[2] Multics [system_performance_graph](http://web.mit.edu/multics-history/source/Multics/doc/privileged/system_performance_graph.info) command reference （提到了1分钟平均负载）
[3] [TENEX](https://github.com/PDP-10/tenex) source code.（CHED.MAC系统中的平均负载代码）
[4] [RFC 546](https://tools.ietf.org/html/rfc546) "TENEX Load Averages for July 1973".（解释了对CPU需求的度量） 
[5] Bobrow, D., et al. “TENEX: A Paged Time Sharing System for the PDP-10,” Communications of the ACM, March 1972.（解释了三重平均负载） 
[6] Gunther, N. "UNIX Load Average Part 1: How It Works" [PDF](http://www.teamquest.com/import/pdfs/whitepaper/ldavg1.pdf). （解释了指数计算公式）
[7] Linus's email about [Linux 0.99 patchlevel 14](http://www.linuxmisc.com/30-linux-announce/4543def681c7f27b.htm).
[8] The load average change email is on [oldlinux.org](http://oldlinux.org/Linux.old/mail-archive/).（在alan-old-funet-lists/kernel.1993.gz压缩包中，不在我一开始搜索的linux目录下）
[9] The Linux kernel/sched.c source before and after the load average change: [0.99.13](http://kernelhistory.sourcentral.org/linux-0.99.13/?f=/linux-0.99.13/S/449.html%23L332), [0.99.14](http://kernelhistory.sourcentral.org/linux-0.99.14/?f=/linux-0.99.14/S/323.html%23L412).
[10] Tarballs for Linux 0.99 releases are on [kernel.org](https://www.kernel.org/pub/linux/kernel/Historic/v0.99/).
[11] The current Linux load average code: [loadavg.c](https://github.com/torvalds/linux/blob/master/kernel/sched/loadavg.c), [loadavg.h](https://github.com/torvalds/linux/blob/master/include/linux/sched/loadavg.h)
[12] The [bcc](https://github.com/iovisor/bcc) analysis tools includes my [offcputime](http://www.brendangregg.com/blog/2016-01-20/ebpf-offcpu-flame-graph.html), used for tracing TASK_UNINTERRUPTIBLE.
[13] [Flame Graphs](http://www.brendangregg.com/flamegraphs.html) were used for visualizing uninterruptible paths.
