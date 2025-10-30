# Lab3 实验报告

### 练习1：完善中断处理

> 编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写kern/trap/trap.c函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字”100 ticks”，在打印完10行后调用sbi.h中的shut_down()函数关机。

**实现过程：** 先调用clock_set_next_event()安排下一次时钟中断（防止丢失后续中断），维护中断计数器。 如果当计数器加到一百时，调用print_ticks()打印一行"100 ticks"，然后把ticks清零，维护打印次数计数器。 如果当打印次数计数器达到十次的时候，调用SBI的关机函数sbi_shutdown()使系统关机。

**定时器中断处理流程：** 内核在初始化时通过clock_set_next_event()调用sbi_set_timer()向OpenSBI请求设置下一次触发时间；OpenSBI在平台计时器（CLINT 的mtimecmp）上设置比较值，当mtime达到mtimecmp时硬件产生机器模式定时器中断，接着OpenSBI将事件交付为Supervisor定时器中断（IRQ_S_TIMER），从而使内核可以处理该中断；然后CPU跳转到汇编异常入口（trapentry.S/__alltraps），保存寄存器构造trapframe并调用trap()，识别中断类型为IRQ_S_TIMER后进入interrupt_handler的相应分支；在该分支中实现相应的中断处理（即练习一要补充的中断处理）。

**实验结果**

