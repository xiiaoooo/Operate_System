### 1 练习1

阅读 kern/init/entry.S内容代码，结合操作系统内核启动流程，说明指令 la sp, bootstacktop 完成了什么操作，目的是什么？ tail kern_init 完成了什么操作，目的是什么？

1. `la sp, bootstacktop`

该指令将bootstacktop这个标签所代表的地址加载给sp栈顶寄存器，从而实现内存栈的初始化。在操作系统的引导过程中，最初的栈通常是一个非常小的栈，用于执行引导加载程序（bootloader）的一些基本操作。bootstacktop就是引导栈的起始地址。

2. `tail kern_init`

该指令通过尾调用的方式跳转执行kern_init这个函数进行内核的一系列初始化操作。尾调用就是在调用函数后，不会返回到原来的函数调用位置，而是将控制权传递给被调用的函数，使其成为新的执行上下文。

### 2 练习2

补充代码

``` C
 clock_set_next_event();
            ticks++;
            if(ticks == 100)
            {
                print_ticks();
                ticks = 0;
                num++;
            }
            if(num == 10)
            {
                sbi_shutdown();
            }
            /*(1)设置下次时钟中断- clock_set_next_event()
             *(2)计数器（ticks）加一
             *(3)当计数器加到100的时候，我们会输出一个`100ticks`表示我们触发了100次时钟中断，同时打印次数（num）加一
            * (4)判断打印次数，当打印次数为10时，调用<sbi.h>中的关机函数关机
            */
```

实现过程

- 时钟中断发生，`clock_set_next_event`,设置下一次时钟中断
- ticks 计数器加一
- 当ticks等于100时，使用函数`print_ticks`打印“100ticks”,将ticks重置为0，打印次数num+1
- num==0时关机

时钟中断产生过程

```C
static uint64_t timebase = 100000;

/* *
 * clock_init - initialize 8253 clock to interrupt 100 times per second,
 * and then enable IRQ_TIMER.
 * */
void clock_init(void) {
    // enable timer interrupt in sie
    set_csr(sie, MIP_STIP);
    // divided by 500 when using Spike(2MHz)
    // divided by 100 when using QEMU(10MHz)
    // timebase = sbi_timebase() / 500;
    clock_set_next_event();

    // initialize time counter 'ticks' to zero
    ticks = 0;

    cprintf("++ setup timer interrupts\n");
}

void clock_set_next_event(void) { sbi_set_timer(get_cycles() + timebase); }
```

### 3 拓展练习Challenge1

#### ucore中处理中断机制的流程

初始化设置： `init.c`文件中调用`idt_init`函数初始化中断，将`sscrach`寄存器赋值为0，表示处于内核态;`trapentry.S`中的中断入口点处定义了一个全局标签`_alltraps`,这里还将`stvec`寄存器设置为`&_alltraps`。

``` C
void idt_init(void) {
    extern void __alltraps(void);
    /* Set sscratch register to 0, indicating to exception vector that we are
     * presently executing in the kernel */
    write_csr(sscratch, 0);
    /* Set the exception vector address */
    write_csr(stvec, &__alltraps);
}
```

- 异常产生后，会跳转到寄存器`stvec`保存的地址执行指令，由于内核初始化时将该寄存器设置为`__alltraps`，所以会跳转到`trapentry.S`中的`alltraps`标签处执行。

```
__alltraps:
    SAVE_ALL

    move  a0, sp
    jal trap
    # sp should be the same as before "jal trap"

    .globl __trapret
__trapret:
    RESTORE_ALL
    # return from supervisor call
    sret
```

- 保存上下文 通过汇编宏`SAVE_ALL`将一个保存了所有寄存器`trapFrame`的结构体放入栈中，实现保存上下文

- 函数trap根据trapframe结构体中的`cause`来区分中断和异常，中段调用`interrupt_hander`,异常调用 `exception_hander`

```C
void trap(struct trapframe *tf) { trap_dispatch(tf); }

static inline void trap_dispatch(struct trapframe *tf) {
    if ((intptr_t)tf->cause < 0) {
        // interrupts
        interrupt_handler(tf);
    } else {
        // exceptions
        exception_handler(tf);
    }
}
```

