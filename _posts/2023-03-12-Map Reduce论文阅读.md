---
layout: article
title: Map Reduce 论文阅读
tags: 分布式 论文阅读
mode: immersive
header:
  theme: dark
article_header:
  type: cover
  image:
    src: https://pictureloomione.oss-cn-beijing.aliyuncs.com/pic/MapReduce%20paper/map_reduce.png
aside:
  toc: true
---

本文为 Google 发表的[Map Reduce](https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf) 论文的阅读笔记, MapReduce 为一个分布式计算框架。

<!--more-->

---

<div>{%- include extensions/soundcloud.html id='291497062' -%}</div>

---

## MapReduce 模型与实现

**分布式系统面临的问题**: 如何**处理大规模数据集**。如何处理**并行计算**、如何**分发数据**、如何**处理错误**？_所有这些问题综合在一起，需要大量的代码处理，因此也使得原本简单的运算变得难以处理_。

**MapRecude 动机**: 设计一个抽象模型，使用这个抽象模型，_我们只要表述我们想要执行的简单运算即可_，而*不必关心并行计算、容错、数据分布、负载均衡等复杂的细节*。

**MapReduce 原理**: 利用一个输入 key/value pair 集合来产生一个输出的 key/value pair 集合。MapReduce 库的用户用两个函数表达这个计算：Map 和 Reduce。

- 用户自定义的 Map 函数接受一个输入的 key/value pair 值，然后产生一个中间 key/value pair 值的集合。MapReduce 库把所有具有相同中间 key 值 I 的中间 value 值集合在一起后传递给 reduce 函数。
- 用户自定义的 Reduce 函数接受一个中间 key 的值 I 和相关的一个 value 值的集合。Reduce 函数合并这些 value 值，形成一个较小的 value 值的集合。一般的，每次 Reduce 函数调用只产生 0 或 1 个输出 value 值。通常我们通过一个迭代器把中间 value 值提供给 Reduce 函数，这样我们就可以处理无法全部放入内存中的大量的 value 值的集合。

**MapReduce 执行**: MapReduce 库把输入数据集分割成 M 个大小相等的片段，Map 调用被分布到多台机器上执行。然后 Map 函数并行处理这些片段，使用**分区函数**将 Map 调用产生的中间 key 值分成 R 个不同分，产生 R 个中间文件。Reduce 函数并行处理这些中间文件，产生最终的输出文件。

<div  align="center">
<img src= "
https://pictureloomione.oss-cn-beijing.aliyuncs.com/pic/MapReduce%20paper/figure-1.png
"/>
</div>

1. 用户程序首先调用的 MapReduce 库将输入文件分成 M 个数据片度，每个数据片段的大小一般从
   16MB 到 64MB(可以通过可选的参数来控制每个数据片段的大小)。然后用户程序在机群中创建大量
   的程序副本。
2. 这些程序副本中的有一个特殊的程序–**master**。副本中其它的程序都是 worker 程序，由 master **分配任务**。有 M 个 Map 任务和 R 个 Reduce 任务将被分配，master 将一个 Map 任务或 Reduce 任务分配给一个空闲的 worker。
3. 被分配了 map 任务的 worker 程序读取相关的输入数据片段，从输入的数据片段中解析出 key/value
   pair，然后把 key/value pair 传递给用户自定义的 Map 函数，由 Map 函数生成并输出的中间 key/value
   pair，并缓存在内存中。
4. 缓存中的 key/value pair 通过分区函数**分成 R 个区域**，之后周期性的写入到本地磁盘上。缓存的
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

MapReduce 中的 Map 和 Reduce 任务被分为三种状态：**idle**、**in-progress**、**completed**。每个 worker 有两个状态：**idle**、**in-progress**。

- 对于一个 Map 任务来说，当一个 worker 从 master 那里得到一个 Map 任务时，这个任务的状态就变为 in-progress。当这个 worker 完成这个任务后，这个任务的状态就变为 completed。当 Map 任务完成后，中间文件的位置信息将被回传给 master，然后 master 会把这些信息传递给 Reduce worker。当 RReduce worker 完成这个任务后，这个任务的状态就变为 completed,同时执行 Map 的 worker 也会被标记为 idle。

**输入数据的存储位置**: 把输入数据(由 GFS 管理)存储在集群中机器的本地磁盘上来节省网络带宽。GFS 把每个文件按 64MB 一个 Block 分隔，每个 Block 保存在多台机器上，环境中就存放了多份拷贝(一般是 3 个拷贝)， 因此在 Map 的任务调度时，会尽量将一个 Map 任务调度在包含相关输入数据拷贝的机器上执行；

