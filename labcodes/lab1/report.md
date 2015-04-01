# Lab1 report

## 练习1

练习1.1. 操作系统镜像文件 ucore.img 是如何一步一步生成的?(需要比较详细地解释 Makefile 中每一条相关命令和命令参数的含义,以及说明命令导致的结果)

```
在Makefile中,生成ucore.img的代码:
$(UCOREIMG): $(kernel) $(bootblock)
$(V)dd if=/dev/zero of=$@ count=10000
$(V)dd if=$(bootblock) of=$@ conv=notrunc
$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

ucore.img的生成依赖于bootblock、kernel,其中
生成bootblock的代码:
$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
@echo + ld $@
$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ \
	-o $(call toobj,bootblock)
@$(OBJDUMP) -S $(call objfile,bootblock) > \
	$(call asmfile,bootblock)
@$(OBJCOPY) -S -O binary $(call objfile,bootblock) \
	$(call outfile,bootblock)
@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

bootblock的生成依赖于bootasm.o、bootmain.o、sign

生成bootasm.o,bootmain.o的代码:
bootfiles = $(call listf_cc,boot) 
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),\
$(CFLAGS) -Os -nostdinc))

bootasm.o依赖于bootasm.S：
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs \
	-nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc \
	-c boot/bootasm.S -o obj/boot/bootasm.o

bootmain.o依赖于bootmain.c：
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc \
	-fno-stack-protector -Ilibs/ -Os -nostdinc \
	-c boot/bootmain.c -o obj/boot/bootmain.o

生成sign的代码为：
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)

生成bootblock.o：
ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 \
obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o

生成kernel的代码：
$(kernel): tools/kernel.ld
$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; \
		/^$$/d' > $(call symfile,kernel)

kernel依赖于kernel.ld init.o readline.o stdio.o kdebug.o kmonitor.o panic.o clock.o console.o intr.o picirq.o trap.o trapentry.o vectors.o pmm.o  printfmt.o string.o
其中kernel.ld已存在
生成.o文件的代码为：
$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,\
$(KCFLAGS))

生成有10000个块的文件，每个块默认512字节：
dd if=/dev/zero of=bin/ucore.img count=10000

把bootblock中的内容写到第一个块：
dd if=bin/bootblock of=bin/ucore.img conv=notrunc

从第二个块开始写kernel中的内容：
dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
```

练习1.2. 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?
```
一个磁盘主引导扇区只有512字节。且第510个字节是0x55，第511个字节是0xAA。
```


## 练习2

练习2.1. 从 CPU 加电后执行的第一条指令开始,单步跟踪 BIOS 的执行。

单步跟踪的方法为：
```
1. 修改 lab1/tools/gdbinit：
set architecture i8086
target remote :1234

2. 在 lab1目录下，执行
make debug

3. 在gdb调试界面下执行
si
即可单步跟踪。
```

练习2.2. 在初始化位置0x7c00 设置实地址断点,测试断点正常。
```
在tools/gdbinit结尾加上
    set architecture i8086  //设置当前调试的CPU是8086
	b *0x7c00  //在0x7c00处设置断点。此地址是bootloader入口点地址，可看boot/bootasm.S的start地址处
	c          //continue简称，表示继续执行
	x /2i $pc  //显示当前eip处的汇编指令
	set architecture i386  //设置当前调试的CPU是80386
运行"make debug"便可得到
	Breakpoint 2, 0x00007c00 in ?? ()
	=> 0x7c00:      cli    
	   0x7c01:      cld    
	   0x7c02:      xor    %eax,%eax
	   0x7c04:      mov    %eax,%ds
	   0x7c06:      mov    %eax,%es
	   0x7c08:      mov    %eax,%ss 
	   0x7c0a:      in     $0x64,%al
	   0x7c0c:      test   $0x2,%al
	   0x7c0e:      jne    0x7c0a
	   0x7c10:      mov    $0xd1,%al
```

练习2.3. 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。
```
与bootasm.S和bootblock.asm中的代码相同。
```

## 练习3
分析bootloader 进入保护模式的过程。
```
从`%cs=0 $pc=0x7c00`，进入后首先清理环境：将flag置0和将段寄存器置0

开启A20：通过将键盘控制器上的A20线置于高电位，全部32条地址线可用，可以访问4G的内存空间。

初始化GDT表：一个简单的GDT表和其描述符已经静态储存在引导区中，载入即可

进入保护模式：通过将cr0寄存器PE位置1便开启了保护模式

通过长跳转更新cs的基地址

设置段寄存器，并建立堆栈

转到保护模式完成，进入boot主方法
```


## 练习4
分析bootloader加载ELF格式的OS的过程。
```
readsect函数从设备的第secno扇区读取数据到dst位置
readseg函数简单包装了readsect，可以从设备读取任意长度的内容。

然后，在bootmain函数中：
	void
	bootmain(void) {
	    // 读取ELF的头部
	    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
	
	    // 通过储存在头部的幻数判断是否是合法的ELF文件
	    if (ELFHDR->e_magic != ELF_MAGIC) {
	        goto bad;
	    }
	
	    struct proghdr *ph, *eph;
	
	    // ELF头部有描述ELF文件应加载到内存什么位置的描述表，先将描述表的头地址存在ph
	    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
	    eph = ph + ELFHDR->e_phnum;
	
	    // 按照描述表将ELF文件中数据载入内存
	    for (; ph < eph; ph ++) {
	        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
	    }
	    // ELF文件0x1000位置后面的0xd1ec比特被载入内存0x00100000
	    // ELF文件0xf000位置后面的0x1d20比特被载入内存0x0010e000

	    // 根据ELF头部储存的入口信息，找到内核的入口
	    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
	
	bad:
	    outw(0x8A00, 0x8A00);
	    outw(0x8A00, 0x8E00);
	    while (1);
	}
```


## 练习5
实现函数调用堆栈跟踪函数 
```
ss:ebp指向的堆栈位置储存着caller的ebp，以此为线索可以得到所有使用堆栈的函数ebp。
ss:ebp+4指向caller调用时的eip，ss:ebp+8等是（可能的）参数。

输出中，堆栈最深一层为
	ebp:0x00007bf8 eip:0x00007d68 \
		args:0x00000000 0x00000000 0x00000000 0x00007c4f
	    <unknow>: -- 0x00007d67 --

其对应的是第一个使用堆栈的函数，bootmain.c中的bootmain。
bootloader设置的堆栈从0x7c00开始，使用"call bootmain"转入bootmain函数。
call指令压栈，所以bootmain中ebp为0x7bf8。
```


## 练习6
完善中断初始化和处理

练习6.1. 中断向量表中一个表项占多少字节？其中哪几位代表中断处理代码的入口？
```
中断向量表一个表项占用8字节，其中2-3字节是段选择子，0-1字节和6-7字节拼成位移，
两者联合便是中断处理程序的入口地址。
```
