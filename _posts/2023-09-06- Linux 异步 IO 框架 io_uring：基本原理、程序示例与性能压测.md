---
title: Linux 异步 IO 框架 io_uring：基本原理、程序示例与性能压测
tags: 异步IO
aside:
  toc: true
---

[Linux 异步 I/O 框架 **io_uring**：基本原理、程序示例与性能压测（2020）](http://arthurchiao.art/blog/intro-to-io-uring-zh/#%E8%AF%91%E8%80%85%E5%BA%8F)[转]

<!--more-->

## 一、Linux I/O 系统调用演进

### 1. 基于 fd 的阻塞式 I/O：read()/write()

&emsp;&emsp; 作为大家最熟悉的读写方式，Linux 内核提供了 **<font color = red>基于文件描述符的系统调用</font>**， 这些描述符指向的可能是存储文件**storage file**，也可能是 **network sockets**：

```c
ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);
```

&emsp;&emsp; 二者称为 **<font color = red>阻塞式系统调用</font>**（blocking system calls），因为程序调用 这些函数时会进入 sleep 状态，然后被调度出去（让出处理器），直到 I/O 操作完成：

- 如果数据在文件中，并且文件内容已经缓存在 page cache 中，调用会**立即返回**；
- 如果数据在另一台机器上，就需要通过网络（例如 TCP）获取，会**阻塞**一段时间；
- 如果数据在硬盘上，也会**阻塞**一段时间。

&emsp;&emsp; 但很容易想到，随着存储设备越来越快，程序越来越复杂， 阻塞式（blocking）已经这种最简单的方式已经不适用了。

### 2. 非阻塞式 I/O：select()/poll()/epoll()

&emsp;&emsp;阻塞式之后，出现了一些新的、非阻塞的系统调用，例如 **select()**、**poll()** 以及更新的 **epoll()**。 应用程序在调用这些函数读写时不会阻塞，而是立即返回，返回的是一个 已经 ready 的文件描述符列表。

<div  align="center">
<img src= "
http://arthurchiao.art/assets/img/intro-to-io-uring/epoll.png" width = "300" height = "200"/>
</div>

&emsp;&emsp;但这种方式存在一个 **<font color = red>致命缺点</font>**：只支持 network sockets 和 pipes —— epoll() 甚至连 storage files 都不支持。

### 3. 线程池方式

&emsp;&emsp;对于 **storage I/O**，经典的解决思路是 **<font color = red>thread pool</font>**： 主线程将 I/O 分发给 worker 线程，后者代替主线程进行阻塞式读写，主线程不会阻塞。

<div  align="center">
<img src= "
https://pictureloomione.oss-cn-beijing.aliyuncs.com/pic/thread-pools.png
" width = "300" height = "300"/>
</div>

&emsp;&emsp;这种方式的问题是<u color = red>线程上下文切换开销可能非常大</u>，后面性能压测会看到。

### 4. Direct I/O（数据库软件）：绕过 page cache

&emsp;&emsp;随后出现了更加灵活和强大的方式：数据库软件（database software） 有时 并不想使用操作系统的 page cache， 而是希望打开一个文件后，直接从设备读写这个文件（direct access to the device）。 这种方式称为 **<font color = red>直接访问（direct access）</font>**或直接 I/O（direct I/O）：

- 需要指定 **<font color = red>O_DIRECT flag</font>**；
- 需要应用自己 **<font color = red>管理自己的缓存</font>** —— 这正是数据库软件所希望的；
- 是 zero-copy I/O，因为应用的缓冲数据直接发送到设备，或者直接从设备读取。

### 5. 异步 IO（AIO）

&emsp;&emsp;前面提到，**随着存储设备越来越快，主线程和 worker 线性之间的上下文切换开销占比越来越高**。 现在市场上的一些设备，例如 Intel Optane ，延迟已经低到和上下文切换一个量级（微秒 us）。换个方式描述， 更能让我们感受到这种开销： 上下文每切换一次，我们就少一次 dispatch I/O 的机会。

&emsp;&emsp;因此，Linux 2.6 内核引入了 **<font color = red>异步 I/O（asynchronous I/O）</font>**接口， 方便起见，本文简写为 linux-aio。AIO 原理是很简单的：

- 用户通过 **<font color = red>io_submit()</font>** 提交 I/O 请求，
- 过一会再调用 **<font color = red>io_getevents()</font>** 来检查哪些 events 已经 ready 了。
- 使程序员能编写完全异步的代码。

&emsp;&emsp;近期，**Linux AIO 甚至支持了 epoll()**：也就是说 **<font color = red>不仅能提交 storage I/O 请求，还能提交网络 I/O 请求</font>**。照这样发展下去，linux-aio 似乎能成为一个王者。但由于它糟糕的演进之路，这个愿望几乎不可能实现了。 我们从 Linus 标志性的激烈言辞中就能略窥一斑：

> Reply to: to support opening files asynchronously<br>
>
> So I think this is ridiculously ugly.<br>
>
> AIO is a horrible ad-hoc design, with the main excuse being “other, less gifted people, made that design, and we are implementing it for compatibility because database people — who seldom have any shred of taste — actually use it”.<br>
>
> — Linus Torvalds (on lwn.net)

&emsp;&emsp;首先，作为数据库从业人员，我们想借此机会为我们的没品（lack of taste）向 Linus 道歉。 但更重要的是，我们要进一步解释一下为什么 Linus 是对的：**<font color = red>Linux AIO 确实问题缠身</font>**，

1. 只支持 O_DIRECT 文件，因此对常规的非数据库应用 （normal, non-database applications）几乎是无用的；
2. 接口在设计时并未考虑扩展性。虽然可以扩展 —— 我们也确实这么做了 —— 但每加一个东西都相当复杂；
3. 虽然从技术上说接口是非阻塞的，但实际上有 很多可能的原因都会导致它阻塞，而且引发的方式难以预料。

### 6. 小结

以上可以清晰地看出 Linux I/O 的演进：

- 最开始是同步（阻塞式）系统调用；
- 然后随着实际需求和具体场景，不断加入新的异步接口，还要保持与老接口的兼容和协同工作。

另外也看到，在非阻塞式读写的问题上并没有形成统一方案：

1. Network socket 领域：添加一个异步接口，然后去轮询（poll）请求是否完成（readiness）；
2. Storage I/O 领域：只针对某一细分领域（数据库）在某一特定时期的需求，添加了一个定制版的异步接口。

这就是 Linux I/O 的演进历史 —— 只着眼当前，出现一个问题就引入一种设计，而并没有多少前瞻性 —— 直到 **io_uring** 的出现。

## 二、io_uring

**io_uring** 来自资深内核开发者 Jens Axboe 的想法，他在 Linux I/O stack 领域颇有研究。 从最早的 patch aio: support for IO polling 可以看出，这项工作始于一个很简单的观察：**随着设备越来越快， 中断驱动（interrupt-driven）模式效率已经低于轮询模式 （polling for completions）** —— 这也是高性能领域最常见的主题之一。

**io_uring** 的基本逻辑与 **linux-aio** 是类似的：**<font color = red>提供两个接口，一个将 I/O 请求提交到内核，一个从内核接收完成事件</font>**。
但随着开发深入，它逐渐变成了一个完全不同的接口：设计者开始从源头思考 如何支持完全异步的操作。

### 1.与 Linux AIO 的不同

**io_uring** 与 **linux-aio** 有着 **<font color = red>本质的不同</font>**：

在设计上是 **<font color = red>真正异步</font>**的（truly asynchronous）。只要设置了合适的 flag，它在系统调用上下文中就只是将请求放入队列，不会做其他任何额外的事情，**<font color = red>保证了应用永远不会阻塞</font>**。

支持任何类型的 I/O：cached files、direct-access files 甚至 blocking sockets。

由于设计上就是异步的（async-by-design nature），因此无需 poll+read/write 来处理 sockets。 只需提交一个阻塞式读（blocking read），请求完成之后，就会出现在 completion ring。

灵活、可扩展：基于 **io_uring** 甚至能重写（re-implement）Linux 的每个系统调用。

### 2. 原理及核心数据结构：SQ/CQ/SQE/CQE

每个 **io_uring** 实例都有两个环形队列（ring），在内核和应用程序之间共享：

**提交队列**：submission queue (SQ)
**完成队列**：completion queue (CQ)

<div  align="center">
<img src= "
https://pictureloomione.oss-cn-beijing.aliyuncs.com/pic/io_uring.png
" width = "400" height = "400"/>
</div>

这两个队列：

都是 **<font color = red>spsc</font>**，size 是 2 的幂次；
提供 **<font color = red>无锁接口</font>**（lock-less access interface），内部使用 **<font color = red>内存屏障</font>**做同步（coordinated with memory barriers）。
使用方式：

请求

应用创建 SQ entries (SQE)，更新 SQ tail；
内核消费 SQE，更新 SQ head。
完成

内核为完成的一个或多个请求创建 CQ entries (CQE)，更新 CQ tail；
应用消费 CQE，更新 CQ head。
完成事件（completion events）可能以任意顺序到达，到总是与特定的 SQE 相关联的。
消费 CQE 过程无需切换到内核态。

### 3. 带来的好处

**io_uring** 这种请求方式还有一个好处是：原来需要多次系统调用（读或写），现在变成批处理一次提交。

还记得 Meltdown 漏洞吗？当时我还写了一篇文章 解释为什么我们的 Scylla NoSQL 数据库受影响很小：aio 已经将我们的 I/O 系统调用批处理化了。

**io_uring** 将这种批处理能力带给了 storage I/O 系统调用之外的 其他一些系统调用，包括：

read
write
send
recv
accept
openat
stat
专用的一些系统调用，例如 fallocate
此外，**io_uring** 使异步 I/O 的使用场景也不再仅限于数据库应用，普通的 非数据库应用也能用。这一点值得重复一遍：

虽然 **io_uring** 与 aio 有一些相似之处，但它的扩展性和架构是革命性的： 它将异步操作的强大能力带给了所有应用（及其开发者），而 不再仅限于是数据库应用这一细分领域。

我们的 CTO Avi Kivity 在 the Core C++ 2019 event 上 有一次关于 async 的分享。 核心点包括：从延迟上来说，

现代多核、多 CPU 设备，其内部本身就是一个基础网络；
CPU 之间是另一个网络；
CPU 和磁盘 I/O 之间又是一个网络。
因此网络编程采用异步是明智的，而现在开发自己的应用也应该考虑异步。 这从根本上改变了 Linux 应用的设计方式：

之前都是一段顺序代码流，需要系统调用时才执行系统调用，
现在需要思考一个文件是否 ready，因而自然地引入 event-loop，不断通过共享 buffer 提交请求和接收结果。

### 4. 三种工作模式

**io_uring** 实例可工作在三种模式：

中断驱动模式（interrupt driven）

默认模式。可通过 **io_uring**\_enter() 提交 I/O 请求，然后直接检查 CQ 状态判断是否完成。

轮询模式（polled）

Busy-waiting for an I/O completion，而不是通过异步 IRQ（Interrupt Request）接收通知。

这种模式需要文件系统（如果有）和块设备（block device）支持轮询功能。 相比中断驱动方式，这种方式延迟更低（连系统调用都省了）， 但可能会消耗更多 CPU 资源。

目前，只有指定了 O_DIRECT flag 打开的文件描述符，才能使用这种模式。当一个读 或写请求提交给轮询上下文（polled context）之后，应用（application）必须调用 **io_uring**\_enter() 来轮询 CQ 队列，判断请求是否已经完成。

对一个 **io_uring** 实例来说，不支持混合使用轮询和非轮询模式。

内核轮询模式（kernel polled）

这种模式中，会 创建一个内核线程（kernel thread）来执行 SQ 的轮询工作。

使用这种模式的 **io_uring** 实例， 应用无需切到到内核态 就能触发（issue）I/O 操作。 通过 SQ 来提交 SQE，以及监控 CQ 的完成状态，应用无需任何系统调用，就能提交和收割 I/O（submit and reap I/Os）。

如果内核线程的空闲时间超过了用户的配置值，它会通知应用，然后进入 idle 状态。 这种情况下，应用必须调用 **io_uring**\_enter() 来唤醒内核线程。如果 I/O 一直很繁忙，内核线性是不会 sleep 的。

### 5. **io_uring** 系统调用 API

有三个：

**io_uring**\_setup(2)
**io_uring**\_register(2)
**io_uring**\_enter(2)
下面展开介绍。完整文档见 manpage。
