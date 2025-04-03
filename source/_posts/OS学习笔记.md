---
title: OS笔记
date: 2025-03-01 23:47:14
top: 2
tags: 
- OS
categories: 
- Lect Notes
- System
---
南京大学蒋炎岩老师《操作系统》课程笔记，AI时代的操作系统课
## 绪论

**一个重塑全人类的 prompt：**

> 我在做 [X]。如果你是一位专业人士，有更好的方法和建议吗？尽可能全面。

操作系统：管理软/硬件资源、为程序提供服务

> A body of software, in fact, that is responsible for *making it easy to run programs* (even allowing you to seemingly run many at the same time), allowing programs to share memory, enabling programs to interact with devices, and other fun stuff like that. (OSTEP)

### 应用视角的操作系统

程序是个状态机，但有些东西单靠程序本身无法实现。

“纯粹的计算” 只能改变**程序内的状态**，但有些 API 涉及到 “程序外的状态”，这就和操作系统有关了

>操作系统最大的职责就是让我们**在编程时感受不到操作系统的存在**——我们在编程时想象程序 “独占整个计算机，逐条指令执行”，大部分时候都不用 “考虑” 操作系统的存在。当系统调用发生时，程序执行被完全暂停，但操作系统依然在工作——就像麻醉后醒来，周围的环境发生了变化，但我们完全没有感到时间的流逝。by jyy

**任何程序 = minimal.S = 状态机**

- 总是从被操作系统加载开始
  - 通过另一个进程执行 execve 设置为初始状态
- 经历状态机执行 (计算 + syscalls)
  - 进程管理：fork, execve, exit, ...
  - 文件/设备管理：open, close, read, write, ...
  - 存储管理：mmap, brk, ...
- 最终调用 _exit (exit_group) 退出



### 硬件视角的操作系统

硬件**根本不知道有没有操作系统**：

- 我就是个无情执行指令的状态机
  - 见什么指令执行什么指令

- 下层不需要知道上面怎么用，只管 “无情地提供服务”
  - 系统调用支撑应用程序
  - 指令集支撑高级语言程序

**操作系统就是一个普通的 (二进制) 程序**：

- 接管了中断、I/O、……
  - 应用程序不能直接访问
- 把应用程序 “放” 到 CPU 上运行一会
  - 中断后，操作系统又开始执行
  - (操作系统启动后，操作系统就变成了一个中断处理程序)

---

刚刚遗漏的一个细节：

CPU = 无情执行指令的机器

- 从 CPU Reset 开始执行
  - 从 Mem[PC] 取指令
  - 译码、执行，如此往复

那么，Reset处的代码是什么呢？

答案：系统厂商的代码

- 把一个特殊的存储器 memory-map 到 CPU Reset 后的代码
  - 这段代码 “出生” 就有机器完整的控制权

---

从固件到操作系统：

- 40年前的约定是，固件(BIOS)判断磁盘的前512字节（MBR, master boot record, 主引导扇区）是否可加载，然后将其内容加载到内存的0x7c00，因此如果你写一个简单的汇编程序在这里，固件就会先加载MBR，然后你的汇编程序再加载大的操作系统进来
- 我们可以“调试”这个过程，从而看到固件加载MBR的过程

`python`中，使用`os.system`执行命令和在命令行执行命令一样，也会有相同的命令行中的输出，如果需要输入中途确认信息，程序会暂停

### 数学视角的操作系统：

- 程序是个状态机，硬件也是个状态机，操作系统就是个状态机管理器。
- 朴素的想，可以通过状态机建模和遍历，实现对软件正确性的证明。

## 虚拟化

>  One of the most fundamental abstractions that the OS provides to users: the **process**

- 把物理计算机 “抽象” 成 “虚拟计算机”
  - 程序好像独占计算机运行

### 程序和进程

- 程序是状态机的静态描述
  - 描述了所有可能的程序状态
  - 程序 (动态) 运行起来，就成了**进程** (进行中的程序)

UNIX 的进程管理：

- 复制状态机: fork()
- 复位状态机: execve()

