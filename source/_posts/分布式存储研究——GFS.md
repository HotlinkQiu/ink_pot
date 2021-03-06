---
title: 分布式存储研究——GFS
date: 2019-07-13 11:04:51
category: 论文研读
tags:
- 分布式
- 存储
- Google
---

为了在分布式系统的技术栈中有更深的认识和更坚实的基础，准备在这个季度（2019年三季度）花费工作外的时间对Google和Amazon一系列分布式数据的论文进行研读学习，并将研读结果整理为博客记录。

本篇研读的论文是The Google File System，发表于2003年的SOSP，介绍了Google的GFS文件存储系统。

<!--more-->

# Design Overview

## Assumptions

GFS本身属于分布式存储系统，其设计主要针对以下几点假设：

+ 系统构建于廉价的部件上，部件失效是正常事件。
+ 系统存储的是数量较少的大文件。具体来说，预期文件数量在百万级，每个文件大小通常会达到100MB以上，也会有文件达到几个GB的大小。
+ 系统上文件的读场景主要有两种。一种是每次读取操作会访问几百K甚至1MB以上连续数据的大规模流式读取。一种则是一次读取几百K随机位置数据的小规模随机读取。
+ 系统上会有大量顺序写入的append操作，数据量与读操作相当，且一旦写入很少会进行修改。
+ 系统需要有效地实现定义良好的语义来支持多客户端的并发操作。其重点是使用最少的同步开销来保证操作原子性。
+ 相比于低延迟，持续的高带宽更重要。

这些假设实际上也就是GFS分布式文件系统的需求。为了精确匹配这些需求，GFS做了许多特殊设计和优化操作。

而从系统设计的角度来说，没有哪一种设计是能够服务所有场景的。在进行系统、算法、数据结构的设计之前，我们都应该为它建立一个精准的场景假设，从而在方案取舍时有明确的倾向性，也符合程序“do one thing and do it well”的哲学。

## Architecture

GFS的主要架构如下图：

