# Lab0.5

## 1. 调试过程概述

通过 GDB 的调试，我们可以梳理从通电到执行真正入口点之间的汇编语言执行过程。首先，启动 QEMU 并等待调试连接：

```bash
qemu-system-riscv64 -machine virt -nographic -bios default -device loader,file=ucore.img,addr=0x80200000 -s -S
然后，将本地代码连接到 QEMU：
riscv64-unknown-elf-gdb -ex 'file bin/kernel' -ex 'set arch riscv:rv64' -ex 'target remote localhost:1234'
程序从复位地址 0x1000 开始，跳转到 0x80000000 执行 bootload，然后进入真正的内核执行。

0x1000:      auipc   t0,0x0
0x1004:      addi    a1,t0,32
0x1008:      csrr    a0,mhartid
0x100c:      ld      t0,24(t0)
0x1010:      jr      t0


通过单步调试，我们观察到寄存器的变化：
(gdb) info r mhartid
mhartid        0x0      0
(gdb) info r a0     
a0             0x0      0
(gdb) si
0x0000000000001004 in ?? ()

在 0x1010 处，指令跳转到 t0 寄存器值对应的地址 0x80000000，开始执行 bootload。