```
swiftiexh@DESKTOP-C6899HL:~/labcode/lab3$ make qemu
+ cc kern/trap/trap.c
+ cc kern/trap/trapentry.S
+ ld bin/kernel
riscv64-unknown-elf-objcopy bin/kernel --strip-all -O binary bin/ucore.img

OpenSBI v0.4 (Jul  2 2019 11:53:53)
   ____                    _____ ____ _____
  / __ \                  / ____|  _ \_   _|
 | |  | |_ __   ___ _ __ | (___ | |_) || |
 | |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
 | |__| | |_) |  __/ | | |____) | |_) || |_
  \____/| .__/ \___|_| |_|_____/|____/_____|
        | |
        |_|

Platform Name          : QEMU Virt Machine
Platform HART Features : RV64ACDFIMSU
Platform Max HARTs     : 8
Current Hart           : 0
Firmware Base          : 0x80000000
Firmware Size          : 112 KB
Runtime SBI Version    : 0.1

PMP0: 0x0000000080000000-0x000000008001ffff (A)
PMP1: 0x0000000000000000-0xffffffffffffffff (A,R,W,X)
DTB Init
HartID: 0
DTB Address: 0x82200000
Physical Memory from DTB:
  Base: 0x0000000080000000
  Size: 0x0000000008000000 (128 MB)
  End:  0x0000000087ffffff
DTB init completed
(THU.CST) os is loading ...
Special kernel symbols:
  entry  0xffffffffc0200054 (virtual)
  etext  0xffffffffc0201f34 (virtual)
  edata  0xffffffffc0206028 (virtual)
  end    0xffffffffc02064a0 (virtual)
Kernel executable memory footprint: 26KB
memory management: default_pmm_manager
physcial memory map:
  memory: 0x0000000008000000, [0x0000000080000000, 0x0000000087ffffff].
check_alloc_page() succeeded!
satp virtual address: 0xffffffffc0205000
satp physical address: 0x0000000080205000
++ setup timer interrupts
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

### 扩展练习 Challenge1：描述与理解中断流程

> 描述ucore中处理中断异常的流程（从异常的产生开始），其中 mov a0，sp的目的是什么？SAVE_ALL中寄存器保存在栈中的位置是什么确定的？对于任何中断，__alltraps 中都需要保存所有寄存器吗？请说明理由。

**解答 :**

- **ucore中处理中断异常的流程**
  1. 硬件触发阶段：
      当硬件检测到中断或异常时，自动执行以下动作：
     - 将触发点的 PC 写入 sepc；
     - 更新 sstatus：SPIE ← SIE、SIE ← 0，并设置 SPP 为之前的特权级；
     - 将中断或异常原因写入 scause，若有相关地址则写入 sbadaddr。
  2. 异常入口跳转：
      硬件根据 stvec 跳转到异常入口向量 _alltraps（在 idt_init 中设置）。
  3. 汇编层保存上下文：
      _alltraps 中通过 SAVE_ALL 宏保存全部通用寄存器和关键 CSR，在当前栈上形成一个完整的 trapframe。
  4. 调用 C 层分发函数：
      汇编入口将 sp 传递给 a0，然后执行 jal trap 调用 C 语言层的 trap 函数。
  5. C 层分发处理：
      trap 调用 trap_dispatch，根据 scause 判断中断或异常类型并执行相应的处理函数。
  6. 返回汇编层：
      C 层处理完成后返回汇编入口，返回点位于 __trapret 之后。
  7. 恢复上下文与返回：
      汇编层通过 RESTORE_ALL 宏恢复通用寄存器和关键 CSR，最后执行 sret 返回到异常前的执行点。
- **mov a0，sp  的目的**：传参，按 RISC‑V 的函数调用约定，a0 是第 1 个参数寄存器。这条指令把当前的栈指针（此时指向刚刚由 SAVE_ALL 在栈上构造好的 trapframe）作为参数传给 C 层的 trap(struct trapframe *tf)。
- **SAVE_ALL 中寄存器保存在栈中的位置** ：由 STORE 指令的槽号定义，C层 trapframe 和 pushregs中的定义顺序需要与之对应，后续这些寄存器都要作为函数 trap 的参数的具体内容。
- **对于任何中断，__alltraps 中是否需要保存所有寄存器** : 是的，因为中断/异常是异步发生的，可能打断任意指令周期和任意代码（用户态或内核态），必须把被打断上下文完整保存才能正确恢复执行并保证被打断程序/内核状态不被破坏。

### 扩展练习 Challenge2：理解上下文切换机制

> 在trapentry.S中汇编代码 csrw sscratch, sp；csrrw s0, sscratch, x0实现了什么操作，目的是什么？save all里面保存了stval scause这些csr，而在restore all里面却不还原它们？那这样store的意义何在呢？

**解答 ：**

- 在 trapentry.S 中汇编代码 csrw sscratch, sp；csrrw s0, sscratch, x0 实现了什么操作，目的是什么?

  - csrw sscratch, sp：将 sp 的值赋值给 sscratch
  - csrrw s0, sscratch, x0：将 sscratch 赋值给 s0 ，将 sscratch 置0

  目的：使用  s0  来表示函数调用前栈顶的位置，之后在 2*REGBYTES(sp) 保存 。将 sscratch 置 0 ，作为 “已进入内核 trap 处理” 的标志，这样如果在处理期间发生递归/嵌套异常，新的入口读到 sscratch==0 就能识别出是从内核上下文来的递归 trap。

- save all 里面保存了 stval、scause 这些 csr ，而在 restore all 里面却不还原它们？那这样 store 的意义何在呢？

  - 不还原的原因：因为它们包含有关导致异常或中断的信息，而异常已经由 trap处理过了，没有必要再去还原。
  - 意义：将这些状态寄存器作为参数的一部分传递给 trap 函数，一是函数中需要用，二是确保在处理异常或中断时能够保留关键的执行上下文，以供进一步处理或记录异常信息。

### 扩展练习 Challenge3：完善异常中断

> 编程完善在触发一条非法指令异常和断点异常，在 kern/trap/trap.c的异常处理函数中捕获，并对其进行处理，简单输出异常类型和异常指令触发地址，即“Illegal instruction caught at 0x(地址)”，“ebreak caught at 0x（地址）”与“Exception type:Illegal instruction"，“Exception type: breakpoint”。

**实现过程 ：**

**结果 :**







