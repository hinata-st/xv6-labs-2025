# Chapter 1 Operating system interfaces

当一个进程需要请求一项内核服务时，它会调用一个 “系统调用（system call）”，每个系统调用就是操作系统所提供的一套接口中的一个实例。

shell 是一个普通的程序，它从用户那里读取命令并执行它们。shell 是一个用户态程序，而不是内核的一部分

## 1.1 Peocesses and memory

一个 xv6 进程由用户空间内存（用于存放指令、数据和实现栈）以及内核私有的每个进程的状态组成。

```C
int pid = fork();
if(pid > 0){
  printf("parent: child=%d\n", pid);
  pid = wait((int *) 0);
  printf("child %d is done\n", pid);
} else if(pid == 0){
  printf("child: exiting\n");
  exit(0);
} else {
  printf("fork error\n");
}
```

## 1.2 I/O and File descriptors

如果两个文件描述符是通过一系列 `fork` 和 `dup` 调用从同一个原始文件描述符派生而来的，则它们共享同一个偏移量。否则，文件描述符不共享偏移量，即使它们是由对同一个文件的 `open` 调用生成的。

## 1.3 Pipes

程序首先调用 `pipe` 函数，创建一个新的管道，传入的参数 `p` 是一个整型数组，用于记录管道两端的读写文件描述符（译者注，具体来说，当 `pipe` 成功返回时，`p[0]` 指向管道的读取端，`p[1]` 指向管道的写入端）。

## 1.4 File system

一个文件的名字与文件本身不是同一个概念；同一个底层的文件（我们称之为 “索引节点（inode）”）可以有多个文件名（我们称之为 “链接（link）”）。每个链接对应目录中的一个 “目录项（entry）” ；该目录项由文件名和对 inode 的引用组成。inode 保存了文件的 “元数据（metadata）”，元数据包括文件的类型（文件、目录或设备）、长度、磁盘上保存文件内容的位置以及指向该文件的链接数量。

## 1.5 Real world

## Lab util

### sleep

```C
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  int n;
  
  if (argc < 2) {
    fprintf(2, "Usage: sleep ticks\n");
    exit(1);
  }
  n = atoi(argv[1]);
  pause(n);
  exit(0);
}
```

### sixfive

```C
#include "kernel/types.h"
#include "kernel/fcntl.h"
#include "user/user.h"

enum token_state { AT_SEPARATOR, IN_NUMBER, INVALID_TOKEN };

void
print_if_multiple(int number)
{
  if (number % 5 == 0 || number % 6 == 0)
    printf("%d\n", number);
}

int
sixfive(int fd, char *name)
{
  char c;
  int n;
  int number = 0;
  enum token_state state = AT_SEPARATOR;

  while ((n = read(fd, &c, 1)) == 1) {
    if ('0' <= c && c <= '9') {
      // 是首位数字
      if (state == AT_SEPARATOR) {
        number = c - '0';
        state = IN_NUMBER;
      } else if (state == IN_NUMBER) {
        number = number * 10 + c - '0';
      }
    } else if (strchr(" -\r\t\n./,", c)) {
      if (state == IN_NUMBER)
        print_if_multiple(number);
      number = 0;
      state = AT_SEPARATOR;
    } else {
      state = INVALID_TOKEN;
    }
  }

  if (n < 0) {
    fprintf(2, "sixfive: read error in %s\n", name);
    return -1;
  }
  if (state == IN_NUMBER)
    print_if_multiple(number);
  return 0;
}

int
main(int argc, char *argv[])
{
  int fd, i;

  if (argc < 2) {
    fprintf(2, "Usage: sixfive files...\n");
    exit(1);
  }

  for (i = 1; i < argc; i++) {
    if ((fd = open(argv[i], O_RDONLY)) < 0) {
      fprintf(2, "sixfive: cannot open %s\n", argv[i]);
      exit(1);
    }
    if (sixfive(fd, argv[i]) < 0) {
      close(fd);
      exit(1);
    }
    close(fd);
  }
  exit(0);
}
```

### memdump

