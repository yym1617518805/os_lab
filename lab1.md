# lab1

## 练习一
阅读 kern/init/entry.S内容代码，结合操作系统内核启动流程，说明指令 la sp, bootstacktop 完成了什么操作，目的是什么？ tail kern_init 完成了什么操作，目的是什么？
la sp, bootstacktop 指令将bootstacktop标签的内存地址加载到栈指针寄存器sp中，目的是将栈指针寄存器sp的值设置为内核启动栈的顶部，从而为内核代码的执行提供栈空间。
tail kern_init 指令的作用是跳转到标签 kern_init 处，这里的 kern_init 是内核初始化函数的入口点。目的是开始执行操作系统的初始化过程。内核初始化函数负责完成各种初始化任务，例如初始化设备驱动程序、建立内存管理机制、创建进程或线程等。


## 练习2

请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写kern/trap/trap.c函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字”100 ticks”，在打印完10行后调用sbi.h中的shut_down()函数关机。
要求完成问题1提出的相关函数实现，提交改进后的源代码包（可以编译执行），并在实验报告中简要说明实现过程和定时器中断中断处理的流程。实现要求的部分代码后，运行整个系统，大约每1秒会输出一次”100 ticks”，输出10行。

每次系统接收到时钟信号并触发中断后，执行时钟中断处理函数，首先通过虚拟外设定时器的驱动文件driver/clock.h的clock_set_next_event()函数设置下一次时钟中断的时间，使用静态变量int ticks在每次触发时钟时实现计数，当计数器ticks到达100时，触发打印次数计数器增加，同时检查如果打印10次，则调用sbi_shutdown()函数关机。
程序片段：

```
  clock_set_next_event();
            ticks++;
            if(ticks==TICK_NUM)
            {
                print_ticks();
                ticks=0;
                num++;
                if(num==10) sbi_shutdown()
             }
```
运行结果

```
+ setup timer interrupts
100 ticks
100 ticks
100 ticks
100 ticks
100 ticks
100 ticks
100 ticks
100 ticks
100 ticks
100 ticks
```



## Challenge 1



### Part 1 分析过程。
中断机制就是无论现在正在做什么工作，处理器都需要停止当前点工作，并且将当前的状态保存下来。当处理完中断的时候，就需要将寄存器的状态恢复到原来的样子，继续执行程序。

首先通电后到入口函数的操作和lab0.5类似都是到地址0x80200000执行entry.S里面的汇编代码首先是执行la sp, bootstacktop，将sp寄存器的值设置为bootstack堆栈的起始地址，这样能够初始化一个区域来存储局部变量和函数返回的地址，之后进入C语言编写的内核kern_init

之后分析该函数
```
int kern_init(void) {
    extern char edata[], end[];
    memset(edata, 0, end - edata);

    cons_init();  // init the console

    const char *message = "(THU.CST) os is loading ...\n";
    cprintf("%s\n\n", message);

    print_kerninfo();

    // grade_backtrace();

    idt_init();  // init interrupt descriptor table

    // rdtime in mbare mode crashes
    clock_init();  // init clock interrupt

    intr_enable();  // enable irq interrupt
    
    while (1)
        ;
}
```
在函数idt_init()执行之前，该内核会有一些特定的输出，包括硬件的配置等等。

接下来分析三个函数：
idt_init()函数用于初始化中断描述符表（IDT）和相关的异常处理机制，write_csr(stvec, &__alltraps)：将 stvec 寄存器的值设置为 __alltraps 的地址。stvec 寄存器指向异常处理程序的入口点。当系统发生异常时，CPU 会跳转到这个地址执行异常处理逻辑。
```
clock_init()的详细函数如下：
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
```
相当于发起一个时钟中断，并且输出相关的中断信息。

intr_enable()函数是给上面的中断加一个是能，使得时钟中断有运行的权限。
之后就是在while(1)循环中不断执行中断过程，并且在处理中断的时候需要调用clock_set_next_event();来告诉底层下一个中断的时间，知道达到我们的要求，最后关机。






