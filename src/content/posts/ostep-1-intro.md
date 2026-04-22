---
title: "OSTEP 导读（一）：操作系统介绍"
description: "理解操作系统的本质：任务驱动的资源管理"
date: 2026-04-22
tags: ["OSTEP", "操作系统"]
---

## 阅读前提

这篇文章（以及整个系列）假设你已经理解：

- 程序是如何运行的（指令、内存、寄存器）
- 什么是编译、链接、加载
- 基本的C语言

如果你还不懂这些，建议先：
- 读CSAPP的前几章
- 或者看jyy的OS课前几讲
- 或者学一门计算机组成原理课程

**为什么要这样？**

因为OS是建立在"程序如何运行"之上的。如果你不懂程序运行的基础，学OS就是空中楼阁。

就像OSTEP书中说的：
> If you don't know what a program is, how it is compiled, or how it runs, you should stop reading this book and first learn about those things.

## 开始之前的检查

如果你能回答这些问题，就可以继续：
- 一个C程序编译后，生成的可执行文件里有什么？
- CPU如何执行指令？
- 程序的代码、数据、栈、堆分别在哪里？

如果不能，请先补基础。

---

## 操作系统是什么？

这个问题很难回答，因为在不同层次，对OS的看法是不一样的。

**从应用层看：**
- OS就像是硬件
- 我有接口（系统调用）去对接
- 我不关心OS内部怎么实现

**从硬件层看：**
- OS就是软件
- OS是第一个运行的程序
- OS无非就是躺在磁盘上的一组字节

**OS如何启动？**

当你给CPU上电时：
1. BIOS启动
2. BIOS加载bootloader
3. bootloader启动第一个程序：操作系统

**从状态机角度看：**

核心思想：任何程序都是状态机。

- 所有的执行都是 current → next 的接续
- 取指 → 译码 → 执行 → 取指...
- 这是状态的转移

在数学上，计算机就是一个无情的执行指令的机器。

OS也是一个程序，也是一个状态机，只不过它管理着其他状态机（进程）。

---

但如果要说OS要解决的核心问题，就是这三个：

1. **让多个程序同时执行**
2. **让不同程序之间保持安全、隔离**
3. **让数据持久化**

为了解决这三个问题，OS发明了三个核心抽象：

- **进程**：让多个程序"同时"运行
- **地址空间**：让程序之间互不干扰
- **文件系统**：让数据可靠存储

## 从硬件到抽象

从硬件角度看，计算机本质上只有三样东西：

1. **CPU**：执行指令
2. **内存**：存放数据
3. **磁盘**：持久化存储

这些资源的特点：
- **有限**：CPU只有几个核心，内存只有几GB
- **共享**：多个程序要同时用
- **需要保护**：程序A不能破坏程序B

OS的目标：让这些有限的资源，被多个程序安全、高效地使用。

为了达到这个目标，OS提供了三大抽象：

| 硬件资源 | OS抽象 | 目的 |
|---------|--------|------|
| CPU | 进程 | 让多个程序"同时"运行 |
| 内存 | 地址空间 | 让程序之间互不干扰 |
| 磁盘 | 文件系统 | 让数据可靠存储 |

## OSTEP的三大主题

OSTEP按三大主题组织：

1. **虚拟化（Virtualization）**：让每个程序觉得独占资源
2. **并发（Concurrency）**：让多个执行流协调工作
3. **持久性（Persistence）**：让数据可靠存储

下面我们用"任务驱动"的方式，理解这三大主题。

## 任务驱动的设计

我学OS最大的感悟是：**OS的设计是任务驱动的**。

不是先有概念，再有实现。而是：
- 我需要什么功能
- 我就设计什么机制
- 所有的抽象都是为了解决具体问题

**举例：**

### 主题1：虚拟化（Virtualization）

#### 任务：我想同时运行多个程序

看OSTEP书中的例子：

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/time.h>
#include <assert.h>
#include "common.h"

int main(int argc, char *argv[]) {
    if (argc != 2) {
        fprintf(stderr, "usage: cpu <string>\n");
        exit(1);
    }
    char *str = argv[1];
    while (1) {
        Spin(1);
        printf("%s\n", str);
    }
    return 0;
}
```

这个程序很简单：循环打印输入的字符串。

运行多个实例：

```bash
prompt> ./cpu A & ; ./cpu B & ; ./cpu C & ; ./cpu D &
[1] 7353
[2] 7354
[3] 7355
[4] 7356
A
B
D
C
A
B
D
C
...
```

**问题来了：**
- 硬件：只有一个CPU（或几个核心）
- 现象：4个程序"同时"在运行
- 怎么做到的？

**方案：时间分片 + 进程抽象**
- CPU快速切换，轮流执行每个程序
- 每个程序觉得自己独占CPU
- 本质：对CPU的扩展
（具体如何切换？通过时钟中断和上下文切换，我们会在下一篇详细讲）
#### 任务：程序之间不能互相干扰

看OSTEP书中的例子：

```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include "common.h"