```C
#include "kernel/types.h"
#include "kernel/fcntl.h"
#include "user/user.h"

#define MAX_DATA 512

void memdump(char *fmt, char *data);

int
main(int argc, char *argv[])
{
  if (argc == 1) {
    int a[2] = { 61810, 2025 };
    char *s = "another";
    struct sample {
      char *ptr;
      int num1;
      short num2;
      char byte;
      char bytes[8];
    } example;

    printf("Example 1:\n");
    memdump("ii", (char *)a);

    printf("Example 2:\n");
    memdump("S", "a string");

    printf("Example 3:\n");
    memdump("s", (char *)&s);

    example.ptr = "hello";
    example.num1 = 1819438967;
    example.num2 = 100;
    example.byte = 'z';
    strcpy(example.bytes, "xyzzy");

    printf("Example 4:\n");
    memdump("pihcS", (char *)&example);

    printf("Example 5:\n");
    memdump("sccccc", (char *)&example);
  } else if (argc == 2) {
    char data[MAX_DATA + 1];
    int n, nn;

    n = 0;
    memset(data, '\0', sizeof(data));
    while (n < MAX_DATA) {
      nn = read(0, data + n, MAX_DATA - n);
      if (nn < 0) {
        fprintf(2, "memdump: read error\n");
        exit(1);
      }
      if (nn == 0)
        break;
      n += nn;
    }
    memdump(argv[1], data);
  } else {
    fprintf(2, "Usage: memdump [format]\n");
    exit(1);
  }
  exit(0);
}

void
memdump(char *fmt, char *data)
{
  int i;

  for (i = 0; fmt[i] != '\0'; i++) {
    switch (fmt[i]) {
    case 'i': {
      int value;

      memmove(&value, data, sizeof(value));
      printf("%d\n", value);
      data += sizeof(value);
      break;
    }
    case 'p': {
      uint64 value;

      memmove(&value, data, sizeof(value));
      // The lab's expected pointer display uses xv6's 32-bit %x output.
      printf("%x\n", (uint)value);
      data += sizeof(value);
      break;
    }
    case 'h': {
      short value;

      memmove(&value, data, sizeof(value));
      printf("%d\n", value);
      data += sizeof(value);
      break;
    }
    case 'c':
      printf("%c\n", *data);
      data += sizeof(char);
      break;
    case 's': {
      char *value;

      memmove(&value, data, sizeof(value));
      printf("%s\n", value);
      data += sizeof(value);
      break;
    }
    case 'S':
      printf("%s\n", data);
      return;
    default:
      fprintf(2, "memdump: unknown format %c\n", fmt[i]);
      return;
    }
  }
}
```

### find

```C
#include "kernel/types.h"
#include "kernel/fcntl.h"
#include "user/user.h"
#include "kernel/stat.h"
#include "kernel/fs.h"
#include "kernel/param.h"

void find(char *path, char *name, char **command, int command_argc);

char *
basename(char *path)
{
  char *p = path + strlen(path);

  while (p > path && p[-1] != '/')
    p--;

  return p;
}

void
run_command(char *path, char **command, int command_argc)
{
  char *exec_argv[MAXARG];
  int i, pid;

  for (i = 0; i < command_argc; i++)
    exec_argv[i] = command[i];
  exec_argv[command_argc] = path;
  exec_argv[command_argc + 1] = 0;

  pid = fork();
  if (pid < 0) {
    fprintf(2, "find: fork failed\n");
    return;
  }
  if (pid == 0) {
    exec(exec_argv[0], exec_argv);
    fprintf(2, "find: exec %s failed\n", exec_argv[0]);
    exit(1);
  }
  wait(0);
}

int
main(int argc, char *argv[])
{
  char **command = 0;
  int command_argc = 0;

  if (argc < 3) {
    fprintf(2, "Usage: find <path> <name> [-exec <cmd> [args...]]\n");
    exit(1);
  }

  if (argc > 3) {
    if (argc < 5 || strcmp(argv[3], "-exec") != 0) {
      fprintf(2, "Usage: find <path> <name> [-exec <cmd> [args...]]\n");
      exit(1);
    }
    command = &argv[4];
    command_argc = argc - 4;
    if (command_argc > MAXARG - 2) {
      fprintf(2, "find: too many arguments for -exec\n");
      exit(1);
    }
  }

  find(argv[1], argv[2], command, command_argc);
  exit(0);
}

void
find(char *path, char *name, char **command, int command_argc)
{
  char buf[512], *p;
  int fd;
  struct dirent de;
  struct stat st;

  if ((fd = open(path, O_RDONLY)) < 0) {
    fprintf(2, "find: cannot open %s\n", path);
    return;
  }

  if (fstat(fd, &st) < 0) {
    fprintf(2, "find: cannot stat %s\n", path);
    close(fd);
    return;
  }

  switch (st.type) {
  case T_DEVICE:
  case T_FILE:
    if (strcmp(basename(path), name) == 0) {
      if (command)
        run_command(path, command, command_argc);
      else
        fprintf(1, "%s\n", path);
    }
    break;

  case T_DIR:
    if (strlen(path) + 1 + DIRSIZ + 1 > sizeof buf) {
      fprintf(2, "find: path too long\n");
      break;
    }
    strcpy(buf, path);
    p = buf + strlen(buf);
    *p++ = '/';
    while (read(fd, &de, sizeof(de)) == sizeof(de)) {
      if (de.inum == 0)
        continue;
      memmove(p, de.name, DIRSIZ);
      p[DIRSIZ] = 0;
      if (strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0)
        continue;
      find(buf, name, command, command_argc);
    }
    break;
  }
  close(fd);
}
```

# Chapter 2 Operating system organization

操作系统必须满足三个需求：复用、隔离、交互

## 2.1 Abstracting physical resources

为了实现更好的隔离，最好禁止应用程序直接访问敏感的硬件资源，代之以将对资源的访问抽象为服务。

Unxi进程之间的许多交互都是通过“文件描述符”实现的。

## 2.2 User mode, supervisor mode, and system calls

"强隔离"需要在应用程序和操作系统之间建立一个明显的边界。

CPU 为实现强隔离从硬件层面提供了支持。例如，RISC-V 的 CPU 有三种 “特权级别（privilege level）”： *机器模式（machine mode）* 、*管理员模式（supervisor mode）* 和  *用户模式（user mode）* 。在管理员模式下，CPU 被允许执行  *特权指令（privileged instructions）* ：例如，启用和禁用中断、对保存页表地址的寄存器进行读取和写入等。如果在用户模式下应用程序试图执行特权指令，那么 CPU 不仅会拒绝执行，还会将处理器通过 “陷入（trap）” 方式切换到管理员模式并执行一段特殊的代码来终止应用程序。