![GFS架构](https://hotlinkqiu-blog.oss-cn-shenzhen.aliyuncs.com/gfs/gfs_architecture.PNG)

系统由一个master节点和多个chunk server节点组成。数据文件都被切分成了固定大小的chunk，并由一个64bit的chunk handle唯一标识。同时每个chunk会保留3份副本以保证高可用性。

## Single Master

单个主节点能够简化整个集群的设计，主节点能够完成所有的决策工作。但同时，为了保证主节点不会成为性能瓶颈，设计时做了几点优化：

+ master节点从不负责数据传输，只负责控制协议，即告知client请求的chunk在哪个chunk server上。
+ client对同一个chunk的请求只需要一次和master的交互，之后会缓存chunk的信息直到过期。
+ client可以一次请求多个chunk信息。
+ master也可以返回请求chunk之后紧跟着的chunk信息给client，减少client和master节点的交互。

可以看到，大部分的优化都利用了程序的局部性，简单且实用。

## Chunk

在GFS系统中，为了应对特殊的大数据需求，chunk设计也有如下特点：

+ chunk大小为64MB，远高于传统的文件系统。
+ chunk存储采用惰性空间分配，尽可能避免碎片。

chunk设计的很大，一方面适用于大数据应用的场景，另一方面减少了client和master、chunk server的交互。同样的数据量下，chunk的位置信息和元数据也更少，client更容易缓存，master也更容易存储。

这种chunk设计也有一个缺点，就是如果一份文件比较小，只占用几个或只有一个chunk，同时又是一个热点文件时，会造成存储这份文件的几个chunk server负载过重。论文中给出了两个方案：

+ 快速的解决方案是增加热点文件的副本数量，分散client的读取。
+ 更长期的解决方案，是允许一个client从其它client上读取热点数据。

## Metadata

主节点保存了所有的元数据。元数据包括文件和chunk的namespace、文件到chunk的映射关系、以及各个chunk所在节点的信息。为了保障整个集群的高可用性，对metadata有以下特殊处理：

+ 所有的metadata均保存在master节点的内存中。这里再次显示出大chunk设计的优势，每个chunk通常只需要64 bytes的metadata。如果数据量进一步增加，只需要再增加master节点的内存即可。
+ master节点并不会持久化存储chunk server存储的chunk副本信息，而是在启动时，以及chunk server加入集群时向chunk server查询。同时通过监控和心跳包时刻保持信息更新。原因则是chunk server是很有可能出现故障的，实际的chunk位置信息也可能会因为各种原因产生改变。
+ GFS会持久化存储的是operation log，操作日志。它记录了并时序化了整个系统metadata的变动信息，所有事件的前后顺序以操作日志记录为准。操作日志通常会保留多个远程副本，只有在所有副本都同步后才会响应metadata的修改操作。
+ 同时当master节点宕机时也会通过重复操作日志来恢复，master节点会在操作日志达到一定大小时在本地保存checkpoint，每次恢复从上一个checkpoint开始重复执行有限的操作。checkpoint是一个压缩B树，可以方便的映射到内存。

## Consistency Model

首先定义：一块文件区域是“consistent”，代表无论从哪个分片，读取到的数据是一致的。一块文件区域是“defined”，代表在一次数据改变之后，这块文件区域是“consistent”，且所有客户端可以观察到这次操作所作的具体修改。这里的修改表示“Write”随机写操作和“append”末尾添加操作。

GFS对于系统的一致性有如下的保障：

+ 文件命名空间的改变（例如创建文件）是原子的。这部分操作由master节点加锁完成，会通过操作日志形成一个唯一的时序动作。
+ 一次成功的修改操作，在没有并发生产者的干扰下，文件区域是“defined”的。
+ 一次成功的并发“write”操作，文件区域是“consistent”的。客户端虽然能读到同样的数据，但是并不能确定每一次操作具体修改了哪一部分。
+ 一次成功的并发“append”操作，文件区域是“defined”的。原因是append的位置是由系统算法选择的，并会返回给客户端以标记出此修改的开始和结束位置。
+ 一次失败的修改操作，会导致文件区域“inconsistent”。

# System Interactions

为了实现上一章中描述的架构，达成一致性模型，GFS系统内部通过通信协作完成三类主要工作：修改数据，原子的append操作以及快照。

## Data Flow

数据修改的流程如下图：

![GFS数据流](https://hotlinkqiu-blog.oss-cn-shenzhen.aliyuncs.com/gfs/gfs_data_flow.PNG)

首先需要说明，在一个chunk的多个副本中，有一个副本作为“primary”副本。primary副本会整合对chunk的所有写操作，形成一个线性序列。

1. client询问master节点持有chunk lease的副本以及其它chunk副本的位置。如果还没有分配lease，则由master节点进行分配。
2. master节点回复primary和其它副本的位置信息，client会缓存这些信息。
3. client将修改推送到所有副本。每个chunk server会将修改缓存到LRU的缓冲区，直到被消费或过期。图中可以看到数据流和控制流进行了解耦，使得数据流的流向可以根据拓扑决定，例如图中副本A离Client更近，数据再由副本A发往primary副本。primary副本再发到副本B，是一条根据最近原则构成的转发链。
4. 当所有副本都收到修改数据后，client向primary副本发送一个写请求。primary副本会对类似的所有client发来的请求进行排序，并按顺序执行。
5. primary副本再通知其它副本按照同样的顺序执行修改。
6. 其它副本修改完后就告知primary副本。
7. primary副本将修改结果告知client。

如果这其中有任何的错误，client都会重试步骤3-7若干次。此时如上一章所述，文件区域是“inconsistent”的。

而如果一次修改涉及到多个chunk，那么client就会把它分解为对应各个chunk的若干个单独修改。此时如果还有其它client在并发修改，那么文件区域就处于“consistent”但不“defined”状态。各个分片上的数据是一致的，但修改是混合的。

## Atomic Record Appends

append的数据流与数据修改基本一致。不同点在于，primary在接到写请求后，会计算写的起始和结束位置，并告知其它副本一模一样的位置。而此时并发的其它写操作，则会在排序后在上一个写操作之后的高地址进行。操作间不会写同一块文件区域。也因此append操作是原子的，且文件区域是“defined”。

需要注意的是，当append操作失败时，同样会进行重试，而此时重试会发生在新的高地址中。也因此chunk中会包含一条record的重复数据。此时需要消费端通过记录的校验和和唯一标识等进行判断。

> 那么问题是，这里为什么不设计成在原地址上进行重试呢？：在原址上进行重试就变成了overwrite而不是append操作了。实际上append操作要比overwrite高效很多，因为它不必考虑过多的分布式锁等问题。那么可以看到，GFS在这里宁愿多浪费存储空间，也要使用append操作提升性能。

## Snapshot

GFS创建快照应用了已有的copy-on-write技术。顾名思义，这种方法在拷贝时会首先创建新的虚拟空间指向原空间，在有写改动时再进行实际的复制。

在GFS中，当创建快照的请求到来时，master会先收回所有待拷贝的chunk租约，保证后续的写请求会先通过和master节点交互。收回租约后就复制原有chunk的metadata，此时新的chunk其实仍然指向旧的空间。接着当一个新的写请求到来时，需要向master询问primary chunk信息，master会挑选一个新的chunk持有租约，并通知所有chunk原来位置的chunk server本地复制chunk，得到新的chunk并开始正常的写操作。

# Master Operations

可以看到，整个GFS系统逻辑最复杂的地方其实是master节点。论文第四章就单列一章讲述了master节点做各种操作的方法。

## Namespace

GFS不会像传统文件系统一样对每个目录产生详细的数据结构，而是将每个文件和它前面完整的目录前缀映射到metadata中。例如“/d1/d2/.../dn/leaf”，作为一个完整的文件路径进行映射。

为了支持并发操作，文件的读操作会依目录顺序依次申请各级目录的读锁：“/d1”、“/d1/d2”、“/d1/d2/.../dn/”、“/d1/d2/.../dn/leaf”，而写操作则会依次申请各级目录的读锁，和最终写文件的写锁“/d1/d2/.../dn/leaf”。

这种机制在保证并发操作正确性的情况下，提供了很好的并发性能。

## Chunk Replica

在创建chunk副本时，master主机会考虑：

+ 将chunk放在磁盘负载低于平均水平的机器上，保证负载均衡。
+ 限制每台主机最近新增的chunk数量，保证写操作能够分散。
+ 将多个副本放在多个机架多个机房的主机上，能够容灾以及在读操作时最大限度利用带宽（当然写操作时也增加了带宽，是一个trade off）。

当chunk副本数量低于预期时，master会对需要恢复的chunk进行排序，然后就像新建chunk一样申请新的位置并从还剩下的valid chunk中复制数据。排序主要遵循以下几点：

+ 副本少于期望的数量越多，优先级越高。
+ 依然存在的chunk优于文件已经被删除的chunk
+ 阻塞了客户端进程，也即正在使用的chunk优先级更高。

master节点也会定期进行系统的rebalance，而不是新增一台chunk server后就把所有新的chunk创建在这台机器上。这会造成短时间内的写入峰值。

## Garbage Collection

GFS实行定期垃圾回收机制，已删除的文件会被master标记（按特殊格式重命名），在常规扫描中发现存在了三天以上的这种文件就会被删除，它的metadata也会被相应清除。扫描中还会清除不再被任何文件指向的chunk。这些信息master会和chunk server在心跳包中交互，此时chunk server可删除对应的chunk。

Master节点还会为chunk存储version信息，用于检测已经过时的chunk副本并进行垃圾回收。由于垃圾回收是定时的，因此master在返回给client对应的chunk信息时也会带上版本，防止client访问到过时副本。

# Fail Tolerance and Diagnosis

为了保障系统的高可用性，GFS的两大策略是快速恢复和使用副本。所有服务器都能快速地保存状态并在几秒钟内重启。而如果硬件故障，不管是chunk server还是master节点都有副本可以快速恢复。master节点的“影子节点”还可以充当只读节点，提供稍微滞后于主节点的信息，提高系统的读性能。

为了保证数据一致性，数据不会在多个副本之间进行比较，而是通过校验和进行本地校验：

+ 在收到读请求时，chunk会在计算检查校验和后再发出数据。
+ 在append操作时，会对最后一块数据校验和进行添加，并计算产生的新数据块的校验和，即使原本最后一块数据已经有问题，那么在下一次读的时候依然能发现。
+ 对应随机write操作，则必须先读取并检查写操作所覆盖的第一片和最后一片数据的校验和，然后进行写入，最后再重新计算校验和进行更新。否则新的校验和会覆盖已经产生的数据错误。

> 为什么append不需要先读取并检查校验和，write需要？：append操作对应的位置其实是空的，可以增量式添加校验和。write本质是覆盖写，需要重新计算校验和。如果不预先检查，则会覆盖原来已经和数据不一致的校验和。

而由于大部分校验和检查发生在读数据时，系统也会定时检查那些不活跃的分块，以免在长时间不活动后失效分片越来越多。

整个系统的诊断工具，最主要依靠的就是各服务器详细地打印出日志。顺序并异步地打印日志不会造成什么性能损失，但是可以用于分析整个系统的流程和性能。

# Measurements

在这一章，论文简单介绍了GFS系统的实验情况。在小规模benchmark实验中，读速率和append速率都与预期相符，write速率稍低，也反映了GFS适合的任务场景。

在真实Google数据集中，实验数据更随机，但也反映了系统读多于写，append多于随机write的特点，反映了系统能很好地利用带宽，master节点不会成为瓶颈，在机器故障时数据恢复的速度也令人满意。

在阅读这一部分的时候，需要稍微注意以下bps和MB之间的8倍数量转换，以及一些小的数学计算。

论文也说明了真实场景的实验数据有一定的偏颇，因为这是GFS系统和Google应用相互调节适配的结果。但GFS仍然适用于更加通用的分布式大数据存储场景。

> "the ratio of writes to record appends is 108:1 by bytes transferred"，这句话的比例是否写反了？

# Conclusion

读完这篇文章，我感触最深的是以下两点：

+ 作为一个分布式存储系统，GFS使用了一个Master节点来居中调度。这看起来没有那种纯粹的数学美感，似乎不能显示出系统设计者的水平。但是，这一方案却是实用而且廉价的。相比于单节点的问题，它带来了更多的收益。况且带有主节点的分布式集群同样能展现摒除了复杂算法的简洁美。
+ 一个方案应该解决一类具体的问题。我们可以提前“优化”，可以考虑一个放之四海而皆准的框架，但是这个世界上能解决一类具体问题的方案就已经是一个好方案了。GFS的设计处处针对着Google的具体应用。定位精准的同时，保留了一定的扩展性。大一统理论还是留给爱因斯坦吧。

# Six Questions

**这个技术出现的背景、初衷和要达到什么样的目标或是要解决什么样的问题。**

GFS是脱胎于Google公司内部的一个分布式文件系统。它的设计重点就是契合于Google内部的一系列文件存储的需求，而谷歌的数据最大的特点是数据量非常之大。其他需求还包括支持高可用，支持高并发等。

**这个技术的优势和劣势分别是什么，或者说，这个技术的trade-off是什么。**

相比于在GFS之前的分布式文件系统，我认为GFS的最大优势是其允许将整个系统构建在廉价的服务器之上。据吴军博士的介绍，GFS加上MapReduce技术，使得Google得以“用大量的廉价服务器来代替超级计算机，而前者的价格不到后者的1/5，这也使得Google的运营成本比微软和雅虎低得多”。在当时“太阳公司的CEO麦克尼利嘲笑Google这种廉价的服务器集群是‘3M胶带纸’粘起来的。但是就是这些‘3M胶带纸粘起来的’廉价服务器让生产高大上服务器的太阳公司关了门”。

而GFS最大的劣势，一是其严格限制了使用场景。因为它的很多设计和优化都是根据使用场景来进行的，那么这个文件系统就失去了通用性。最大的限制就是对读写场景的限制：顺序读，顺序写。另外其设计使得只有在上规模的系统中才能充分展现其优势，否则呈现的更多是其复杂的设计，包括复杂的流程设计和调度，以及其中对文件进行分片，存储索引信息等引起的空间浪费。

另外还有一些小的trade-off，例如使用中心的master节点代替纯分布式设计，使用可能的master单点瓶颈交换了纯分布式系统设计的复杂性。

**这个技术使用的场景。**

GFS本质上是一个分布式文件系统。论文的Assumption环节已经详细阐述了GFS的使用场景，这里我再用自己的语言整理一下：

1. 低成本的硬件：系统构建于一批廉价硬件之上，其良好的高可用性就是针对这一点进行设计。 
2. 大文件存储：一般的文件大小会达到100MB级别，总的文件数量在M级别。因此像视频流分片之类的存储就不适用。这类文件可能需要在上层的业务逻辑中进行整合。
3. 流式读取：GFS最喜欢的读操作还是流式读取一整个大文件，而非随机读取，也只支持小规模的随机读取。
4. 顺序写入：于流式读取相对应，GFS最喜欢不断地进行Append写，写入后很少进行修改。
5. 支持高并发：这一点是GFS自身的原子操作设计实现的。
6. 支持持续高带宽，而非低延时：GFS本质上还是一个文件存储系统，本质上作文件传输是其最主要的使命。又由于其上存储大文件的特性，持续保持高带宽更为重要。

**技术的组成部分和关键点。**

GFS最主要的组成部分就是两部分的实现：GFS master和GFS chunkserver。

**技术的底层原理和关键实现。**

GFS底层实际上是通过软件控制的方式使用磁盘来进行备份，实现其高可用性，单纯的磁盘相较于磁盘阵列要更加简单，也更加便宜。其中最主要的关键实现就是数据流的控制和数据一致性的保证。

**已有的实现和它之间的对比。**

根据论文的Related Work描述，GFS的主要比较者有：AFS，xFS，Swift，Frangipani，Intermezzo，Minnesota's GFS，GPFS，Lustre，NASD等。

与这些系统相比：

1. GFS类似AFS提供了与位置无关的命名空间
2. GFS数据流的过程更像xFS和Swift，不像AFS。
3. GFS只使用磁盘做备份，不像xFS和Swift需要使用复杂的RAID（磁盘阵列）
4. 与AFS，xFS，Frangipani，Intermezzo相比，GFS不做缓存，因为它的使用场景是流式读取，跑一次一般整个文件只读一遍
5. GFS有一个中心master服务器，而Frangipani，xFS，Minnesota's GFS和GPFS是纯依赖于分布式算法的系统。
6. 与Lustre相同，GFS没有实现POSIX文件系统。
7. GFS与NASD的架构很相像，不过GFS基于普通的chunkserver机器，使用固定大小的chunk，同时比NASD实现了更多功能，例如重平衡，快照，数据恢复等，更适用于生产环境。
8. 与Minnesota's GFS和NASD相比，GFS不支持修改存储设备的模型。
9. 在数据传输队列中，与River相比，River使用内存队列，支持m-n的分布式队列，但不保证高可用性。GFS只支持m-1的队列，但实现起来高效可靠。
