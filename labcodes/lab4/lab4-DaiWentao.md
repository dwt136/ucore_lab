# Lab4 report

## 练习1
练习1.0. 请在实验报告中简要说明你的设计实现过程。
```
首先调用alloc_proc函数来通过kmalloc函数获得proc_struct结构的一块内存块，作为第0个进程控制块。并把proc进行初步初始化（即把proc_struct中的各个成员变量清零）。但有些成员变量设置了特殊的值，具体有：
proc->state = PROC_UNINIT;  设置进程为“初始”态
proc->pid = -1;             设置进程pid的未初始化值
proc->cr3 = boot_cr3;       使用内核页目录表的基址
```

练习1.1. 请说明proc_struct中struct context context和struct trapframe *tf成员变量含义和在本实验中的作用是啥？（提示通过看代码和编程调试可以判断出来）
```
context是用来存储进程上下文的结构体，在线程切换时要保存原进程的上下文到该结构体，并从该结构体中恢复切换到的进程的上下文；tf是指向中断帧结构体的指针。
```

## 练习2
练习2.0. 请在实验报告中简要说明你的设计实现过程
```
分配并初始化进程控制块（alloc_proc函数），分配并初始化内核栈（setup_stack函数），根据clone_flag标志复制或共享进程内存管理结构（copy_mm函数），设置进程在内核（将来也包括用户态）正常运行和调度所需的中断帧和执行上下文（copy_thread函数），把设置好的进程控制块放入hash_list和proc_list两个全局进程链表中。自此，进程已经准备好执行了，把进程状态设置为“就绪”态。设置返回码为子进程的id号。
如果上述前3步执行没有成功，则需要做对应的出错处理，把相关已经占有的内存释放掉。在第5步里，为了防止多线程不同步问题，需要暂时关中断。
```

练习2.1. 请说明ucore是否做到给每个新fork的线程一个唯一的id？请说明你的分析和理由。
```
是。get_pid函数负责分配PID。在这个函数里，循环查找了proc_list链表，一旦将要分配的PID与链表中某个进程的PID相同，就把将要分配的PID加一。函数保证区间[last_pid, next_safe)中没有已被分配的PID，一旦这个区间的长度缩短到0，就重新遍历proc_list链表，最终保证PID没有重复。
这个方法不是多线程安全的，所以get_pid()到list_add(&proc_list, &proc->list_link);的步骤之间是关中断的，来保证get_pid函数单线程运行，因此分配的PID是唯一的。
```

## 练习3
练习3.0. 请在实验报告中简要说明你对proc_run函数的分析
```
proc_run在关中断的情况下，进行如下操作
current = proc;
load_esp0(next->kstack + KSTACKSIZE);
lcr3(next->cr3);
switch_to(&(prev->context), &(next->context));
load_esp0和lcr3加载新进程的esp和cr3，切换内核栈、二级页表。switch_to切换进程上下文，包括寄存器、用户栈等，并把指令指针准备在栈顶iret读取返回值的位置。
```

练习3.1. 在本实验的执行过程中，创建且运行了几个内核线程？
```
2个
```

练习3.2. 语句local_intr_save(intr_flag);....local_intr_restore(intr_flag);在这里有何作用?请说明理由
暂时关闭中断：如果当前中断为开，则在local_intr_save关闭中断，待local_intr_restore再打开中断；如果当前中断为关，那么什么也不做。
目的是防止在进程切换的时候被中断打断，造成一些寄存器的值改变，导致恢复进程上下文或恢复页表时发生错误。