下面是中断trap执行的过程。
```
void trap(struct trapframe *tf) { trap_dispatch(tf); }
trap函数就是传递了一个指向trapframe类的一个指针，这个类中有后续处理中断所需要的信息。
struct pushregs {
    uintptr_t zero;  // Hard-wired zero
    uintptr_t ra;    // Return address
    uintptr_t sp;    // Stack pointer
    uintptr_t gp;    // Global pointer
    uintptr_t tp;    // Thread pointer
    uintptr_t t0;    // Temporary
    uintptr_t t1;    // Temporary
    uintptr_t t2;    // Temporary
    uintptr_t s0;    // Saved register/frame pointer
    uintptr_t s1;    // Saved register
    uintptr_t a0;    // Function argument/return value
    uintptr_t a1;    // Function argument/return value
    uintptr_t a2;    // Function argument
    uintptr_t a3;    // Function argument
    uintptr_t a4;    // Function argument
    uintptr_t a5;    // Function argument
    uintptr_t a6;    // Function argument
    uintptr_t a7;    // Function argument
    uintptr_t s2;    // Saved register
    uintptr_t s3;    // Saved register
    uintptr_t s4;    // Saved register
    uintptr_t s5;    // Saved register
    uintptr_t s6;    // Saved register
    uintptr_t s7;    // Saved register
    uintptr_t s8;    // Saved register
    uintptr_t s9;    // Saved register
    uintptr_t s10;   // Saved register
    uintptr_t s11;   // Saved register
    uintptr_t t3;    // Temporary
    uintptr_t t4;    // Temporary
    uintptr_t t5;    // Temporary
    uintptr_t t6;    // Temporary
};

struct trapframe {
    struct pushregs gpr;
    uintptr_t status;
    uintptr_t epc;
    uintptr_t badvaddr;
    uintptr_t cause;
};
```
这是这个类的全部信息展示，可以看到trapframe中又包含了一个类，存储着寄存器的值，用来保存当前状态。

```
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
之后就是通过传入的类中cause关键字的值来判断到底是中断处理还是异常处理。
Interrupts（中断）：由硬件产生的异步信号，请求CPU注意，通常由外部设备触发，可以随时中断当前执行的程序。
Exceptions（异常）：由程序自身产生的同步事件，通常由于程序错误（如除零、非法指令）导致，发生在特定指令执行时，与程序状态密切相关。

这里我们处理的是中断interrupts
之后进入interrupt_handler()阶段，其中会对我们中断类型作进一步的划分，进行处理。


### Part2 问题解答：
首先是异常处理的机制，在上面已经有详细的分析了，之后就是mov a0，sp的目的，本身这条语句就是将sp的值赋值给a0，在这之后有一个函数调用，调用的trap函数，前面分析过，其需要一个trapframe类类型指针，该条指令就是有这个作用。

接下来就是SAVE_ALL中寄存器保存在栈中的位置是什么确定的，其实观察汇编程序我们能够得知是通过sp栈帧的地址加上一定的偏移量决定的。
```
STORE x0, 0*REGBYTES(sp)
    STORE x1, 1*REGBYTES(sp)
    STORE x3, 3*REGBYTES(sp)
    STORE x4, 4*REGBYTES(sp)
    STORE x5, 5* REGBYTES (sp)
    STORE x6, 6*REGBYTES(sp)
    STORE x7, 7*REGBYTES(sp)
    STORE x8, 8*REGBYTES(sp)
STORE x9, 9*REGBYTES(sp)
```
这是一些通用的寄存器存储的位置在RISC-V中位置，在RISCV-V三十二位中REGBYTES为定值4，这样就能够得出x1存储在栈中的地址为sp+4。

此外还有其他一些特殊的寄存器的地址也是通过偏移量来寻址的
```
csrrw s0, sscratch, x0
    csrr s1, sstatus
    csrr s2, sepc
    csrr s3, sbadaddr
    csrr s4, scause

    STORE s0, 2*REGBYTES(sp)
    STORE s1, 32*REGBYTES(sp)
    STORE s2, 33*REGBYTES(sp)
    STORE s3, 34*REGBYTES(sp)