---

#### fork：立即复制状态机

- 包括所有状态的完整拷贝
  - 寄存器 & 每一个字节的内存
  - Caveat: 进程在操作系统里也有状态: ppid, 文件, 信号, ...
    - **小心这些状态的复制行为**
  - 复制失败返回 -1

如何区分呢？

- 新创建进程返回 0
- 执行 fork 的进程返回子进程的进程号——“父子关系”

理解fork：两个例子：

```c
// 第一个例子，来自OSTEP，很好的描述了fork的基本原理
int main(int argc, char *argv[])
{
    printf("hello world (pid:%d)\n", (int) getpid());
    int rc = fork();
    if (rc < 0) {
        // fork failed; exit
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // child (new process)
        printf("hello, I am child (pid:%d)\n", (int) getpid());
    } else {
        // parent goes down this path (original process)
        printf("hello, I am parent of %d (pid:%d)\n",
	       rc, (int) getpid());
    }
    return 0;
}
```

```c
// 一个非常tricky的例子，来自课件
for (int i = 0; i < 2; i++) {
    fork();
    printf("Hello\n");
}
// ./a.out 和 ./a.out | wc -l 输出结果不同，前者是6行hello，后者输出8，为什么？
```

**例子2的关键点：缓冲机制与 `fork()` 的交互：**

1. **缓冲模式**：
   - **终端**：行缓冲（`\n` 触发刷新）。
   - **管道/文件**：全缓冲（缓冲区满或程序结束才刷新）。
2. **`fork()` 复制缓冲区**：
   - 子进程会复制父进程的未刷新缓冲区，导致多次输出相同内容。

同时，必须注意的是，fork之后，如果不加限制，子进程和父进程的执行顺序是没有约定的，下一行代码可能是父进程先执行，也可能是子进程先执行，这中不确定性可能导致很多问题，将在并发模块详细讨论。
但是如果加上一个`wait()`系统调用，那就确定了，wait的行为是：

> This system call won’t return until the child has run and exited. Thus, even when the parent runs first, it politely waits for the child to finish running, then wait() returns, and then the parent prints its message.

---

（下面这部分与OS主线内容无关）

除了写到环境变量里，API_KEY放到哪更安全？（可以使用SOPS对配置文件数据进行加密，回头试试）

在不同shell里执行enecve env，输出的结果是不同的：

- 不同 Shell 对 `execve` 的隐式处理（如自动添加 `PWD`、`SHLVL`、`_` 等变量）。

- ```c
  // 直接这样指定就可以不带任何你自己没有传入的环境变量
  char *const argv[] = {
          "/bin/env",
          NULL,
      };
  ```

---

#### execve：复位状态机

> a successful call to exec() never returns

```c
int execve(const char *filename,
           char * const argv[], char * const envp[]);
```

**核心行为：**

`execve` 是一个 **进程映像替换（Process Image Replacement）** 的系统调用，其行为规则是：

1. 成功调用时：
   - 当前进程的代码、数据、堆栈等会被完全替换为新的程序（此处是 `/bin/bash`）。
   - **原进程的后续代码（包括 `printf`）不再执行**，因为原进程的映像已被覆盖。
2. 失败调用时：
   - `execve` 返回 `-1`，并设置 `errno` 表示错误原因。
   - 程序继续执行后续代码（即 `printf`）。

execve 是唯一能够 “执行程序” 的系统调用：

- 因此也是一切进程 strace 的第一个系统调用

### 进程的地址空间

> 每个进程都 “拥有” 自己的内存——我们可以用 new/delete 管理它们。但进程的内存是哪里来的？一个指针 `p` 到底能指向什么？管理地址空间的系统调用: mmap 给我们答案。

从进程的初始状态说起：

寄存器

- (之前观察过一次)
- 这个好办，直接打印出来就行了

内存

- 这个有点难办
  - 我们可以把任何整数 cast 成指针
  - **用工具，看真实系统**
  - 向AI老师学习，AI可以为我们生成`statedump.py`脚本，查看内存映射

