# Lab0.5

## 1. 调试过程概述

Lab0.5的要求就是通过gdb的调试，梳理出从通电到执行到真正入口点之间汇编语言的执行过程。
提前启动qemu：qemu-system-riscv64 -machine virt -nographic -bios default -device loader,file=ucore.img,addr=0x80200000 -s -S

通过上面的指令使得qemu处于等待的阶段。

之后将本地对代码连接到qemu：riscv64-unknown-elf-gdb -ex 'file bin/kernel' -ex 'set arch riscv:rv64' -ex 'target remote localhost:1234'

这样就能使用gdb的指令了。

首先是bootload之前的汇编代码：



```
=> 0x1000:      auipc   t0,0x0
   0x1004:      addi    a1,t0,32
   0x1008:      csrr    a0,mhartid
   0x100c:      ld      t0,24(t0)
   0x1010:      jr      t0
   0x1014:      unimp
   0x1016:      unimp
   0x1018:      unimp
   0x101a:      0x8000
   0x101c:      unimp
```
之后经过单步汇编代码调试，我们得到了如下信息：


```
(gdb) info r mhartid
mhartid        0x0      0
(gdb) info r a0     
a0             0x0      0
(gdb) si
0x0000000000001004 in ?? ()
(gdb) info r a0
a0             0x0      0
(gdb) info r a1
a1             0x0      0
(gdb) info r t0
t0             0x1000   4096
(gdb) si
0x0000000000001008 in ?? ()
(gdb) info r t0
t0             0x1000   4096
(gdb) info r a0
a0             0x0      0
(gdb) info r a1
a1             0x1020   4128
(gdb) info r mhartid
mhartid        0x0      0
(gdb) si
0x000000000000100c in ?? ()
(gdb) info r mhartid
mhartid        0x0      0
(gdb) info r a1     
a1             0x1020   4128
(gdb) info r a0
a0             0x0      0
(gdb) si
0x0000000000001010 in ?? ()
(gdb) info r mhartid
mhartid        0x0      0
(gdb) info r a0     
a0             0x0      0
(gdb) info r a1     
a1             0x1020   4128
(gdb) info r t0     
t0             0x80000000       2147483648
(gdb) si       
0x0000000080000000 in ?? ()
```


之后是当指令跳转到bootload时：

```
(gdb) x/10i $pc
=> 0x80000000:  csrr    a6,mhartid
   0x80000004:  bgtz    a6,0x80000108
   0x80000008:  auipc   t0,0x0
   0x8000000c:  addi    t0,t0,1032
   0x80000010:  auipc   t1,0x0
   0x80000014:  addi    t1,t1,-16
   0x80000018:  sd      t1,0(t0)
   0x8000001c:  auipc   t0,0x0
   0x80000020:  addi    t0,t0,1020
   0x80000024:  ld      t0,0(t0)
```
csrr a6, mhartid (0x80000000)：
•	从mhartid寄存器读取当前硬件线程ID，存入a6。
bgtz a6, 0x80000108 (0x80000004)：
•	如果a6大于零，则跳转到地址0x80000108。否则，继续执行下一条指令。
之后到0x80200000之间执行的汇编代码都涉及内存和寄存其的初始话。


之后我们在0x80200000添加断点，观察接下来需要执行的汇编指令。

```
(gdb) x/10i $pc
=> 0x80200000 <kern_entry>:     auipc   sp,0x3
   0x80200004 <kern_entry+4>:   mv      sp,sp
   0x80200008 <kern_entry+8>:   j       0x8020000a <kern_init>
   0x8020000a <kern_init>:      auipc   a0,0x3
   0x8020000e <kern_init+4>:    addi    a0,a0,-2
   0x80200012 <kern_init+8>:    auipc   a2,0x3
   0x80200016 <kern_init+12>:   addi    a2,a2,-10
   0x8020001a <kern_init+16>:   addi    sp,sp,-16
   0x8020001c <kern_init+18>:   li      a1,0
   0x8020001e <kern_init+20>:   sub     a2,a2,a0
```

•  auipc sp, 0x3 (0x80200000 <kern_entry>)：
•	将当前程序计数器（PC）的高20位与0x3结合，结果存入sp。假设PC为0x80200000，sp的高位被设置。
•  mv sp, sp (0x80200004 <kern_entry+4>)：
•	将sp的值移动到sp自己，即此操作没有实际改变，通常用于保持对齐或占位。
•  j 0x8020000a (0x80200008 <kern_entry+8>)：
•	无条件跳转到地址0x8020000a，进入kern_init函数。


我们对比.S代码中的指令:
```
kern_entry:
    la sp, bootstacktop

tail kern_init
```
与汇编指令：

```
0x80200000 <kern_entry>:     auipc   sp,0x3
   0x80200004 <kern_entry+4>:   mv      sp,sp
   0x80200008 <kern_entry+8>:   j       0x8020000a <kern_init>
```
是一一对应的，实现了•  初始化堆栈指针sp为指定的堆栈顶位置。然后直接跳转到内核初始化函数kern_init，开始执行内核的初始化过程。


## 问题分析
首先加电后，PC复位地址为0x1000，之后一直执行到0x1010，在此期间调整好寄存器t0的数值，并且将其通过提前设置好的规则形成一个新地址，将该地址的数值（即0x80000000）读取出来，这样得到bootload的入口地址，之后bootload执行后进入entry.S，按照其设定的顺序，先初始化堆栈指针sp在栈顶的位置。之后跳转到内核初始化函数kern_init,开始运行操作系统，在OpenSBI中输出(THU.CST) os is loading ...，之后进入while(1)的死循环。