- 中断和异常处理完成后，回到trapentry.S文件中的_trapret标签位置，使用`RESTORE_ALL`恢复上下文，继续执行中断前的操作。

**question1:** 此处`move  a0, sp`的作用
该指令是把保存上下文之后的栈顶指针寄存器赋值给a0寄存器，a0寄存器是参数寄存器，这样就可以把当前的中断帧作为参数传递给中断处理程序，从而实现对中断的处理。

**question2:** `SAVE_AL`L中寄存器保存在栈中的位置是什么确定的。

各个寄存器保存的位置是通过栈顶寄存器sp来索引的。在保存上下文之前我们首先通过指令`addi sp, sp, -36 * REGBYTES`，在内存中开辟出了保存上下文的内存区域，然后我们通过栈顶指针sp来访问该段区域的不同位置，从而把对应的寄存器保存在栈中。

**question3:** 对于任何中断，__alltraps 中都需要保存所有寄存器吗？
需要保存所有的寄存器。因为这些寄存器都将用于函数trap参数的一部分，如果不保存所有寄存器，函数参数不完整。如果修改trap的参数的结构体的定义，可以不需要存所有的寄存器

### 4 拓展练习Challenge2

**Q1：**`csrw sscratch, sp`；`csrrw s0, sscratch, x0`实现了什么操作，目的是什么？

`csrw sscratch, sp`：表示将栈顶指针sp的值赋值给sscratch寄存器

`csrrw s0, sscratch, x0`：表示将寄存器sscratch赋值给s0的同时将sscratch置0

**Q2：**`save all`里面保存了stval scause这些csr，而在`restore all`里面却不还原它们？那这样store的意义何在呢？

这两个csr的作用只在处理中断程序时有所显现，能够用来查询相关信息，在处理结束后就没有特定作用了，后续也会重新赋值，所以没有必要再进行还原。

### 5 拓展练习Challenge3

### 6 实验中的知识点

#### (1)中断的分类

- 异常(Exception)，指在执行一条指令的过程中发生了错误，此时我们通过中断来处理错误。最常见的异常包括：访问无效内存地址、执行非法指令(除零)、发生缺页等。他们有的可以恢复(如缺页)，有的不可恢复(如除零)，只能终止程序执行。

- 陷入(Trap)，指我们主动通过一条指令停下来，并跳转到处理函数。常见的形式有通过ecall进行系统调用(syscall)，或通过ebreak进入断点(breakpoint)。

- 外部中断(Interrupt)，简称中断，指的是 CPU 的执行过程被外设发来的信号打断，此时我们必须先停下来对该外设进行处理。典型的有定时器倒计时结束、串口收到数据等。

#### (2)RISCV的特权级别

- 机器模式（M模式）
  M模式全称为Machine mode（机器模式）运行在这个模式下的程序为最高权限，它属于RISC-V里的最高权限模式，它具有访问所有资源的权限，它的代码是百分百可信的，通常运行在这个模式下的为固件和操作系统内核。

- 监管者模式（S模式）
  S模式全称“Supervisor mode”（监管者模式）监管者模式通常是用来运行操作系统内核，它的权限要比M模式低，它无法直接操作特殊寄存器和某些资源，但内核通常会运行在S和M两个模式之间，在最初系统启动阶段内核是运行在M模式下的，在这个模式下内核需要初始化所有的硬件资源和进行内存管理等等，当初始化完成之后会切换到S模式下，通常内核里有一段代码是运行在M模式下和S模式下，M模式下的代码为S模式下的代码提供访问硬件资源的能力。而S模式下的内核主要是为应用程序提供系统调用以及上下文切换。

- 用户模式（U模式）
  U模式全称为“User Model”（用户模式）为级别最低的模式，它不能访问硬件资源，只能访问某些通用寄存器和通用指令，一般用于执行应用程序。

#### (3)RISCV 与中断有关的状态控制器

`sstatus` 决定cpu是否中断
`stvec` 中断向量表基址，通过映射lab1 王南钧+闫恒瑞+彭晓晓处理不同种类的中断处理程序
`cause` 记录中断产生的原因。

### 7 实验结果

输入`make qemu`之后结果如下：![](C:\Users\彭小小\Desktop\.png)

输入`make grade`之后结果如下：

![1](C:\Users\彭小小\Desktop\1.png)