STORE s4, 35*REGBYTES(sp)
```
对于是否需要保存所有寄存器
对于任何中断，__alltraps 中需要保存所有寄存器，以确保在中断处理完成后，CPU能够恢复到中断发生前的状态。这是因为中断处理程序可能会使用和修改寄存器，如果不保存原寄存器的值，处理完成后返回时可能会导致程序状态丢失或错误。因此，保存寄存器是确保系统稳定性和可靠性的必要步骤。


## Challenge 2
回答：在trapentry.S中汇编代码 csrw sscratch, sp；csrrw s0, sscratch, x0实现了什么操作，目的是什么？save all里面保存了stval scause这些csr，而在restore all里面却不还原它们？那这样store的意义何在呢？
csrw sscratch, sp
将当前的堆栈指针（sp）的值写入到 sscratch 寄存器中。sscratch是一个在RISC-V架构中用于异常处理的专用寄存器，主要用于在trap处理期间保存临时状态或数据。sscratch存储原始的 sp 值目的是在异常处理完成后，能够恢复到原来的堆栈环境。
csrrw s0, sscratch, x0
将 sscratch 寄存器的值读到 s0 寄存器中，同时将 x0（即零寄存器）写入到 sscratch 寄存器。将 sscratch 寄存器清零，这通常用作一个标志，表示当前正在处理一个来自内核的异常（如果是从用户空间进入的异常，sscratch 将不会被清零），区分不同类型的异常。
在save all中保存stval和scause等CSR（控制状态寄存器）是为了在中断处理中使用其中的信息进行处理，因为在使用完之后这些信息不再具有价值，因此在restore all里面不还原它们。。
scause（异常原因寄存器）包含了异常的类型和原因。这个信息在异常处理时重要，但恢复时不需要重新设置，因为异常的原因已经处理完毕。
stval保存与trap相关的某些关键信息。


## Challenge 3


kern\init\init.c
mret是RISC-V中的特权指令，用于从机器模式（M-mode）返回到先前的模式，因此，将init.c中的ker_init函数改为：
```
int kern_init(void) {
    extern char edata[], end[];
    memset(edata, 0, end - edata);
    cons_init();  // init the console
    const char *message = "(THU.CST) os is loading ...\n";
    cprintf("%s\n\n", message);
    print_kerninfo();
    // grade_backtrace();
idt_init();  // init interrupt descriptor table
// rdtime in mbare mode crashes
clock_init();  // init clock interrupt
intr_enable();  // enable irq interrupt
    asm("mret");  // 测试非法指令异常
    asm("ebreak");  // 测试断点异常
    while (1)
        ;
}
```
其中，我们加入了
```
asm("mret");  
    asm("ebreak");
```
以实现非法指令异常的触发和断点异常的触发。

kern/trap/trap.c
```
void exception_handler(struct trapframe *tf) {
    switch (tf->cause) {
        case CAUSE_MISALIGNED_FETCH:
            break;
        case CAUSE_FAULT_FETCH:
            break;
        case CAUSE_ILLEGAL_INSTRUCTION:
             // 非法指令异常处理
             /* LAB1 CHALLENGE3   YOUR CODE :  */
            /*(1)输出指令异常类型（ Illegal instruction）
             *(2)输出异常指令地址
             *(3)更新 tf->epc寄存器
            */
            cprintf("Exception type:Illegal instruction\n");
// tf->epc寄存器保存了触发中断的指令的地址，因此，输出该寄存器的值就得到中断指令的地址
            cprintf("Illegal instruction caught at 0x%016llx\n", tf->epc);

            // 内部地址偏移+4，以跳过当前的4字节指令。
            tf->epc += 4;
            break;
        case CAUSE_BREAKPOINT:
            //断点异常处理
            /* LAB1 CHALLLENGE3   YOUR CODE :  */
            /*(1)输出指令异常类型（ breakpoint）
             *(2)输出异常指令地址
             *(3)更新 tf->epc寄存器
            */
            cprintf("Exception type: breakpoint\n");

// tf->epc寄存器保存了触发中断的指令的地址，因此，输出该寄存器的值就得到中断指令的地址
            cprintf("ebreak caught at 0x%016llx\n", tf->epc);
            // 内部地址偏移+4，以跳过当前的4字节指令。
            tf->epc += 4;
            break;
        case CAUSE_MISALIGNED_LOAD:
            break;
        case CAUSE_FAULT_LOAD:
            break;
        case CAUSE_MISALIGNED_STORE:
            break;
        case CAUSE_FAULT_STORE:
            break;
        case CAUSE_USER_ECALL:
            break;
        case CAUSE_SUPERVISOR_ECALL:
            break;
        case CAUSE_HYPERVISOR_ECALL:
            break;
        case CAUSE_MACHINE_ECALL:
            break;
        default:
            print_trapframe(tf);
            break;
    }
}
```
其中，tf->epc寄存器保存了触发中断的指令的地址，因此，输出该寄存器的值就得到中断指令的地址。再将tf->epc加 4，以跳过当前的 4 字节指令。