**备用任务**: 根据木桶效应，如果一个任务的执行时间远远大于其它任务，那么这个任务就会成为整个 MapReduce 运算的瓶颈。当一个 MapReduce 操作接近完成的时候，master
调度备用（backup）任务进程来执行剩下的、处于处理中状态（in-progress）的任务。无论是最初的执行进程、还是备用（backup）任务进程完成了任务，我们都把这个任务标记成为已经完成。

## MapReduce 容错

### worker 故障

**master 周期性的 ping 每个 worker**。如果在一个约定的时间范围内没有收到 worker 返回的信息，master 将
把这个 worker **标记为失效**。所有由这个失效的 worker 完成的 Map 任务被重设为初始的空闲状态，之后这些
任务就可以被安排给其他的 worker。同样的，worker 失效时正在运行的 Map 或 Reduce 任务也将被重新置为
**空闲状态**，等待**重新调度**。

当 worker 故障时，由于已经完成的 Map 任务的输出存储在这台机器上，Map 任务的输出已不可访问了，因此必须重新执行。而已经完成的 Reduce 任务的输出存储在全局文件系统上，因此不需要再次执行。

### master 失效

如果 master 失效，就中止 MapReduce 运算。客户可以检查到这个状态，并且可以根据需要重新执行 MapReduce
操作。

### MapReduce 的原子性

当用户提供的 Map 和 Reduce 操作是输入确定性函数（即相同的输入产生相同的输出）时，分布式实现在任何情况下的输出都和所有程序没有出现任何错误、顺序的执行产生的输出是一样的。这依赖对 Map 和 Reduce 任务的输出是原子提交的来完成这个特性。

## 问题记录

在 3.3.3 节中，论文提到了一个问题：如果同一个 Reduce 任务在多台机器上执行，针对同一个最终的输出文件将有多个重命名操作执行。我们依赖底层文件系统提供的重命名操作的原子性来保证最终的文件系统状态仅仅包含一个 Reduce 任务产生的数据。**为什么会出想同一个 reduce 出现子啊多台机器上执行的情况？？**

> 可能的一个原因就是备用任务的出现，由于有 Reduce 任务成为了整个系统的瓶颈，那么就会调度备用任务来执行这个 Reduce 任务，这样就会出现同一个 Reduce 任务在多台机器上执行的情况。

```cpp
#include "mapreduce/mapreduce.h"

// User’s map function
class WordCounter : public Mapper {
  public:
    virtual void Map(const MapInput& input) {
      const string& text = input.value();
      const int n = text.size();
      for (int i = 0; i < n; ) {
        // Skip past leading whitespace
        while ((i < n) && isspace(text[i]))
          i++;

        // Find word end
        int start = i;
        while ((i < n) && !isspace(text[i]))
          i++;

        if (start < i)
          Emit(text.substr(start,i-start),"1");
      }
  }
};
REGISTER_MAPPER(WordCounter);

// User’s reduce function
class Adder : public Reducer {
  virtual void Reduce(ReduceInput* input) {
    // Iterate over all entries with the
    // same key and add the values
    int64 value = 0;
    while (!input->done()) {
      value += StringToInt(input->value());
      input->NextValue();
    }

    // Emit sum for input->key()
    Emit(IntToString(value));
  }
};
REGISTER_REDUCER(Adder);

int main(int argc, char** argv) {
  ParseCommandLineFlags(argc, argv);

  MapReduceSpecification spec;

  // Store list of input files into "spec"
  for (int i = 1; i < argc; i++) {
    MapReduceInput* input = spec.add_input();
    input->set_format("text");
    input->set_filepattern(argv[i]);
    input->set_mapper_class("WordCounter");
  }

  // Specify the output files:
  // /gfs/test/freq-00000-of-00100
  // /gfs/test/freq-00001-of-00100
  // ...
  MapReduceOutput* out = spec.output();
  out->set_filebase("/gfs/test/freq");
  out->set_num_tasks(100);
  out->set_format("text");
  out->set_reducer_class("Adder");

  // Optional: do partial sums within map
  // tasks to save network bandwidth
  out->set_combiner_class("Adder");

  // Tuning parameters: use at most 2000
  // machines and 100 MB of memory per task
  spec.set_machines(2000);
  spec.set_map_megabytes(100);
  spec.set_reduce_megabytes(100);

  // Now run it
  MapReduceResult result;
  if (!MapReduce(spec, &result)) abort();

  // Done: ’result’ structure contains info
  // about counters, time taken, number of
  // machines used, etc.

  return 0;
}
```
