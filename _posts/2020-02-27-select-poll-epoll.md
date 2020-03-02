---
layout: post
title: 📔【操作系统】I/O 多路复用，select / poll / epoll 详解
date: 2020/02/27 10:00
---

## 简介
阻塞 I/O 与非阻塞 I/O：
* 阻塞 I/O，即进程发起调用后，会被挂起（阻塞），直到收到数据再返回。缺点：如果调用一直不返回，进程就会一直被挂起
* 非阻塞 I/O 不会将进程挂起，调用时会立即返回成功或错误，优点是如果调用一直不成功，进程可以主动放弃这次调用，执行别的操作。缺点是非阻塞 I/O 需要不断轮询返回值，直到返回成功，这样效率很低

上述两种 I/O 方式每次调用时，只能检查一个文件描述符是否就绪。而 I/O 多路复用可以一次性检查**多个文件描述符**是否就绪，其中**任何**一个描述符就绪，就返回并执行后面的代码。I/O 多路复用在检查每个描述符的就绪状态时，使用的是非阻塞 I/O。

进程可以通过 select、poll、epoll 发起 I/O 多路复用的系统调用，这些系统调用都是同步阻塞的：**如果传入的多个文件描述符中，有任何一个描述符就绪，就返回；否则如果所有文件描述符都未就绪，就阻塞调用进程，直到某个描述符就绪，或者阻塞时长超过设置的 timeout 后，再返回**。

如果 `timeout` 参数设为 NULL，会无限阻塞直到某个描述符就绪；如果 `timeout` 参数设为 0，会立即返回，不阻塞。