应用程序通过系统调用（例如 `read`）与内核交互。应用程序不允许直接调用内核函数或访问内核内存。RISC-V 为系统调用提供了 `ecall` 指令；该指令将 CPU 从用户模式切换到管理员模式，并跳转到内核指定的 “入口点（entry point）” 。

## 2.3 Kernel organization

有一个关键的设计问题是：操作系统的哪些部分应该在管理员模式下运行。一种可能性是整个操作系统驻留在内核中，这样所有系统调用的实现都以管理员模式运行。这种组织方式被称为  *宏内核（monolithic kernel）* 。

与大多数 Unix 操作系统一样，xv6 采用了宏内核的组织方式。因此，xv6 内核的接口就是操作系统的接口，内核实现了完整的操作系统。由于 xv6 提供的服务较少，其内核比一些微内核更小，但从概念上讲，xv6 仍然属于宏内核。

## 2.4 Code:xv6 organization

![1784537989001](image/note/1784537989001.png)

内核的主要职责进行划分，这包括：启动系统（引导）、创建和控制进程、处理 “陷阱（traps）”（中断和系统调用）、分配内存和配置虚拟地址、控制设备以及管理文件系统。

## 2.5 Process overview

（和其他 Unix 操作系统一样）xv6 中实现隔离的基本单元是一个  *进程（process）* 。进程是一个抽象的概念，一个进程无法访问（更谈不上破坏）另一个进程所拥有的内存、CPU、文件描述符等。一个进程也无法破坏内核本身，自然这样一个进程就不能突破内核的隔离机制。内核必须小心地实现进程这个抽象设计，因为一个有缺陷或恶意的应用程序可能会欺骗内核或硬件做坏事（例如，绕过隔离机制）。内核用来实现进程的机制包括区分用户模式和管理员模式，引入地址空间的概念以及对线程的运行划分时间片。

为了加强隔离，进程这种抽象设计给程序提供了一种错觉，即好像它拥有自己专有的机器。一个程序似乎拥有一块私有的内存区域，也叫作  *地址空间（address space）* ，其他进程对它不能读取也不能写入。这个程序还拥有自己的 CPU 来执行它的指令。

xv6 使用 “页表（page tables）”（在硬件的帮助下）为每个进程提供自己的地址空间。RISC-V 的页表将  *虚拟地址（virtual address）* （RISC-V 的指令操作的地址）“翻译（translate）”（或 “映射（map）”）为  *物理地址（physical address）* （CPU 芯片发送到主存储器的地址）。

xv6 为每个进程维护一个单独的页表，定义了该进程的地址空间。

![1784538368881](image/note/1784538368881.png)

xv6 内核为每个进程维护许多信息，所有的这些内容都定义在一个结构体 `struct proc` (2034) 中。

### `proc.h` 总览

`kernel/proc.h` 定义了 xv6 的“执行状态模型”。它本身不实现进程操作，而是规定内核如何表示：

- 被暂停的内核执行流：`context`
- 每个 CPU 的调度状态：`cpu`
- 用户态进入内核时的寄存器快照：`trapframe`
- 一个完整的进程控制块：`proc`

### `struct context`

`context` 用于内核上下文切换：

```c
struct context {
  uint64 ra;
  uint64 sp;
  uint64 s0;
  // ...
  uint64 s11;
};
```

它只保存 `ra`、`sp` 和 callee-saved 寄存器 `s0~s11`。原因是 `swtch()` 按普通 C 函数调用边界工作：调用者负责保存临时寄存器，而被调用者必须保持 callee-saved 寄存器。

真正的保存和恢复发生在 `kernel/swtch.S` 中。它没有单独保存 `pc`，因为恢复 `ra` 后执行 `ret`，`ra` 就充当了恢复执行的位置；`sp` 恢复后，原来的内核调用栈、局部变量和返回链也随之恢复。

因此 `context` 并不是完整的进程状态。它必须和该进程的 `kstack` 配合：

```text
context：保存少量关键寄存器
kstack：保存尚未返回的函数调用、局部变量和返回地址
```

### `struct cpu`

`cpu` 描述的是一个 CPU 核心，而不是进程：

| 字段        | 含义                                             |
| ----------- | ------------------------------------------------ |
| `proc`    | 当前在这个 CPU 上运行的进程；调度器运行时为`0` |
| `context` | 该 CPU 的调度器上下文                            |
| `noff`    | 当前嵌套关闭中断的层数                           |
| `intena`  | 第一次关闭中断前，中断是否开启                   |

`cpus[NCPU]` 的实体定义在 `kernel/proc.c` 中，`proc.h` 里的 `extern` 只是声明。当前配置中 `NCPU=8`。

调度时存在两个 `context`：

```text
CPU 的 c->context  <---- swtch() ---->  进程的 p->context
调度器执行流                            进程的内核执行流
```

`noff` 让嵌套的 `push_off()/pop_off()` 不会过早打开中断。`myproc()` 读取 `c->proc` 前也会暂时关闭中断，防止读取过程中进程被调度到另一个 CPU。

