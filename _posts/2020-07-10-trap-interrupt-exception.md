---
layout: post
title: 📔【操作系统】陷阱、中断、异常、信号
date: 2020/7/15 9:00
permalink: 2020/07/09/trap-interrupt-exception.html
---

## 前言：异常控制流
每个进程对应的程序文件由一条一条的指令组成。进程在执行的时候，会将程序文件加载到进程的内存空间中，这些指令在内存空间中是相邻的。进程会通过调整程序计数器 PC 的值，一条一条地执行指令。我们将进程执行的指令序列叫做处理器的「控制流」。

正常情况下，进程可能会顺序执行相邻的指令（“平滑的”序列），也可能通过跳转、调用和返回等程序指令转移到另一个位置开始执行（平滑流的突变）。无论是前者还是后者，都是程序正常执行的结果，是可以预知的、符合预期的。

但是，系统中也会发生一些异常情况。处理这些异常时，会打断进程正常执行的控制流，转而执行相应的处理程序，执行完毕再返回。我们将这类突变称为「异常控制流」（Exceptional Control Flow，ECF）。

![-w471](/media/15948281929782.jpg){:width="500px"}

异常控制流的形式有以下几种：陷阱、中断与异常。此外还有信号，这是一种更高层的异常形式，也会改变进程的控制流。

## 陷阱（trap）
陷阱是**有意**造成的“异常”，是执行一条指令的结果。陷阱是同步的。

陷阱的主要作用是实现**系统调用**。比如，进程可以执行 `syscall n` 指令向内核请求服务。当进程执行这条指令后，会中断当前的控制流，**陷入**到内核态，执行相应的系统调用。内核的处理程序在执行结束后，会将结果返回给进程，同时退回到用户态。进程此时继续执行**下一条指令**。

每个系统调用有一个唯一的整数号，对应于内核中一个跳转表的偏移量。这个跳转表中的每个条目表示一个系统调用的代码位置。执行系统调用时，通过这个整数号作为跳转表的偏移量，就可以执行相应的系统调用。

![-w463](/media/15948286786784.jpg){:width="500px"}

