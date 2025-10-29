# Lab3 实验报告

### 练习1：完善中断处理 （需要编程）

>请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写kern/trap/trap.c函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字”100 ticks”，在打印完10行后调用sbi.h中的shut_down()函数关机。
>
>要求完成问题1提出的相关函数实现，提交改进后的源代码包（可以编译执行），并在实验报告中简要说明实现过程和定时器中断中断处理的流程。实现要求的部分代码后，运行整个系统，大约每1秒会输出一次”100 ticks”，输出10行。

**实现过程：**
先调用clock_set_next_event()安排下一次时钟中断（防止丢失后续中断），维护中断计数器。
如果当计数器加到一百时，调用print_ticks()打印一行"100 ticks"，然后把ticks清零，维护打印次数计数器。
如果当打印次数计数器达到十次的时候，调用SBI的关机函数sbi_shutdown()使系统关机。

**定时器中断处理流程：**
内核在初始化时通过clock_set_next_event()调用sbi_set_timer()向OpenSBI请求设置下一次触发时间；OpenSBI在平台计时器（CLINT 的mtimecmp）上设置比较值，当mtime达到mtimecmp时硬件产生机器模式定时器中断，接着OpenSBI将事件交付为Supervisor定时器中断（IRQ_S_TIMER），从而使内核可以处理该中断；然后CPU跳转到汇编异常入口（trapentry.S/__alltraps），保存寄存器构造trapframe并调用trap()，识别中断类型为IRQ_S_TIMER后进入interrupt_handler的相应分支；在该分支中实现相应的中断处理（即练习一要补充的中断处理）。

### 扩展练习 Challenge1：描述与理解中断流程

>回答：描述ucore中处理中断异常的流程（从异常的产生开始），其中mov a0，sp的目的是什么？SAVE_ALL中寄寄存器保存在栈中的位置是什么确定的？对于任何中断，__alltraps 中都需要保存所有寄存器吗？请说明理由。

**解答：**

### 扩展练习 Challenge2：理解上下文切换机制

>回答：在trapentry.S中汇编代码 csrw sscratch, sp；csrrw s0, sscratch, x0实现了什么操作，目的是什么？save all里面保存了stval scause这些csr，而在restore all里面却不还原它们？那这样store的意义何在呢？

**解答：**

### 扩展练习Challenge3：完善异常中断

>编程完善在触发一条非法指令异常和断点异常，在 kern/trap/trap.c的异常处理函数中捕获，并对其进行处理，简单输出异常类型和异常指令触发地址，即“Illegal instruction caught at 0x(地址)”，“ebreak caught at 0x（地址）”与“Exception type:Illegal instruction"，“Exception type: breakpoint”。

**解答：**