### `struct trapframe`

`trapframe` 保存用户态寄存器。它和 `context` 最容易混淆：

```text
用户态 --trap--> 内核        保存到 trapframe
进程内核态 --swtch--> 调度器  保存到 context
```

前五个字段是进入内核所需的引导信息：

| 字段              | 用途                                    |
| ----------------- | --------------------------------------- |
| `kernel_satp`   | 内核页表对应的`satp` 值               |
| `kernel_sp`     | 当前进程内核栈顶                        |
| `kernel_trap`   | `usertrap()` 地址                     |
| `epc`           | 返回用户态后继续执行的地址              |
| `kernel_hartid` | 当前 CPU 的 hart id，即内核使用的`tp` |

后面保存 RISC-V 的 31 个非零通用寄存器。`x0/zero` 永远为零，无须保存。字段前的 `0、8、16...` 是字节偏移，必须和 `kernel/trampoline.S` 中的硬编码偏移严格一致，所以不能随意调整字段顺序。

每个进程分配一整页 trapframe，并映射到该进程页表中的固定地址 `TRAPFRAME`：

```text
高地址
TRAMPOLINE  进入/退出内核的汇编代码
TRAPFRAME   当前进程的寄存器保存页
...
用户栈、堆、代码
低地址
```

这两个映射都没有 `PTE_U`，用户程序不能访问。固定虚拟地址很关键：发生 trap 时还在使用用户页表，汇编代码却能立即通过 `TRAPFRAME` 找到当前进程的保存区。

一次系统调用大致如下：

```text
ecall
 -> uservec 保存用户寄存器到 trapframe
 -> 从 kernel_* 字段取得内核栈、页表和 usertrap 地址
 -> usertrap()
 -> syscall() 从 a7 取系统调用号，从 a0~a5 取参数
 -> 返回值写入 trapframe->a0
 -> userret 恢复寄存器
 -> sret 根据 sepc 返回用户态
```

对应代码位于 `kernel/trap.c` 和 `kernel/syscall.c`。

### 进程状态

`enum procstate` 的典型转换是：

```text
UNUSED -> USED -> RUNNABLE -> RUNNING
                     ^          |
                     |          +-> RUNNABLE  yield
                     |          +-> SLEEPING
                     |                |
                     +----------------+
                                wakeup
RUNNING -> ZOMBIE -> UNUSED
             exit      wait 回收
```

`USED` 表示进程槽位已经占用并正在初始化，但还不能被调度；`RUNNABLE` 表示可以运行但正在等 CPU；`RUNNING` 才表示正在某个 CPU 上执行；`ZOMBIE` 已经退出，但必须保留 `pid` 和退出状态，等待父进程调用 `wait()`。

### `struct proc`

`proc` 就是 xv6 的进程控制块。系统预先建立固定大小的 `proc[NPROC]` 数组，当前最多 64 个进程。

| 字段          | 含义                                  |
| ------------- | ------------------------------------- |
| `lock`      | 保护该进程的共享状态                  |
| `state`     | 当前生命周期状态                      |
| `chan`      | 睡眠时等待的事件标识                  |
| `killed`    | 已收到终止请求，但不代表已经退出      |
| `xstate`    | `exit(status)` 留给父进程的退出状态 |
| `pid`       | 进程 ID                               |
| `parent`    | 父进程，由全局`wait_lock` 保护      |
| `kstack`    | 进入内核后使用的内核栈虚拟地址        |
| `sz`        | 用户地址空间的逻辑大小                |
| `pagetable` | 用户页表根节点                        |
| `trapframe` | 用户寄存器保存页                      |
| `context`   | 内核调度上下文                        |
| `ofile[16]` | 文件描述符到`struct file` 的映射    |
| `cwd`       | 当前工作目录的 inode                  |
| `name[16]`  | 调试名称，不是完整路径或身份标识      |

`chan` 通常只是一个用作事件身份的地址，不一定会被解引用。`sleep(chan, lock)` 设置 `chan` 和 `SLEEPING`；`wakeup(chan)` 扫描进程表，把匹配者改为 `RUNNABLE`。

`killed` 也不是强制立即销毁进程。`kill()` 设置标志，并唤醒正在睡眠的进程；目标进程在 trap 等安全边界检查标志后调用 `exit()`。

### 锁的划分

`state`、`chan`、`killed`、`xstate`、`pid` 由 `p->lock` 保护。`parent` 单独由 `wait_lock` 保护，因为父子关系、`exit()`、`wait()` 和唤醒必须作为一套协议处理，避免丢失唤醒。锁顺序要求先拿 `wait_lock`，再拿某个 `p->lock`。

注释所说的“private to the process”不是用户态内存隔离，而是内核的所有权约定：通常只有当前进程修改这些字段，因此正常使用时不需要 `p->lock`。

最值得牢牢记住的是：

```text
pagetable   决定用户程序能看见什么内存
trapframe   决定怎样回到用户程序
kstack      保存当前内核函数调用链
context     决定怎样恢复被暂停的内核执行流
struct cpu  连接当前进程与每 CPU 调度器
struct proc 把以上所有状态和 Unix 资源组织起来
```

---