观测后的发现：不是所有的系统调用都要中断交给内核（如映射中的vvar, vdso区）

**vdso和vvar：**

- `vdso` 是一个共享库，包含可以直接在用户空间调用的函数。
- `vvar` 是 `vdso` 函数所需的数据来源，存储内核维护的变量或数据结构。
- 当用户空间程序调用 `vdso` 中的函数（如 `gettimeofday`）时，`vdso` 会从 `vvar` 中读取数据，而无需进入内核模式。

**使用场景：**

- **性能优化**：`vdso` 和 `vvar` 减少了频繁系统调用的开销，特别适用于高频率的时间获取操作。
- **简化开发**：用户空间程序可以直接调用 `vdso` 中的函数，而无需关心底层系统调用的细节。

---

多想一想：关于进程的初始状态

- 只有ELF 文件里声明的内存和一些操作系统分配的内存
  - 任何其他指针的访问都是非法的
  - 如果我们从输入读一个 size，然后 malloc(size)
    - 内存从拿来呢？

一定有一个**系统调用**可以改变进程的地址空间！

> Memory Map系统调用：在状态机状态上增加/删除/修改一段可访问的内存

```c
// 映射
void *mmap(void *addr, size_t length, int prot, int flags,
           int fd, off_t offset);
int munmap(void *addr, size_t length);

// 修改映射权限
int mprotect(void *addr, size_t length, int prot);
```

`mmap` 的主要作用包括：

1. **文件映射**
   - 将文件内容映射到进程的虚拟内存中，从而实现对文件的高效读写（避免频繁的 `read`/`write` 系统调用）。
   - 例如，可以将一个大文件映射到内存中，直接通过指针访问文件内容。（或者把可执行文件加载到内存，就可以运行了（后面讲加载器还会再讲））
2. **共享内存**
   - 使用 `MAP_SHARED` 标志，多个进程可以共享同一块内存区域，用于进程间通信（IPC）。
3. **匿名内存分配**
   - 使用 `MAP_ANONYMOUS` 标志，可以分配一块匿名内存区域，类似于 `malloc`，但更灵活。

---

**进程 (状态机) 在 “无情执行指令机器” 上执行**

- 状态机是一个封闭世界
- 但如果允许一个进程对其他进程的地址空间有访问权？
  - 意味着可以任意改变另一个程序的行为
    - 听起来就很 cool

**“入侵” 进程地址空间的例子**

- 调试器 (gdb)
  - gdb 可以任意观测和修改程序的状态
- 金山游侠（游戏外挂）

**怎样入侵？**

- 一切皆文件，**直接修改 /proc/[pid]/mem**
- ptrace系统调用

### 访问操作系统对象

前言：

> 操作系统还必须给应用程序提供访问操作系统对象的机制。当然，这可以直接以 API 的形式提供，例如 Win32 API 包含 “RegOpenKeyEx” 访问注册表。但我们主要关注 UNIX 的 “Everything is a file” 访问操作系统对象的机制，以及这一机制带来的方便 (和不便)。

#### 文件描述符

文件：有 “名字” 的数据对象

- 字节流 (终端，random)
- 字节序列 (普通文件)

文件描述符

- **指向操作系统对象的 “指针”**（所以名字起的不好，并不是什么描述信息）
  - Everything is a file
  - 通过指针可以访问 “一切”

文件描述符的分配：总是分配最小的未使用描述符

- 0, 1, 2 是标准输入、输出和错误
- 新打开的文件从 3 开始分配
  - 文件描述符是进程文件描述符表的索引
  - 关闭文件后，该描述符号可以被重新分配
  - have a try，看看具体分配和重新分配的规律

文件描述符是 “进程状态的” 的一部分

- 保存在操作系统中；程序只能通过整数编号访问
- 文件描述符自带一个 offset

**Quiz: fork() 和 dup() 之后，文件描述符共享 offset 吗**？

- 这就是 fork() 看似优雅，实际复杂的地方
- have a try
- 通过AI编写的代码，事实证明是的

对比：Linux中的文件描述符和Windows中的handle

