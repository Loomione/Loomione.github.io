---
title: Map Reduce 论文阅读
tags: 分布式 论文阅读
aside:
  toc: true
---

本文为 Google 发表的[Map Reduce](https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf) 论文的阅读笔记。

<!--more-->

## Map Reduce 模型与实现

<div>{%- include extensions/soundcloud.html id='2139367135' -%}</div>

分布式系统的一个重要问题是如何处理大规模数据集。如何处理**并行计算**、如何**分发数据**、如何**处理错误**？_所有这些问题综合在一起，需要大量的代码处理，因此也使得原本简单的运算变得难以处理_。

**MapRecude 动机**: 设计一个抽象模型，使用这个抽象模型，_我们只要表述我们想要执行的简单运算即可_，而*不必关心并行计算、容错、数据分布、负载均衡等复杂的细节*。

**MapReduce 原理**: 利用一个输入 key/value pair 集合来产生一个输出的 key/value pair 集合。MapReduce 库的用户用两个函数表达这个计算：Map 和 Reduce。

- 用户自定义的 Map 函数接受一个输入的 key/value pair 值，然后产生一个中间 key/value pair 值的集合。MapReduce 库把所有具有相同中间 key 值 I 的中间 value 值集合在一起后传递给 reduce 函数。
- 用户自定义的 Reduce 函数接受一个中间 key 的值 I 和相关的一个 value 值的集合。Reduce 函数合并这些 value 值，形成一个较小的 value 值的集合。一般的，每次 Reduce 函数调用只产生 0 或 1 个输出 value 值。通常我们通过一个迭代器把中间 value 值提供给 Reduce 函数，这样我们就可以处理无法全部放入内存中的大量的 value 值的集合。

**MapReduce 执行**: MapReduce 库把输入数据集分割成 M 个大小相等的片段，Map 调用被分布到多台机器上执行。然后 Map 函数并行处理这些片段，使用**分区函数**将 Map 调用产生的中间 key 值分成 R 个不同分，产生 R 个中间文件。Reduce 函数并行处理这些中间文件，产生最终的输出文件。

<div  align="center">
<img src= "
https://pictureloomione.oss-cn-beijing.aliyuncs.com/pic/MapReduce%20paper/map_reduce.png
"/>
</div>

1. 用户程序首先调用的 MapReduce 库将输入文件分成 M 个数据片度，每个数据片段的大小一般从
   16MB 到 64MB(可以通过可选的参数来控制每个数据片段的大小)。然后用户程序在机群中创建大量
   的程序副本。
2. 这些程序副本中的有一个特殊的程序–**master**。副本中其它的程序都是 worker 程序，由 master **分配任务**。有 M 个 Map 任务和 R 个 Reduce 任务将被分配，master 将一个 Map 任务或 Reduce 任务分配给一个空闲的 worker。
3. 被分配了 map 任务的 worker 程序读取相关的输入数据片段，从输入的数据片段中解析出 key/value
   pair，然后把 key/value pair 传递给用户自定义的 Map 函数，由 Map 函数生成并输出的中间 key/value
   pair，并缓存在内存中。
4. 缓存中的 key/value pair 通过分区函数分成 R 个区域，之后周期性的写入到本地磁盘上。缓存的
   key/value pair 在本地磁盘上的存储位置将被回传给 master，由 master 负责把这些存储位置再传送给
   Reduce worker。
5. 当 Reduce worker 程序接收到 master 程序发来的数据存储位置信息后，使用 RPC 从 Map worker 所在
   主机的磁盘上读取这些缓存数据。当 Reduce worker 读取了所有的中间数据后，通过对 key 进行**排序**
   后使得具有相同 key 值的数据聚合在一起。由于许多不同的 key 值会映射到相同的 Reduce 任务上，
   因此必须进行排序。如果中间数据太大无法在内存中完成排序，那么就要在外部进行排序。
6. Reduce worker 程序遍历排序后的中间数据，对于每一个唯一的中间 key 值，Reduce worker 程序将这
   个 key 值和它相关的中间 value 值的集合传递给用户自定义的 Reduce 函数。Reduce 函数的输出被追
   加到所属分区的输出文件。
7. 当所有的 Map 和 Reduce 任务都完成之后，master 唤醒用户程序。在这个时候，在用户程序里的对
   MapReduce 调用才返回。

<div  align="center">
<img src= "
https://pictureloomione.oss-cn-beijing.aliyuncs.com/pic/MapReduce%20paper/figure-1.png
"/>
</div>

## MapReduce 容错

### worker 故障

**master 周期性的 ping 每个 worker**。如果在一个约定的时间范围内没有收到 worker 返回的信息，master 将
把这个 worker 标记为失效。所有由这个失效的 worker 完成的 Map 任务被重设为初始的空闲状态，之后这些
任务就可以被安排给其他的 worker。同样的，worker 失效时正在运行的 Map 或 Reduce 任务也将被重新置为
空闲状态，等待重新调度。

当 worker 故障时，由于已经完成的 Map 任务的输出存储在这台机器上，Map 任务的输出已不可访问了，因此必须重新执行。而已经完成的 Reduce 任务的输出存储在全局文件系统上，因此不需要再次执行。

### master 失效

如果 master 失效，就中止 MapReduce 运算。客户可以检查到这个状态，并且可以根据需要重新执行 MapReduce
操作。