陷阱也被称为“软中断”，它不是真正意义上的中断，但是和硬件中断的处理流程类似。陷阱指令的细节见下文[系统调用表](#system-call)。

## 中断（interrupt）
中断由处理器**外部**的**硬件**产生，不是执行某条指令的结果，也无法预测发生时机。由于中断独立于当前执行的程序，因此中断是异步事件。

中断包括 I/O 设备发出的 I/O 中断、各种定时器引起的时钟中断、调试程序中设置的断点等引起的调试中断等。

每个中断都有一个中断号。操作系统使用**中断描述符表**（Interrupt Descriptor Table，IDT）来保存每个中断的中断处理程序的地址。当发生中断时，操作系统会根据中断号，在中断描述表中查找并执行相应的中断处理程序。当处理程序返回后，进程继续执行**下一条指令**，就好像没有发生过中断一样。

![-w420](/media/15948291330824.jpg){:width="500px"}

## 异常（exception）
异常是一种错误情况，是执行当前指令的结果，可能被错误处理程序修正，也可能直接终止应用程序。异常是同步的。

注意这里的“异常”和上文一开始说的“异常”的区别。上文的“异常”是一个通用的术语，表示因为某些事件/操作而引起的控制流的改变，包括陷阱、中断和异常。这里的“异常”特指因为执行当前指令而产生的**错误情况**，比如除法异常、缺页异常等。有些书上为了区分，也将这类“异常”称为**“故障”**。

当发生异常时，操作系统会将控制转移给相应的异常处理程序。如果处理程序能够修正这个错误情况，就将返回到**引起异常的指令**重新执行。否则，**终止**该应用程序。

异常处理程序的地址也保存在中断描述符表（IDT）中。

![-w494](/media/15948647387834.jpg){:width="500px"}

常见的异常类型及处理：
* 除法错误（异常 0）：当应用程序试图除以零，或者一个除法指令结果溢出的时候，就会发生除法错误。Linux 会**直接终止**程序。Linux shell 报告为“浮点异常（Floating exception）”
* 一般保护故障（异常 13）：当应用程序访问一个未定义的虚拟内存区域（如访问空指针），或者试图写一个只读的文本段时，会发生一般保护故障。Linux 会**直接终止**程序。Linux shell 报告为“段错误（Segmantation fault）”
* 缺页异常（异常 14）：当应用程序访问未加载的页面时，会引起缺页异常。缺页处理程序会加载适当的页面，然后**重新执行**引起异常的指令
* ...

## 总结：陷阱、异常与中断

| | 陷阱 | 中断 | 异常 |
|---|---|---|---|
| 来源 | 处理机内部/陷阱指令 | 处理机外部/硬件 | 处理机内部 |
| 是否是执行当前指令的结果 | 是 | 否 | 是 |
| 同步/异步 | 同步 | 异步 | 同步 |
| 是否陷入内核| 是 | 是 | 是 |
| 处理程序位置 | IDT->系统调用表 | IDT | IDT |
| 返回行为 | 下一条指令 | 下一条指令 | 当前指令或终止 |
| 是否会导致进程终止 | 否 | 否 | 可能 |
{: .no-wrap-table}

<div id="_signal"></div>

## 信号（signal）
信号是一种**更高层的**软件形式的异常，同样会中断进程的控制流，可以由进程进行处理。一个信号代表了一个消息。信号的作用是用来**通知进程**发生了某种系统事件。

上文的陷阱、中断和异常都是低层异常机制，由内核的异常处理程序进行处理，正常情况下对用户进程是不可见的。信号提供了一种机制，通知用户进程发生了这些异常。比如，如果一个进程试图除以 0，那么内核会收到一个除零异常，内核会给进程发送一个 SIGFPE 信号（=8）；如果一进程进行非法内存访问，那么内核会收到一个一般保护故障，内核会给进程发送一个 SIGSEGV 信号（=11）...

此外还有其他系统事件，也可以通过信号来通知进程。比如，如果按下 Ctrl+C，那么内核会给进程发送一个 SIGINT 信号（=2）...

下图是 Linux 系统上支持的 30 种不同类型的信号：
![-w471](/media/15949139049229.jpg)

### 发送信号
信号除了由内核发给进程，也可以作为进程间通信的一种方式，明确地由一个进程发送给另一个进程。见[进程间的通信方式]({% post_url 2020-02-26-ipc %})。

发送信号的机制：
1. 用 `/bin/kill` 程序发送信号
2. 从键盘发送信号，比如按下 Ctrl+C 发送 SIGINT 信号、按下 Ctrl+Z 发送 SIGTSTP 信号
3. 用 `kill` 函数发送信号
4. 用 `alarm` 函数向自己发送 SIGLALRM 信号

### 接收信号
每个进程有一个待处理信号的集合。待处理信号表示发送给该进程但是还未被处理（接收）的信号，任何时刻同一类型的待处理信号最多只有一个，后续发送的同类型信号将会被丢弃（隐式阻塞）。

进程也可以选择阻塞某种信号。当一种信号被阻塞时，它仍可以被发送，但是产生的待处理信号不会被目标进程接收，直到进程取消对这种信号的阻塞（显式阻塞）。

### 处理信号
信号的处理时机：
* 当内核把进程**从内核态切换到用户态时**。例如，从系统调用返回，或是完成一次上下文切换
* 内核通过**控制转移**来强制进程*接收/处理*信号。如果进程的未被阻塞的待处理信号集合不为空，则内核会选择集合中的某个信号（通常是最小的），并将控制传递到信号处理程序；否则，内核正常地将控制传递到进程的下一条指令

用户进程对信号的处理过程有三种：
1. 执行默认操作，linux 对每种信号都规定了**默认行为**（见上图），是下面的一种
    * 进程终止
    * 进程终止并转储内存（core dump）
    * 进程停止（挂起）直到被 SIGCONT 信号重启
    * 进程忽略该信号
2. 忽略信号。当不希望接收到的信号对进程的执行产生影响，而让进程继续执行时，可以忽略该信号，即不对信号进程作任何处理
3. 处理信号。定义信号处理程序，当信号发生时，执行相应的处理程序

信号处理程序是一个**用户层函数**。进程可以为某个信号指定一个信号处理程序，接收到信号后，进程会跳转执行信号处理程序，执行完成后再返回到中断位置的**下一条指令**继续执行。
![-w407](/media/15949144782600.jpg){:width="500px"}

### `signal` 函数
进程可以通过 `signal` 函数修改某个信号的默认行为，包括忽略该信号或指定信号处理程序。唯一的例外是无法更改 SIGSTOP 和 SIGKILL 的默认行为。

终端执行 `man signal` 查看文档。

## 术语说明
### 中断向量表 IVT、中断描述符表 IDT
中断向量表（interrupt vector table，IVT）是由一系列中断向量（interrupt vector）组成的列表。每个中断向量都是一个中断处理程序的入口地址。中断向量的类型包括：**硬件中断**、**软件中断**和**处理器异常**，这些事件在中断向量表中统一称作*中断*。

当中断或异常产生时，由硬件负责产生一个中断标记，CPU 根据中断标记获得相应的**中断向量号**，然后将其作为偏移，在中断向量表中获得相应的处理程序地址，并执行。

中断向量表是一个通用的概念，在不同的架构下有不同的实现，比如在 x86 处理器下的实现是中断描述符表（Interrupt Descriptor Table，IDT）。本文不区分中断向量表和中断描述符表。

### 中断向量号 / 异常号
每种类型的异常都有一个唯一的异常号，也称作中断向量号，相当于是中断向量表的下标。中断描述符表中包含 256 个中断向量，对于 256 个异常号。

前 32 个是预留的、由**处理器产生**的异常，包括被零除、缺页、内存访问违例、断点以及算数运算溢出等，这些号码。其他号码是由**操作系统内核**定义的，包括系统调用（软中断）和来自外部设备的中断信号（硬中断）。

由硬件产生的异常：

| `INT_NUM` | 描述 |
| --- | --- |
| 0 | 被零除 |
| 1 | 单步调试 |
| 2 | NMI 中断 |
| 3 | 程序断电 |
| 4 | 溢出 |
| 5 | 边界范围超出 |
| 6 | 无效操作码 |
| 7 | 设备不存在 |
| 8 | 双重错误 |
| 9 | 协处理器段超越 |
| 10 | 无效的任务状态段 TSS |
| 11 | 段不存在 |
| 12 | 堆栈段错误 |
| 13 | 一般保护错误（比如访问空指针产生的“段错误”）|
| 14 | 缺页异常 |
| 15 | *保留* |
| 16 | 浮点错误 |
| 17 | 对齐检查 |
| 18 | 机器检查 |
| 19 | SIMD 浮点异常 |
| 20 | 虚拟化异常 |
| 21 | 控制保护异常 |

### 系统调用表
{: #system-call}

Linux 内核给 `0x80` 中断注册了名为 `ia32_syscall` 的中断执行程序。当执行汇编指令 `INT 0x80`、`syscall` 或 `sysenter` 时，都会执行到这个中断处理程序，这是所有系统调用的入口程序。

内核的每个系统调用的入口地址也保存在一个表格中，称为系统调用表（sys_call_table）。每个系统调用有一个唯一的整数号，对应于到内核中的系统调用表的偏移量。

进行系统调用前，进程会将系统调用号保存在 `%eax` 寄存器中，这样在陷入内核后，处理程序就知道该执行哪个系统调用。然后根据系统调用号，在系统调用表中找到并执行系统调用代码。

这就是通过陷阱指令实现系统调用的原理。陷阱指令不是一个指令，而是一类指令(?)。

## 参考资料
* [80386 的异常号与描述](https://wizardforcel.gitbooks.io/intel-80386-ref-manual/10.html)
* [Linux 系统调用指南](https://zcfy.cc/article/the-definitive-guide-to-linux-system-calls-670.html?t=new)
* 《深入理解计算机系统》第八章