每个进程有两个栈：一个用户栈和一个内核栈（`p->kstack`）。当进程执行用户指令时，只使用它的用户栈，它的内核栈是空的。当进程进入内核（由于系统调用或中断）时，内核代码使用进程的内核栈；当一个进程进入内核态后，它的用户栈仍然包含保存的数据，只是不被内核指令使用。

总之，进程的概念包含了两个设计思想：一个是地址空间的概念，它让进程感觉拥有自己专属的内存；另一个是线程的概念，它让进程感觉拥有自己专属的 CPU。在 xv6 中，一个进程由一个地址空间和一个线程组成。在实际操作系统中，一个进程可能含有多个线程，以充分利用处理器中的多个核。

---

## 2.6 Code: starting xv6, the first process and system call

`kernel/entry.S`、`kernel/start.c`、`kernel/main.c` 和 `user/init.c` 共同描述了 xv6 从“裸 CPU”到“可交互 shell”的启动过程。学习重点不是背初始化函数，而是理解栈、地址空间和执行上下文如何逐步建立。

```text
QEMU
  -> entry.S:_entry
  -> start.c:start
  -> main.c:main
  -> proc.c:userinit
  -> scheduler
  -> proc.c:forkret
  -> exec.c:kexec("/init")
  -> trampoline.S:userret
  -> user/init.c:main
  -> fork + exec("sh")
```

### `kernel/entry.S`

QEMU 使用 `-kernel kernel/kernel` 将内核加载到物理地址 `0x80000000`。链接脚本 `kernel/kernel.ld` 指定 `_entry` 为入口，并将它放在该地址。

`entry.S` 执行时：

- CPU 处于 Machine mode。
- 分页尚未启用。
- 没有进程。
- 没有可供 C 函数使用的栈。
- 所有 hart 都从 `_entry` 开始执行。

它的核心工作是为每个 hart 选择独立启动栈：

```asm
la sp, stack0
li a0, 4096
csrr a1, mhartid
addi a1, a1, 1
mul a0, a0, a1
add sp, sp, a0
```

最终：

```text
hart 0: sp = stack0 + 1 * 4096
hart 1: sp = stack0 + 2 * 4096
hart 2: sp = stack0 + 3 * 4096
```

使用 `(hartid + 1)` 是因为 RISC-V 栈向低地址增长，`sp` 应指向每块栈的顶部。这是 CPU 启动栈，不是进程的 `p->kstack`。

建立栈后才能安全执行：

```asm
call start
```

如果 `start()` 意外返回，CPU 会进入 `spin` 死循环。正常情况下它不会返回。

### `kernel/start.c`

`start()` 仍运行在 Machine mode。它的职责是完成只有 M-mode 有权执行的硬件配置，然后把控制权交给 Supervisor mode。

首先设置 `mstatus.MPP`：

```c
x &= ~MSTATUS_MPP_MASK;
x |= MSTATUS_MPP_S;
w_mstatus(x);
```

`MPP` 表示执行 `mret` 后进入哪个特权级，这里选择 S-mode。

然后设置：

```c
w_mepc((uint64)main);
```

`mepc` 决定 `mret` 后从哪里继续执行。因此最后的：

```c
asm volatile("mret");
```

语义是：

```text
目标地址 = mepc = main
目标模式 = mstatus.MPP = Supervisor
```

其他配置的作用如下：

| 配置                 | 作用                                     |
| -------------------- | ---------------------------------------- |
| `satp = 0`         | 暂时关闭分页，使用物理地址               |
| `medeleg/mideleg`  | 将可委托的异常和中断交给 S-mode          |
| `sie`              | 允许 supervisor 定时器和外部中断源       |
| `pmpaddr0/pmpcfg0` | 允许 S-mode 访问物理内存                 |
| `timerinit()`      | 开放`stimecmp/time` 并安排首次时钟中断 |
| `tp = mhartid`     | 让`cpuid()` 能从 `tp` 读取 hart id   |

这里的 `mret` 不是普通函数返回，而是一次特权级切换。

### `kernel/main.c`

所有 hart 都会进入 `main()`，但初始化分为两类。

hart 0 建立全局共享设施：

```text
consoleinit / printkinit  控制台和内核输出
kinit                     物理页分配器
kvminit                   创建共享内核页表
procinit                  初始化进程表
trapinit                  初始化 trap 共享状态
plicinit                  配置设备中断优先级
binit / iinit / fileinit  文件系统内存结构
virtio_disk_init          磁盘设备
userinit                  创建第一个进程
```

每个 hart 都必须单独执行：

```text
kvminithart   将内核页表写入当前 hart 的 satp
trapinithart  将 kernelvec 写入当前 hart 的 stvec
plicinithart  启用当前 hart 的设备中断
```

`kvminit()` 和 `kvminithart()` 的区别尤其重要：

```text
kvminit()      创建页表，只执行一次
kvminithart()  启用页表，每个 hart 都要执行
```

其他 hart 在 `started` 上等待。`volatile` 防止编译器把循环读取优化掉，内存 fence 保证 hart 0 的初始化结果对其他 hart 可见。

最终所有 hart 都进入：

```c
scheduler();
```

`scheduler()` 永不返回。从此 CPU 不再沿启动代码向下运行，而是在进程之间切换。