windows是更面向工程的设计

- 默认 handle 是不继承的 (和 UNIX 默认继承相反)
  - 可以在创建时设置 bInheritHandles，或者运行时修改
  - “最小权限原则”（更注重安全，而unix是hack风格:joy:）
- lpStartupInfo 用于配置 stdin, stdout, stderr

#### 访问操作系统中的对象

文件描述符是用来访问对象的指针，操作系统提供了用来创建对象的API，比如mount

通过mount挂载，相当于创建了可以访问的文件描述符对象

dup命令

**任何“可读写”的东西都是文件！**

管道：一个特殊的 “文件” (流)

- 由读者/写者共享
  - 读口：支持 read
  - 写口：支持 write

```c
// 匿名管道
int pipe(int pipefd[2]); 
```

- 返回两个文件描述符
- 进程同时拥有读口和写口
- 如果读口得不到东西，就等待（前提是写端也是打开的，不能说写段中途被关闭，这样写口的关闭被读口检测到，读口也会结束读取）
- **管道规则**：当管道的**所有写端被关闭**时，`read()` 会返回 `0`（表示 EOF），而不是无限阻塞。
  - 此时子进程的 `read(fd[0], ...)` 会立即返回 `0`，子进程继续执行后续代码（打印空数据并退出）。

#### 一切皆文件的优劣

- 优雅，文本接口，就是好用
- 和各种 API 紧密耦合
- 对高速设备不够友好
  - 额外的延迟和内存拷贝
  - 单线程 I/O

本讲总结：

> 操作系统必需提供机制供应用程序访问操作系统对象。对于 UNIX 系统，文件描述符是操作系统中表示打开文件 (操作系统对象) 的指针；而 Windows 则更直接地提供了 handle 机制。

### 终端和 UNIX Shell

从打字机时代->键位为什么是现在这样的？到电传打字机到模拟终端，终端终究是一个人机交互的设备

/dev/pts（伪终端从设备）、/dev/ptmx（伪终端主设备）

pty：伪终端

#### 伪终端的核心结构

伪终端由一对“设备”组成：

1. **主设备（PTY Master）**：
   - 由终端模拟器（如 `GNOME Terminal`、`xterm` 或 `SSH 客户端`）控制。
   - 负责从用户输入（如键盘）接收数据，并将数据发送到从设备。
   - 同时接收从设备的输出（如程序的执行结果），并显示给用户。
2. **从设备（PTY Slave）**：
   - 连接到实际运行的程序（如 `bash`、`python` 等）。
   - 程序认为自己是在和真实的终端通信，但实际上是通过伪终端与主设备交互。

```
+----------------+         双向管道           +----------------+
|  终端模拟器     | <-----------------------> |   Shell 程序    |
|  (PTY Master)  |                           |  (PTY Slave)   |
+----------------+                           +----------------+
```

```python
import os
import pty

# 创建伪终端的主从设备
master, slave = pty.openpty()

# 打印主从设备的路径
print("主设备文件描述符:", master)  # 例如 3
print("从设备路径:", os.ttyname(slave))  # 例如 /dev/pts/5

# 在从设备上运行一个 Shell（这里用子进程模拟）
pid = os.fork()
if pid == 0:
    # 子进程：连接到从设备，并启动 bash
    os.close(master)
    os.dup2(slave, 0)  # 将标准输入/输出/错误重定向到从设备
    os.dup2(slave, 1)
    os.dup2(slave, 2)
    os.execlp("bash", "bash")
else:
    # 父进程（主设备）：向从设备发送命令并读取输出
    os.close(slave)
    os.write(master, b"echo 'Hello PTY!'\n")  # 发送命令
    output = os.read(master, 1024)  # 读取输出
    print("收到输出:", output.decode())  # 输出 "Hello PTY!"
```

---

/dev/tty和/dev/pts有什么区别？为什么tty显示的是/dev/pts（tty现在代表的好像是控制终端）

`openpty()`: 通过 `/dev/ptmx` 申请一个新终端，everything is a file，这也是操作系统中的对象。