I/O 多路复用引入了一些额外的操作和开销，性能更差。但是好处是用户可以在一个线程内同时处理多个 I/O 请求。如果不采用 I/O 多路复用，则必须通过多线程的方式，每个线程处理一个 I/O 请求。后者线程切换也是有一定的开销的。这部分内容可以查看最下文[Redis 的线程模型](#redis-的线程模型)。

### 为什么 I/O 多路复用需要使用非阻塞 I/O
I/O 多路复用内部会遍历集合中的每个文件描述符，判断其是否就绪：
```js
for fd in read_set
    if（ readable(fd) ) // 判断 fd 是否就绪
        count++
        FDSET(fd, &res_rset) // 将 fd 添加到就绪集合中
        break
...
return count
```

这里的 `readable(fd)` 就是一个非阻塞 I/O 调用。试想，如果这里使用阻塞 I/O，那么 `fd` 未就绪时，`select` 会阻塞在这个文件描述符上，无法检查下个文件描述符。

## select
### 函数签名与参数
```c
int select(int nfds,
            fd_set *restrict readfds,
            fd_set *restrict writefds,
            fd_set *restrict errorfds,
            struct timeval *restrict timeout);
```

`readfds`、`writefds`、`errorfds` 是三个文件描述符集合。`select` 会遍历每个集合的前 `nfds` 个描述符，分别找到可以读取、可以写入、发生错误的描述符，统称为“就绪”的描述符。然后用找到的子集替换参数中的对应集合，返回所有就绪描述符的总数。

`timeout` 参数表示调用 `select` 时的阻塞时长。如果所有文件描述符都未就绪，就阻塞调用进程，直到某个描述符就绪，或者阻塞超过设置的 timeout 后，返回。如果 `timeout` 参数设为 NULL，会无限阻塞直到某个描述符就绪；如果 `timeout` 参数设为 0，会立即返回，不阻塞。

### 什么是文件描述符 fd
文件描述符（file descriptor）是一个非负整数，从 0 开始。进程使用文件描述符来标识一个打开的文件。

系统为每一个进程维护了一个文件描述符表，表示该进程打开文件的记录表，而**文件描述符实际上就是这张表的索引**。当进程打开（`open`）或者新建（`create`）文件时，内核会在该进程的文件描述符表中新增一个表项，同时返回一个文件描述符 —— 也就是新增表项的下标。

一般来说，每个进程最多可以打开 64 个文件，`fd ∈ 0~63`。在不同系统上，最多允许打开的文件个数不同，Linux 2.4.22 强制规定最多不能超过 1,048,576。

### socket 与 fd 的关系
socket 是 Unix 中的术语。socket 可以用于同一台主机的不同进程间的通信，也可以用于不同主机间的通信。一个 socket 包含地址、类型和通信协议等信息，通过 `socket()` 函数创建：
```c
int socket(int domain, int type, int protocol)
```

返回的就是这个 socket 对应的文件描述符 `fd`。

可以这样理解：**socket 是进程间通信规则的高层抽象，而 fd 提供的是底层的具体实现**。socket 与 fd 是一一对应的。**通过 socket 通信，实际上就是通过文件描述符 `fd` 读写文件**。这也符合 Unix“一切皆文件”的哲学。

后面可以将 socket 和 fd 视为同义词。

### fd_set 文件描述符集合
参数中的 `fd_set` 类型表示文件描述符的集合。

由于文件描述符 `fd` 是一个从 0 开始的无符号整数，所以可以使用 `fd_set` 的**二进制每一位**来表示一个文件描述符。某一位为 1，表示对应的文件描述符已就绪。比如比如设 `fd_set` 长度为 1 字节，则一个 `fd_set` 变量最大可以表示 8 个文件描述符。当 `select` 返回 `fd_set = 00010011` 时，表示文件描述符 `1`、`2`、`5` 已经就绪。

`fd_set` 的使用涉及以下几个 API：
```c
#include <sys/select.h>   
int FD_ZERO(int fd, fd_set *fdset);  // 将 fd_set 所有位置 0
int FD_CLR(int fd, fd_set *fdset);   // 将 fd_set 某一位置 0
int FD_SET(int fd, fd_set *fd_set);  // 将 fd_set 某一位置 1
int FD_ISSET(int fd, fd_set *fdset); // 检测 fd_set 某一位是否为 1
```

### select 使用示例
下图的代码说明：
1. 先声明一个 `fd_set` 类型的变量 `readFDs`
2. 调用 `FD_ZERO`，将 `readFDs` 所有位置 0
3. 调用 `FD_SET`，将 `readFDs` 感兴趣的位置 1，表示要监听这几个文件描述符
4. 将 `readFDs` 传给 `select`，调用 `select`
5. `select` 会将 `readFDs` 中就绪的位置 1，未就绪的位置 0，返回就绪的文件描述符的数量
6. 当 `select` 返回后，调用 `FD_ISSET` 检测给定位是否为 1，表示对应文件描述符是否就绪

比如进程想监听 1、2、5 这三个文件描述符，就将 `readFDs` 设置为 `00010011`，然后调用 `select`。

如果 `fd=1`、`fd=2` 就绪，而 `fd=5` 未就绪，`select` 会将 `readFDs` 设置为 `00000011` 并返回 2。

如果每个文件描述符都未就绪，`select` 会阻塞 `timeout` 时长，再返回。这期间，如果 `readFDs` 监听的某个文件描述符上发生可读事件，则 `select` 会将对应位置 1，并立即返回。

![15732186159520](/media/15732186159520.jpg)


### select 的缺点
1. 性能开销大
    1. 调用 `select` 时会陷入内核，这时需要将参数中的 `fd_set` 从用户空间拷贝到内核空间
    2. 内核需要遍历传递进来的所有 `fd_set` 的每一位
2. 同时能够监听的文件描述符数量太少。受限于 `sizeof(fd_set)` 的大小，在编译内核时就确定了且无法更改。一般是 1024，不同的操作系统不相同

## poll
poll 和 select 几乎没有区别。poll 采用链表的方式存储文件描述符，没有最大存储数量的限制。

从性能开销上看，poll 和 select 的差别不大。

## epoll
epoll 是对 select 和 poll 的改进，避免了“性能开销大”和“文件描述符数量少”两个缺点。

select、poll 模型都只使用一个函数，而 epoll 模型使用三个函数：`epoll_create`、`epoll_ctl` 和 `epoll_wait`。

### epoll_create
```c
int epoll_create(int size);
```
`epoll_create` 会创建一个 `epoll` 实例，同时返回一个引用该实例的文件描述符。

返回的文件描述符仅仅指向对应的 `epoll` 实例，并不表示真实的磁盘文件节点。其他 API 如 `epoll_ctl`、`epoll_wait` 会使用这个文件描述符来操作相应的 `epoll` 实例。

当创建好 epoll 句柄后，它就是会占用一个 fd 值，在 linux 下如果查看 `/proc/进程id/fd/`，是能够看到这个 fd 的。所以在使用完 epoll 后，必须调用 `close(epfd)` 关闭对应的文件描述符，否则可能导致 fd 被耗尽。当指向同一个 `epoll` 实例的所有文件描述符都被关闭后，操作系统会销毁这个 `epoll` 实例。

`epoll` 实例内部存储：
* 监听列表：所有要监听的文件描述符，使用红黑树
* 就绪列表：所有就绪的文件描述符，使用链表

### epoll_ctl
```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```
`epoll_ctl` 会监听文件描述符 `fd` 上发生的 `event` 事件。

参数说明：
* `epfd` 即 `epoll_create` 返回的文件描述符，指向一个 `epoll` 实例
* `fd` 表示要监听的目标文件描述符
* `event` 表示要监听的事件（可读、可写、发送错误...）
* `op` 表示要对 `fd` 执行的操作，有以下几种：
    * `EPOLL_CTL_ADD`：为 `fd` 添加一个监听事件 `event`
    * `EPOLL_CTL_MOD`：Change the event event associated with the target file descriptor fd（`event` 是一个结构体变量，这相当于变量 `event` 本身没变，但是更改了其内部字段的值）
    * `EPOLL_CTL_DEL`：删除 `fd` 的所有监听事件，这种情况下 `event` 参数没用

返回值 0 或 -1，表示上述操作成功与否。

`epoll_ctl` 会将文件描述符 `fd` 添加到 `epoll` 实例的监听列表里，同时为 `fd` 设置一个回调函数，并监听事件 `event`。当 `fd` 上发生相应事件时，会调用回调函数，将 `fd` 添加到 `epoll` 实例的就绪队列上。

### epoll_wait
```c
int epoll_wait(int epfd, struct epoll_event *events,
               int maxevents, int timeout);
```
这是 epoll 模型的主要函数，功能相当于 `select`。

参数说明：
* `epfd` 即 `epoll_create` 返回的文件描述符，指向一个 `epoll` 实例
* `events` 是一个数组，保存就绪状态的文件描述符，其空间由调用者负责申请
* `maxevents` 指定 `events` 的大小
* `timeout` 类似于 `select` 中的 timeout。如果没有文件描述符就绪，即就绪队列为空，则 `epoll_wait` 会阻塞 timeout 毫秒。如果 timeout 设为 -1，则 `epoll_wait` 会一直阻塞，直到有文件描述符就绪；如果 timeout 设为 0，则 `epoll_wait` 会立即返回

返回值表示 `events` 中存储的就绪描述符个数，最大不超过 `maxevents`。

### epoll 的优点
一开始说，epoll 是对 select 和 poll 的改进，避免了“性能开销大”和“文件描述符数量少”两个缺点。

对于“文件描述符数量少”，select 使用整型数组存储文件描述符集合，而 epoll 使用红黑树存储，数量较大。

对于“性能开销大”，`epoll_ctl` 中为每个文件描述符指定了回调函数，并在就绪时将其加入到就绪列表，因此 epoll 不需要像 `select` 那样遍历检测每个文件描述符，只需要判断就绪列表是否为空即可。这样，在没有描述符就绪时，epoll 能更早地让出系统资源。
> 相当于时间复杂度从 O(n) 降为 O(1)

此外，每次调用 `select` 时都需要向内核拷贝所有要监听的描述符集合，而 epoll 对于每个描述符，只需要在 `epoll_ctl` 传递一次，之后 `epoll_wait`  不需要再次传递。这也大大提高了效率。

## 三者对比
* `select`：调用开销大（需要复制集合）；集合大小有限制；需要遍历整个集合找到就绪的描述符
* `poll`：poll 采用链表的方式存储文件描述符，没有最大存储数量的限制，其他方面和 select 没有区别
* `epoll`：调用开销小（不需要复制）；集合大小无限制；采用回调机制，不需要遍历整个集合

此外 `select` 只支持水平触发，`epoll` 支持边缘触发。

### 什么是水平触发、边缘触发
水平触发（LT，Level Trigger）：当文件描述符就绪时，会触发通知，如果用户程序没有一次性把数据读写完，下次还会通知。

边缘触发（ET，Edge Trigger）：仅当描述符从未就绪变为就绪时，通知一次，之后不会再通知。

区别：边缘触发效率更高，减少了被重复触发的次数，函数不会返回大量用户程序可能不需要的文件描述符。

## 适用场景
当连接数较多并且有很多的不活跃连接时，epoll 的效率比其它两者高很多。当连接数较少并且都十分活跃的情况下，由于 epoll 需要很多回调，因此性能可能低于其它两者。

## Redis 的线程模型
Redis 是一个单线程的工作模型，使用 I/O 多路复用来处理客户端的多个连接。为什么 Redis 选择单线程也能效率这么高？

I/O 设备（如磁盘、网络）等速度远远慢于 CPU，因此引入了多线程技术。当一个线程发起 I/O 请求时，先将它挂起，切换到别的线程；当 I/O 设备就绪时，再切换回该线程。总之，**多线程技术是为了充分利用 CPU 的计算资源，适用于下层存储慢速的场景**。

而 redis 是纯内存操作，读写速度非常快。所有的操作都会在内存中完成，不涉及任何 I/O 操作，因此**多线程频繁的上下文切换反而是一种负优化**。Redis 选择基于非阻塞 I/O 的 **I/O 多路复用机制**，在单线程里**并发**处理客户端的多个连接，减少多线程带来的系统开销，同时也有更好的可维护性，方便开发和调试。

不过 redis 在最新的几个版本中也引入了多线程，目的是：
1. 异步处理删除操作。当删除超大键值对的时候，单线程内同步地删除可能会阻塞待处理的任务
2. 应对网络 I/O 的场景，网络 I/O 是慢速 I/O。redis6 吞吐量提高了 1 倍

## 参考资料
* [Linux man page - epoll](https://linux.die.net/man/7/epoll)
* [Linux man page - epoll_create](https://linux.die.net/man/2/epoll_create)
* [Linux man page - epoll_ctl](https://linux.die.net/man/2/epoll_ctl)
* [Linux man page - epoll_wait](https://linux.die.net/man/2/epoll_wait)