### `main()` 如何到达 `/init`

内核的 `main()` 不能直接调用用户程序的 `main()`。两者属于不同地址空间和特权级。

`userinit()` 只创建第一个进程的内核骨架：

- 分配 `struct proc`
- 分配 trapframe
- 创建空用户页表
- 准备进程内核栈
- 设置 `context.ra = forkret`
- 设置根目录
- 将状态改为 `RUNNABLE`

调度器第一次选择该进程时，`swtch()` 恢复它的 `context`，于是从 `forkret()` 开始运行。

`forkret()` 完成：

```c
fsinit(ROOTDEV);
kexec("/init", (char *[]){"/init", 0});
prepare_return();
userret(...);
```

`fsinit()` 可能睡眠，所以需要在普通进程上下文中执行，不能直接放在启动阶段的 `main()` 中。

`kexec()` 从文件系统读取 `/init` ELF，建立用户页表和用户栈，并设置：

```text
trapframe->epc = ELF 入口，即 ulib.c:start
trapframe->sp  = 用户栈顶
trapframe->a0  = argc
trapframe->a1  = argv
```

最后 `userret` 切换用户页表并执行 `sret`，完成 S-mode 到 U-mode 的转换。

需要注意：较老版本的 xv6 使用内嵌的 `initcode.S` 启动 `/init`；当前这版直接在第一次 `forkret()` 中调用 `kexec("/init")`。

### `user/init.c`

`user/init.c` 是第一个用户进程，通常 PID 为 1。它不是 shell，而是用户空间的根进程。

第一项职责是建立标准文件描述符：

```c
if (open("console", O_RDWR) < 0) {
  mknod("console", CONSOLE, 0);
  open("console", O_RDWR);
}
dup(0);
dup(0);
```

第一次 `open()` 获得最低空闲 fd，也就是 0；两次 `dup(0)` 依次获得 1 和 2：

```text
fd 0 -> console：标准输入
fd 1 -> console：标准输出
fd 2 -> console：标准错误
```

之后 shell 通过 `fork()` 继承这些文件描述符，`exec()` 又保留它们，因此 shell 无须重新配置终端。

第二项职责是监督 shell：

```c
pid = fork();

if (pid == 0)
  exec("sh", argv);
```

父进程 `init` 不断调用 `wait()`：

- 如果回收的是 shell，说明 shell 已退出，于是重新启动。
- 如果回收的是孤儿进程，只负责清理 zombie。
- 如果 `wait()` 出错，则 `init` 退出，但内核实际上禁止初始进程退出。

这是因为进程退出时，内核会把失去父进程的子进程重新交给 `initproc`。没有 `init`，孤儿进程将无人回收。

### 三条启动主线

```text
特权级：
M-mode entry/start -> S-mode main/kernel -> U-mode init/sh

栈：
stack0 启动栈 -> p->kstack 进程内核栈 -> 用户栈

地址空间：
satp=0 -> kernel_pagetable -> init 用户页表
```

---

## 2.7 Securuty Model

内核的目标是限制每个用户进程，使其只能访问自己用户内存上的内容和指令，此外还要限制用户进程只能使用 32 个通用的 RISC-V 寄存器，并只允许它以系统调用允许的方式影响内核和其他进程。内核必须阻止任何其他操作。这通常是内核设计中必须要满足的需求。

作为对内核错误的应对措施的一部分，xv6 代码包含对不一致和不可恢复错误的检查，并通过调用 `panic()` 来处理这些异常（译者注，这里我们称该动作为 “崩溃（Panicking）”）。

## 2.8 Real world

大多数操作系统都采纳了进程的概念，并且大多数操作系统的进程看起来与 xv6 的很像。然而，现代操作系统支持在一个进程中创建多个线程，使得一个进程能够利用多个处理器。

## 2.9 Exercises

1. 为 xv6 增加一个系统调用, 返回当前可用的内存的大小。

## 补充

## 4.3 Code: calling system calls

C 编译器为这个函数调用生成相关的指令将三个参数分别存放在寄存器 `a0`、`a1` 和 `a2` 中，然后 `write()` 函数负责将系统调用号，即 `SYS_write`（其具体值为 16）放在 `a7` 中。

`ecall` 指令触发 trap，将处理器从用户态切换到内核态，并顺序执行 `uservec`、`usertrap` 和 `syscall`。

`syscall` (3731) 从 trapframe 中保存的 `a7` 中取到系统调用号，并用它作为索引在 `syscalls` 中找到对应的项。

`syscall` 会将其返回值记录在 `p->trapframe->a0` 中。

下面以 `write(1, buf, n)` 为例，把这条路径拆开说明。这里假定用户程序已经在用户态运行，且 `buf` 指向用户虚拟地址空间中的一段数据。

### 1. 用户 C 调用与 RISC-V 调用约定

用户代码写下：

```C
int r = write(1, buf, n);
```

`user/user.h` 只提供函数原型；真正被链接进去的是构建时由 `user/usys.pl` 生成的 `user/usys.S` 汇编桩。按照 RISC-V ABI，C 编译器在调用该桩之前先放置参数：

```text
a0 = 1       第 0 个参数：文件描述符 fd
a1 = buf     第 1 个参数：用户缓冲区地址
a2 = n       第 2 个参数：写入字节数
```