- 这也能间接看出一切皆文件的一点坏处，就是如果linux里面有/dev/ptmx，开发者就倾向不用api直接访问文件，但是这种代码就是不可移植的

#### 终端和操作系统

程序是个状态机，除了内部的内存和寄存器状态，以及之前介绍的进程状态pid,ppid，还有sid,pgid,uid,……，因此进程状态非常复杂，这个状态机大的超乎想象

> 我们有那么大一棵进程树，都指向同一个终端，有的在前台，有的在后台，Ctrl-C 到底终止哪个进程？

解决方案：

<img src="./../Pictures/Screenshots/屏幕截图 2025-03-13 213046.png" alt="屏幕截图 2025-03-13 213046" style="zoom:75%;" />

**给进程引入一个额外编号 (Session ID，大分组)**

- 子进程会继承父进程的 Session ID
  - 一个 Session 关联一个控制终端 (controlling terminal)
  - Leader 退出时，全体进程收到 Hang Up (SIGHUP)

**再引入另一个编号 (Process Group ID，小分组)**

- 通常由同一个命令行触发的进程及其子进程组成，这个PGID通常是进程组首进程（第一个创建的进程）的 PID
- 管道命令 `ls | grep "file" | wc -l` 会创建一个进程组，包含 `ls`、`grep` 和 `wc` 三个进程。

- 只能有一个前台进程组
- 操作系统收到 Ctrl-C，向前台进程组所有进程发送 SIGINT
  - (真累……但你也想不到更好的设计了)

一个进程id的例子：（这里是先挂起了两个python和一个vim，然后使用ps命令）

```
 PID    PPID    PGID     SID TT       COMMAND
620     356     620     356 pts/0    python
621     356     621     356 pts/0    vim
647     356     647     356 pts/0    python
657     356     657     356 pts/0    ps
可见它们有相同的sid，但是pgid不同
```

- stty -a: 你可以看到按键绑定 (奇怪的知识增加了)
- fg %i（恢复/切换）“最小化” = Ctrl-Z (SIGTSTP)
- 最大化，最小化，关闭按钮，都早在命令行终端就实现了

**创建一棵进程子树**的实验，更多的细节：按ctrl+c，不管有没有托孤，进程都会退出（说明进程退出不是看pid，而是有追踪pgid的方法）



从这两节课讲的各种设备其实让我们看到了系统的复杂性：

- 系统构建总是走向屎山，因为不可能预见未来的需求，同时你的基础设施用的人太多了，根本不能做减法

### C 标准库和实现

> 在操作系统 API 之上，为了服务应用程序，毋庸置疑需要有一套 “好用” 的库函数。虽然 libc 在今天的确谈不上 “好用”，但成就了 C 语言今天 “高级的汇编语言” 的可移植地位，以 ISO 标准的形式支撑了操作系统生态上的万千应用。

#### libc简介

C 语言：世界上 “最通用” 的高级语言

- C 是一种 “高级汇编语言”
  - 作为对比，C++ 更**好用**，但也更**难移植**
- 系统调用的一层 “浅封装”

libc：语言机制上的运行库

- 大部分可以用 C 语言本身实现
- 少部分需要一些 “底层支持”
  - 例子：体系结构相关的内联汇编

libc库也被标准化

- ISO IEC 标准的一部分
- POSIX C Library 的子集
  - 稳定、可靠 (不用担心升级版本会破坏实现)
  - 极佳的移植性：包括你自己的操作系统！

#### 基础编程机制的抽象

看看perror，env和jyy的调试过程

#### 系统调用和环境的抽象

反复出现的 “No such file or directory”

- 这不是巧合！（是由于出错时设置了不同的错误标志位）
  - 我们也可以 “山寨” 出同样的效果
- Quiz: errno 是进程共享还是线程独享？
  - 线程有一些我们 “看不到” 的开销：ucontext, errno, ...

#### 内存管理

使用的系统调用：mmap (历史：sbrk)

