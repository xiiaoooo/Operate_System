# **Lab0.5**

## **一、实验目的**

实验0.5主要讲解最小可执行内核和启动流程。我们的内核主要在 Qemu 模拟器上运行，它可以模拟一台 64 位 RISC-V 计算机。为了让我们的内核能够正确对接到 Qemu 模拟器上，需要了解 Qemu 模拟器的启动流程，还需要一些程序内存布局和编译流程（特别是链接）相关知识。



## 二、实验内容

### 练习1: 使用GDB验证启动流程

为了熟悉使用qemu和gdb进行调试工作,使用gdb调试QEMU模拟的RISC-V计算机加电开始运行到执行应用程序的第一条指令（即跳转到0x80200000）这个阶段的执行过程，说明RISC-V硬件加电后的几条指令在哪里？完成了哪些功能？要求在报告中简要写出练习过程和回答。



### 实验步骤：

首先进入lab0目录中打开两个shell窗口，分别输入：

```
make debug
make gdb
```

将显示：

```
The target architecture is set to "riscv:rv64".
Remote debugging using localhost:1234
0x0000000000001000 in ?? ()
```

表明已经连接到了1234接口，并且计算机加电后，进行复位会先停留在`0x1000`（模拟的处理器的复位地址）处，之后将会启动bootloader。

之后用gdb进行调试，输入`x/10i $pc`查看之后的10条汇编指令：

```
(gdb) x/10i $pc
=> 0x1000:	auipc	t0,0x0          #将当前PC加上立即数0x0存入t0（0x1000）
   0x1004:	addi	a1,t0,32        #t0地址加上立即数32存入a1（0x1020）
   0x1008:	csrr	a0,mhartid      #读取当前mhartid到a0（0）
   0x100c:	ld	t0,24(t0)           #将t0当前值加上24，再将该地址的值存入t0（0x80000000）
   0x1010:	jr	t0                  #跳转到t0（0x80000000）处
   0x1014:	unimp
   0x1016:	unimp
   0x1018:	unimp
   0x101a:	0x8000
   0x101c:	unimp
```

由上可知将通过`jr t0`进入`0x80000000`后，通过`si`指令进行调试，调试至`0x80000000`处，此时对应的指令将启动bootloader，通过输入`x/10i 0x80000000`查看之后的10条汇编指令：

该地址加载的是作为bootloader的OpenSBI.bin，为了加载操作系统内核并启动操作系统。

```
(gdb) si
0x0000000080000000 in ?? ()
(gdb) x/10i 0x80000000
=> 0x80000000:	csrr	a6,mhartid        #读取当前mhartid到a6
   0x80000004:	bgtz	a6,0x80000108     #若a6>0,跳转到0x80000108
   0x80000008:	auipc	t0,0x0            #将当前PC加上立即数0x0存入t0（0x80000008）
   0x8000000c:	addi	t0,t0,1032        #t0加上立即数1032存入t0（0x80000408）
   0x80000010:	auipc	t1,0x0            #将当前PC加上立即数0x0存入t1（0x80000010）
   0x80000014:	addi	t1,t1,-16         #t1减去立即数16存入t1（0x80000000）
   0x80000018:	sd	t1,0(t0)              #将t1的值（0x80000000）存到t0（0x80000408）处
   0x8000001c:	auipc	t0,0x0            #将当前PC加上立即数0x0存入t0（0x8000001c）
   0x80000020:	addi	t0,t0,1020        #t0加上立即数1020存入t0（0x80000400）
   0x80000024:	ld	t0,0(t0)              ##将t0地址的值存入t0（0x80000400）
```

 接下来，为了和上一阶段的OpenSBI对接，需要保证内核第一条指令位于`0x80200000`处，该地址由qemu指定。因此，需要将内核镜像先加载到`0x80200000`处，这里主要通过`break *0x80200000`在该处打断点，并运行到该位置，并查看后续指令：

```
(gdb) break *0x80200000
Breakpoint 1 at 0x80200000: file kern/init/entry.S, line 7.
(gdb) continue
Continuing.

Breakpoint 1, kern_entry () at kern/init/entry.S:7
7	    la sp, bootstacktop
(gdb) x/5i $pc
=> 0x80200000 <kern_entry>:		auipc	sp,0x3                #将当前PC加上立即数0x3存入sp（0x80203000）
   0x80200004 <kern_entry+4>:	mv	sp,sp                     
   0x80200008 <kern_entry+8>:	j	0x8020000c <kern_init>    #跳转至0x80200000c处继续执行
   0x8020000c <kern_init>:		auipc	a0,0x3                #将当前PC加上立即数0x3存入a0（0x8020300c）
   0x80200010 <kern_init+4>:	addi	a0,a0,-4
```

可以看到，代码实际上是在执行应用程序的指令，需要跳转到`0x80200000`后执行，不属于加电之后的复位阶段。其作用主要是操作系统内核的初始化等等，需要从入口kern_entry跳转到初始化阶段kern_init继续执行。



## 三、重要知识点

应用程序执行流程：

- 加电，复位，从`0x1000`处开始执行，之后跳转到`0x80000000`
- 启动作为bootloader的OpenSBI，跳转到`0x80200000`，运行`kern_entry`，后进入`kern_init`继续执行 
运行`make gdb`之前需要先运行`make debug`，运行`make debug`后会调用命令构建程序，之后将调试其构建的二进制文件，否则将会报错：

```
localhost:1234: Connection timed out.
```