`write` 对应的桩等价于：

```asm
write:
        li a7, SYS_write   # 本仓库中 SYS_write 为 16
        ecall
        ret
```

`a7` 专门保存系统调用号。`ecall` 不会像普通函数调用那样跳转到一个由用户指定的地址；它是特权指令，要求 CPU 进入内核预先设置好的 trap 入口。至此，参数仍只在用户寄存器中，尚未由内核读取。

### 2. `ecall` 使 CPU 从 U-mode 陷入 S-mode

执行 `ecall` 后，RISC-V 硬件自动完成最少的一组状态转换：

- 进入 supervisor mode（S-mode）；
- 把导致陷入的用户 PC 写入 `sepc`；
- 在 `scause` 中记录原因。来自用户态的 `ecall` 对应 `scause == 8`；
- 根据 `stvec` 跳转到 trap 入口。

硬件**不会**自动把 `a0`--`a7` 等通用寄存器压栈，也不会自动切换到内核页表或内核栈。因此 xv6 需要首先执行一小段汇编入口代码来保存现场。

在即将返回用户态时，`prepare_return()` 已把 `stvec` 设置为 `TRAMPOLINE + (uservec - trampoline)`。这个地址位于 trampoline 页；它被映射到用户页表和内核页表的相同虚拟地址，所以页表尚未切换时也能开始执行 `uservec`，切换后仍能继续执行。

### 3. `trampoline.S:uservec` 保存用户现场并切换执行环境

`kernel/trampoline.S` 的 `uservec` 此时在 S-mode 执行，但仍使用用户页表。它完成以下工作：

1. 先把原来的 `a0` 暂存到 CSR `sscratch`，因为稍后要用 `a0` 作为临时地址寄存器；
2. 将固定映射的 `TRAPFRAME` 地址装入 `a0`，依次把用户的通用寄存器保存到 `p->trapframe`；最后从 `sscratch` 取回并保存原来的用户 `a0`；
3. 从 trapframe 读取 `kernel_sp`，切到当前进程的内核栈；读取 `kernel_hartid` 到 `tp`；
4. 从 trapframe 读取 `kernel_satp`，写入 `satp`，并用 `sfence.vma` 刷新旧地址转换缓存，切换到内核页表；
5. 从 trapframe 读取 `kernel_trap`，间接调用 `usertrap()`。

所以，进入 `usertrap()` 时，用户调用 `write` 时设置的值已经保存为：

```text
p->trapframe->a0 == 1
p->trapframe->a1 == buf
p->trapframe->a2 == n
p->trapframe->a7 == SYS_write
```

### 4. `usertrap()` 识别系统调用

`kernel/trap.c:usertrap()` 首先把 `stvec` 改为 `kernelvec`：从现在开始，如果内核代码再次发生 trap，应使用内核 trap 入口而不是 `uservec`。

接着它把硬件 CSR `sepc` 保存到 `p->trapframe->epc`。对于系统调用，`sepc` 指向的正是那条 `ecall`。如果原样返回会再次执行 `ecall`，形成无限陷入，因此 xv6 执行：

```C
p->trapframe->epc += 4;
```

RISC-V 的 `ecall` 指令是 4 字节长；这样最终回到用户态时将从其后的 `ret` 执行。

当 `r_scause() == 8` 时，`usertrap()` 确认这是系统调用，完成必要检查后打开中断，并调用 `syscall()`。打开中断放在保存 `sepc`、`scause` 后，是因为中断也会改变这些 trap CSR。

### 5. `syscall()` 查表分发，并由处理函数读取参数

`kernel/syscall.c:syscall()` 读取：

```C
num = p->trapframe->a7;
```

例如 `num == 16` 时，`syscalls[SYS_write]` 指向 `sys_write`，内核通过函数指针调用对应的系统调用实现。此时 C 函数没有显式形参，是因为它从当前进程的 trapframe 中取参数；详细的取参过程见下一节 4.4。

对 `sys_write()` 而言，参数读取逻辑可概括为：

```C
argint(0, &fd);       // trapframe->a0
argaddr(1, &addr);    // trapframe->a1：用户缓冲区的虚拟地址
argint(2, &n);        // trapframe->a2
```

注意 `addr` 只是一个来自用户态的地址数值，不能由内核直接解引用。实际读写数据时，内核会通过 `copyin`、`copyout` 或文件层函数，结合该进程的用户页表安全地访问用户内存。

处理函数的返回值会被 `syscall()` 写回：

```C
p->trapframe->a0 = syscalls[num]();
```

因此 `a0` 在进入内核时是第一个参数，返回用户态前则被覆盖为系统调用的返回值。例如 `write` 成功时为已写入的字节数，失败时通常为 `-1`。

### 6. 恢复现场并返回用户态

`usertrap()` 完成后调用 `prepare_return()`，它为下一次用户态 trap 重新设置 `stvec = uservec`，并在 trapframe 中填好下一次进入内核所需的 `kernel_satp`、`kernel_sp`、`kernel_trap` 和 `kernel_hartid`。它还设置：

- `sepc = p->trapframe->epc`：返回到 `ecall` 之后的用户指令；
- `sstatus.SPP = 0`：`sret` 后回到 U-mode；
- `sstatus.SPIE = 1`：返回用户态后恢复中断使能。