- 大段内存，要多少有多少
  - 用 MAP_ANONYMOUS 申请
  - 超过物理内存上限都行 (Demo)

操作系统**不支持**分配一小段内存

- 这是应用程序自己的事
  - 应用程序可以每次向操作系统 “多要一点” 内存
  - 自己在内存上实现一个数据结构

实现malloc(n)：

如果n足够大

- 直接用 mmap 分配内存

如果n不够大

- 从一个 mmap 得到的内存池中分配
  - 我们有一个大区间 [L,R)
  - 维护其中不相交的小区间集合
    - 啊！太好了！这是个数据结构问题……
    - 最简单想法：free list（低效）
    - Interval tree 可以实现 O(log⁡n) 的分配/回收

### 可执行文件

> (静态链接) 可执行文件的概念和基本原理；我们如何自己动手构建一个可执行文件格式。

#### ELF：Executable and Linkable Format

可执行文件是进程初始进程状态的描述

回顾：[System V ABI](https://jyywiki.cn/OS/manuals/sysv-abi.pdf)

- Section 3.4: “Process Initialization”
  - 只规定了部分寄存器和栈
  - 其他状态 (主要是内存) 由**可执行文件**指定

“可执行文件” 需要包含什么？

- (今天先假设不依赖 “动态链接库”)
- 基本信息 (版本、体系结构……)
- 内存布局 (哪些部分是什么数据)
- 其他 (调试信息、符号表……)

ELF文件：一个经典、没有任何问题的设计，但：

- ELF 不是一个人类友好的 “状态机数据结构描述”
  - 为了性能，彻底违背了可读 (“信息局部性”) 原则



 [CRIU](https://criu.org/Main_Page)：项目状态存储和迁移工具，利用了core dump。（做一个core dump并迁移不难，但是操作系统还保留了一部分当前执行文件的信息，这部分信息的取得和迁移，冲突避免等就很有挑战了）

- 在AI agent时代就太有用了

#### Funny Little Executable

计算机是人造的科学，随时问一句：如果是我来设计，会怎样做？有想法就能实现

ELF这种文件并不神秘，它就是一个“数据结构”

### 动态链接和加载

#### 动态链接

为什么要有动态链接：

- 实现运行库和应用代码分离
  - 运行库和应用代码还可以分别独立升级
  - 可以用 ldd 命令查看
- 大型项目的分解
  - 改一行代码不用重新链接 2GB 的文件

如果 Linux 应用世界是静态链接的……

- libc 紧急发布安全补丁 → 重新链接**所有**应用
- [Semantic Versioning](https://semver.org/)
  - “Compatible” 是个有些微妙的定义
  - “Dependency hell”

#### mmap 和虚拟内存

一个实验：

- 构造一个非常大的 libbloat.so
  - 我们的例子：100M of nop (0x90)
- 创建 1,000 个进程动态链接 libbloat.so 的进程
- 观察系统的内存占用情况（`vmstat or free -h`）
  - 100MB or 100GB?

结论：**只读方式 mmap 同一个文件，物理内存中只有一个副本**。进程的地址空间(mmap+进程号查看，会发现每个进程都以为自己有一个libbloat.so)，实际是**分页机制维护的 “幻象”**

更多的结论：（可以通过观察gcc --static和不加static选项的a.out来观测）

- a.out (动态链接) 的第一条指令不在程序里
  - 可以 starti 验证 (在 ld.so 里)
  - 这是 “写死” 在 ELF 文件的 INTERP (interpreter) 段里的
    - (我们甚至可以直接编辑它)
- a.out 执行时，libc 还没有加载
  - 可以 info proc mappings (pmap) 验证
- libc 是用 mmap 系统调用加载的
  - 可以 strace 验证

Virtual Memory：

- 操作系统维护 “memory mappings” 的数据结构
  - 这个数据结构很紧凑 (“哪一段映射到哪里了”)
- 延迟加载
  - 不到万不得已，不给进程分配内存
- 写时复制 (Copy-on-Write)
  - fork() 时，父子进程先只读共享全部地址空间
    - Page fault 时，写者复制一份

#### 实现动态链接

编译器在编译和汇编时，不知道一个函数是静态还是动态链接，因此，对于一个函数调用，编译器只能生成短跳转。

链接器会判断出这个函数是不是动态链接的，如果是，则在 a.out 里 **“合成” 一段小代码**：

```
printf@plt:    jmp *PRINTF_OFFSET(%rip) 
```

这就是**PLT (Procedure Linkage Table)**！

得到PLT中的信息后，再去**全局偏移表**（**GOT**, Global Offset Table）中获取存放外部的函数地址的数据。

**延迟绑定的实现机制：**

在 Linux 系统中，延迟绑定是通过 **PLT（Procedure Linkage Table，过程链接表）** 和 **GOT（Global Offset Table，全局偏移表）** 实现的：

1. **PLT**：存储跳转到 GOT 的代码。
2. **GOT**：存储函数实际地址的表格。
3. **动态链接器**：在函数第一次被调用时，动态链接器会解析函数的实际地址并更新 GOT。

---

但是PLT没有解决数据的问题：（这段内容不用掌握）

- 对于 `extern int x`，我们不能 “间接跳转”！

- **数据访问必须立即解析地址**，因为 CPU 指令需要确定的内存地址来执行读/写操作。（否则每次数据访问都查表，性能无法忍受）
- **函数调用可以延迟绑定**，因为跳转指令可以通过 PLT/GOT 间接实现。

不优雅的解决方法

- -fPIC 默认会为所有 extern 数据增加一层间接访问
  - `__attribute__((visibility("hidden")))`

---

LD_PRELOAD：一个神奇的 “hook” 机制

- 允许 “preload” 一个自己的库
  - 当然，没有魔法
  - LD_PRELOAD 会传递给 ld-linux.so
- 我们可以在运行时，用一个自己的库替换掉某个库：（比如替换malloc/free，更好的观察内存分配）

```shell
LD_PRELOAD=./mylib.so ./a.out 
```

LD_PRELOAD的核心原理：

- 动态链接器（如 `ld-linux.so`）在加载程序时，会按以下顺序搜索符号（函数/变量）：
  1. **LD_PRELOAD 指定的库** → 2. **程序的依赖库** → 3. **系统默认库（如 `libc.so`）**。
- 如果多个库定义了相同的符号，**LD_PRELOAD 的库优先级最高**，其函数会被调用。

### 构建应用生态

（本讲科普性质）

#### 除了对象和系统调用API：

The last piece: “初始状态”

- Everything is a state machine
  - “操作系统中的对象” 应该也有一个初始状态

它是什么呢？

- 观察到 Linux 的系统更新
  - “update-initramfs... (漫长的等待)”
  - 这就是 Linux Kernel 启动后的 “初始状态”

我们的 initramfs（最小的initramfs）

- 可以只有一个 init 文件
  - 可以是任意的 binary，包括我们自己实现的
- (系统启动后，Linux 还会增加 /dev 和 /dev/console)
  - 需要给 stdin/stdout/stderr 一个 “地方”

#### initramfs: 并不是我们看到的Linux世界

启动的初级阶段

- 加载剩余必要的驱动程序，例如磁盘/网卡
- 挂载必要的文件系统
  - Linux 内核有启动选项 (类似环境变量)
    - /proc/cmdline (man 7 bootparam)
  - 读取 root filesystem 的 /etc/fstab
- 将根文件系统和控制权移交给另一个程序，例如 systemd

启动的第二级阶段

- 看一看系统里的 /sbin/init 是什么？
- 计算机系统没有魔法 (一切都有合适的解释)

---

#### 可以自己接管这一切：

创建一个磁盘镜像

- /sbin/init 指向任何一个程序
  - 同样需要加载必要的驱动
  - 例子：pivot_root 之后才加载网卡驱动、配置 IP
  - 例子：tty 字体变化
    - 这些都是 systemd 的工作
- 例子：[NOILinux Lite](https://zhuanlan.zhihu.com/p/619237809)

Initramfs 会被**释放**

- 功成身退



