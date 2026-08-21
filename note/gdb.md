# 使用 GDB 调试操作系统

> 资料：MIT 6.828《Using the GNU Debugger》
> [gdb_slides.pdf](https://pdos.csail.mit.edu/6.828/2019/lec/gdb_slides.pdf)

这份讲义以 JOS/x86 的启动过程为例介绍 GDB。虽然示例中的寄存器和启动汇编是 x86 风格，但断点、单步、查看寄存器和调用栈等方法同样适用于 xv6-riscv。

## 一、从启动代码理解栈

启动代码设置栈指针并调用 bootmain：

```asm
movl $start, %esp
call bootmain
```

call bootmain 会先把返回地址压入栈。进入 bootmain() 后，函数序言建立栈帧：

```asm
push %ebp
mov  %esp, %ebp
push %edi
push %esi
push %ebx
sub  $0x1c, %esp
```

这段代码保存旧的 %ebp 和被调用者保存寄存器，并为局部变量分配空间。之后 bootmain() 从 ELF 文件头获得入口地址并调用：

```c
entry = (void(*)(void))(elf->entry);
entry();
```

调用 entry() 又会压入一个返回地址。因此调试启动代码时，可以通过分析 call、函数序言和 %esp/%ebp 的变化，手工还原当前栈帧。这对于定位栈破坏、错误返回地址和异常控制流很有帮助。

> 这个例子是 x86；在 xv6-riscv 中应观察 RISC-V 的 sp、ra 和保存寄存器。

## 二、启动 GDB

项目通常提供 .gdbinit，用于配置 GDB 与 QEMU 的连接。启动 GDB 时应位于实验或 xv6 目录中，并确保用户级 ~/.gdbinit 允许加载项目目录下的初始化文件。

典型流程：

```text
终端一：make qemu-gdb
终端二：gdb
```

不需要调试时使用普通的 make qemu。不确定命令用法时，执行：

```text
help <command>
```

GDB 命令只要缩写后仍然无歧义即可，例如：

```text
continue -> c
step     -> s
stepi    -> si
```

##### 示例：

打开一个终端

```Shell
cd /root/xv6/xv6-riscv
make qemu-gdb
```

打开另一个终端

```Shell
cd /root/xv6/xv6-riscv
gdb-multiarch kernel/kernel

然后在GDB中执行
target remote localhost:25000
break syscall
continue
```

## 三、控制执行

### 源代码级单步

```text
step       # 执行一行，遇到函数调用则进入函数
next       # 执行一行，遇到函数调用则跳过函数
```

### 汇编级单步

```text
stepi      # 执行一条汇编指令，进入调用的函数
nexti      # 执行一条汇编指令，跳过调用的函数
```

这些命令可以带数字参数表示重复执行多次。直接按回车会重复上一个命令。

### 运行到指定位置

```text
continue             # 继续运行，直到断点或 Ctrl-C
finish               # 运行到当前函数返回
advance <location>   # 运行到指定位置
```

## 四、断点

设置断点：

```text
break <location>
```

位置可以是内存地址、函数名或文件行号：

```text
break *0x7c00
break backtrace
break monitor.c:71
```

管理断点：

```text
delete       # 删除断点
disable      # 暂时禁用断点
enable       # 重新启用断点
```

条件断点只在条件成立时暂停：

```text
break <location> if <condition>
cond <number> <condition>  # 给已有断点添加条件
```

## 五、观察点（Watchpoint）

观察点监视数据，而不是代码位置。

```text
watch <expression>       # 表达式的值变化时暂停
watch-l <address>        # 指定内存地址的内容变化时暂停
rwatch [-l] <expression> # 表达式对应的数据被读取时暂停
```

watch counter 关注变量值的变化，而 watch-l &counter 直接关注变量所在地址的内存写入。观察点适合定位哪条指令修改了变量，或哪里破坏了栈。

## 六、查看内存和变量

### 原始内存：x

```text
x/x <address>   # 按十六进制查看
x/i <address>   # 按汇编指令查看
```

例如 x/i $pc 可以查看当前程序计数器对应的指令。

### 按 C 类型查看：print

```text
print variable
p *((struct elfhdr *) 0x10000)
```

print 会按 C 表达式和类型解释数据，通常比逐字节使用 x 更容易读。查看 ELF 头时，按 struct elfhdr 打印比 x/13x 更直观。

## 七、寄存器、栈帧和调用链

```text
info registers   # 查看所有寄存器
info frame       # 查看当前栈帧
list <location>  # 查看指定位置附近的源代码
backtrace        # 查看函数调用栈
```

backtrace 对 xv6/JOS 的异常处理、系统调用和 Lab 1 的 backtrace 练习尤其有用。

## 八、TUI 和其他技巧

GDB 提供基于文本界面的 TUI，可以同时显示源代码、反汇编和寄存器：

```text
layout <name>
```

还可以在运行过程中修改变量：

```text
set variable = value
```

调试不同目标（例如用户程序和内核）时，需要切换对应的符号文件：

```text
symbol-file obj/user/<name>
symbol-file obj/kern/kernel
```

## 九、xv6 调试常用命令

```text
break function       # 在函数入口停下
continue             # 继续运行
next                 # 源代码级单步并跳过调用
step                 # 源代码级单步并进入调用
info registers       # 查看寄存器
print variable       # 查看变量
backtrace            # 查看调用链
x/i $pc              # 查看当前汇编指令
```

调试陷阱、系统调用、上下文切换等底层代码时，通常需要结合 stepi、info registers、x/i $pc 和 backtrace 一起使用。

## 核心要点

1. 先理解函数调用如何改变栈和寄存器。
2. 用断点控制程序停在关键位置。
3. 用 step、next、stepi、nexti 观察执行过程。
4. 用 print、x、info registers、info frame 检查状态。
5. 用观察点定位数据到底在哪里被修改。
6. 必要时用 set 修改运行状态进行实验。
7. 多使用 help 和 GDB 手册；讲义只介绍了 GDB 很小的一部分功能。