`usertrap()` 将用户页表的 `satp` 值作为返回值交给 `trampoline.S:userret`。`userret` 切回用户页表，随后从 trapframe 恢复寄存器：`a1`--`a7` 等恢复为调用时的值，而 `a0` 恢复为处理函数刚写入的返回值。最后 `sret` 根据 `sepc` 和 `sstatus` 进入用户态。

控制流于是回到 `usys.S` 中 `ecall` 后的 `ret`；该 `ret` 再回到最初的 C 调用点，C 变量 `r` 从 `a0` 接收返回值。

可以把整个过程压缩为：

```text
C 调用 write(1, buf, n)
  -> a0/a1/a2 放参数，a7 放 SYS_write
  -> usys.S: ecall
  -> 硬件写 sepc/scause，跳 uservec
  -> uservec 保存寄存器到 trapframe，切内核栈和内核页表
  -> usertrap: 保存 epc，epc += 4，调用 syscall
  -> syscall: 用 trapframe->a7 查表，sys_write 用 a0/a1/a2 取参
  -> 返回值写 trapframe->a0
  -> userret 恢复寄存器和用户页表，sret 回用户态
  -> usys.S: ret，C 调用者得到 a0 中的返回值
```

```
用户 C 代码
   │
   │ 例如 write(1, buf, n)
   ▼
user/user.h                 仅提供函数声明
   │
   ▼
user/usys.S                 构建时由 user/usys.pl 生成的汇编桩
   │
   │ 设置 a7 = 系统调用号
   │ 执行 ecall
   ▼
RISC-V 硬件陷阱入口
   │
   ▼
kernel/trampoline.S:uservec
   │ 保存用户寄存器，切换到内核页表和内核栈
   ▼
kernel/trap.c:usertrap()
   │
   │ 识别 scause == 8
   │
   ▼
kernel/syscall.c:syscall()
   │ 根据 a7 查表分发
   ▼
sys_write() / sys_read() / sys_fork() / ...
   │
   ▼
内核子系统，例如 filewrite()
   │
   ▼
返回值写入 trapframe->a0
   │
   ▼
usertrap() → trampoline.S:userret
   │ 恢复用户寄存器，切回用户页表
   ▼
sret
   │
   ▼
用户态 usys.S 中的 ret
   │
   ▼
C 调用者得到返回值
```

## 4.4 Code:System call arguments

内核实现了安全地根据用户提供的地址传输数据的函数。

## Lab: system calls

### Using gdb(easy)

打开一个终端

```Shell
cd /root/xv6/xv6-riscv
make qemu-gdb
```

打开另一个终端

```Shell
cd /root/xv6/xv6-riscv
gdb-multiarch
```

在系统调用开头将语句`num = p->trapframe->a7;` 替换为 `num = * (int *) 0;`，然后运行 `make qemu`

```Shell
riscv64-unknown-elf-ld: warning: kernel/kernel has a LOAD segment with RWX permissions
riscv64-unknown-elf-objdump -S kernel/kernel > kernel/kernel.asm
riscv64-unknown-elf-objdump -t kernel/kernel | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$/d' > kernel/kernel.sym
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 2 starting
hart 1 starting
scause=0xd sepc=0x800027fc stval=0x0
panic: kerneltrap
QEMU: Terminated
```

查看根据sepc值kernal.asm文件，a3寄存器与变量num对应

```Shell
(gdb) p p->name
$1 = "init", '\000' <repeats 11 times>
```

#### 崩溃原因分析

`kernel/syscall.c` 中的代码：

```C
num = *(int *)0;
```

会让内核从虚拟地址 `0` 读取一个整数。启动时输出的三个寄存器值分别表示：

- `scause=0xd`：异常号 13，即 **Load Page Fault（加载页错误）**。
- `sepc=0x800027fc`：发生异常的指令地址。通过 `kernel/kernel.asm` 或 GDB 可以确认，它对应 `syscall()` 中的 `lw a3, 0(zero)`，也就是上面的空指针解引用。
- `stval=0x0`：本次页错误所访问的虚拟地址是 `0`。

xv6 构造内核页表时，只映射了 UART、VirtIO、PLIC、从 `KERNBASE` 开始的内核内存、trampoline 和各进程的内核栈，并没有映射虚拟地址 `0`。因此，内核在 S-mode 下读取地址 `0` 时触发页错误。该异常发生在内核态，`kerneltrap()` 没有可用的恢复方式，于是执行 `panic("kerneltrap")`，导致内核崩溃。

严格来说，`scause=0xd` 单独只能证明发生了加载页错误：无效页表项或者页权限不允许读取都可能产生该异常。结合 `stval=0`、`sepc` 指向的加载指令以及内核页表的映射代码，才能确认这里是因为地址 `0` 没有映射。

在故障指令执行前用 GDB 查看当前进程：

```Shell
(gdb) p p->name
$1 = "init", '\000' <repeats 11 times>
(gdb) p p->pid
$2 = 1
```

所以，内核崩溃时正在运行的进程是 `init`，进程 ID 是 `1`。完成实验后，应将代码恢复为：

```C
num = p->trapframe->a7;
```

## Sandbox a command(moderate)