int main(int argc, char *argv[]) {
    int *p = malloc(sizeof(int));
    assert(p != NULL);
    printf("(%d) address pointed to by p: %p\n", getpid(), p);
    *p = 0;
    while (1) {
        Spin(1);
        *p = *p + 1;
        printf("(%d) p: %d\n", getpid(), *p);
    }
    return 0;
}
```

运行多个实例：

```bash
prompt> ./mem &; ./mem &
[1] 24113
[2] 24114
(24113) address pointed to by p: 0x200000
(24114) address pointed to by p: 0x200000
(24113) p: 1
(24114) p: 1
(24113) p: 2
(24114) p: 2
...
```

**神奇的现象：**
- 两个进程的地址都是 0x200000
- 但它们各自计数，互不干扰
- 怎么做到的？

**方案：虚拟内存 + 地址空间抽象**
- 每个进程有独立的地址空间
- 0x200000 是虚拟地址，映射到不同的物理内存
- 本质：对内存的扩展
（具体如何映射？通过页表和MMU，我们会在内存篇详细讲）
**虚拟化的本质：**
- CPU虚拟化 → 进程抽象
- 内存虚拟化 → 地址空间抽象
- 让每个程序觉得独占CPU和内存

### 主题2：并发（Concurrency）

OSTEP在第二章简单提到了并发，主要讲线程。

看OSTEP书中的例子：

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include "common.h"

volatile int counter = 0;
int loops;

void *worker(void *arg) {
    int i;
    for (i = 0; i < loops; i++) {
        counter++;
    }
    return NULL;
}

int main(int argc, char *argv[]) {
    if (argc != 2) {
        fprintf(stderr, "usage: threads <value>\n");
        exit(1);
    }
    loops = atoi(argv[1]);
    pthread_t p1, p2;
    printf("Initial value : %d\n", counter);
    
    pthread_create(&p1, NULL, worker, NULL);
    pthread_create(&p2, NULL, worker, NULL);
    pthread_join(p1, NULL);
    pthread_join(p2, NULL);
    printf("Final value   : %d\n", counter);
    return 0;
}
```

运行结果：

```bash
prompt> ./threads 100000
Initial value : 0
Final value   : 143012  # 应该是200000，但结果错了！
```

**问题：**
- 两个线程各自循环100000次，counter应该是200000
- 但实际结果是143012（每次运行结果都不同）
- 为什么？

**原因：竞态条件（Race Condition）**
- 两个线程同时访问counter
- counter++不是原子操作
- 会发生数据冲突

**方案：锁、条件变量、信号量**

（并发部分会在后续章节详细讲，这里只是展示问题）

### 主题3：持久性（Persistence）

#### 任务：数据要持久化

看OSTEP书中的例子：

```c
#include <stdio.h>
#include <unistd.h>
#include <assert.h>
#include <fcntl.h>
#include <sys/types.h>

int main(int argc, char *argv[]) {
    int fd = open("/tmp/file", O_WRONLY | O_CREAT | O_TRUNC, S_IRWXU);
    assert(fd > -1);
    int rc = write(fd, "hello world\n", 13);
    assert(rc == 13);
    close(fd);
    return 0;
}
```

**问题：**
- 程序退出后，数据还在吗？
- 系统崩溃后，数据还在吗？

**方案：文件系统 + 文件抽象**
- 数据写到磁盘，持久化存储
- 即使程序退出、系统重启，数据还在
- 本质：对磁盘的扩展

## OS的本质

所以OS的本质是什么？

**OS就是对CPU、内存、磁盘这三个硬件资源的扩展和管理。**

- 所有的概念（进程、线程、虚拟内存、文件系统）
- 所有的机制（调度、分页、缓存、日志）
- 都是为了让这三个硬件资源更好地服务于程序

**一句话总结：OS就是任务驱动的资源管理。**

需要什么功能，就设计什么机制。所有的抽象都是为了解决具体问题。

## OSTEP的学习方式

理解了这一点，OSTEP的学习方式就清晰了：

不是先讲概念，而是：
1. 先提出任务（我想同时运行多个程序）
2. 分析资源（只有一个CPU）
3. 设计方案（时间分片）
4. 引入概念（进程）
5. 讨论实现（上下文切换）
6. 引出新问题（怎么调度？）

每一步都是为了解决具体问题，每个概念都有存在的理由。

## 下一篇预告

下一篇开始虚拟化篇：

**为什么要有进程？**

我们会从"我想同时运行多个程序"这个任务出发，一步步推导出进程这个抽象。
