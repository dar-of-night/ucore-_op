# Lab1 report

## [练习]

[练习1.1] 操作系统镜像文件 ucore.img 是如何一步一步生成的?(需要比较详细地解释 Makefile 中
每一条相关命令和命令参数的含义,以及说明命令导致的结果)

makefile文件的开头进行了条件定义，对可能用到的变量或者命令，根据需要进行说明或者替换。然后，包含了需要用到的头文件（也进行了定义，但方式有点不太明白）。而后，进入到makefile函数实体：
```
bin/ucore.img   /*镜像文件生成函数*/
|
|UCOREIMG	:= $(call totarget,ucore.img)
|$(UCOREIMG): $(kernel) $(bootblock)
|	$(V)dd if=/dev/zero of=$@ count=10000
|	$(V)dd if=$(bootblock) of=$@ conv=notrunc
|	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
|从这里可知，生成镜像需要调用kernel和bootblock函数
|
|>bin/bootblock
|   |bootblock函数相关代码为：
|   |$(bootblock): $(call toobj,$(bootfiles))   | $(call totarget,sign)
|   |@echo + ld $@
|   |$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
|	|@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
|	|@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
|	|@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
|   |为了生成bootblock，首先需要生成bootasm.o、bootmain.o、sign
|   |
|   |>obj/bootasm.o,bootmain.o
|   |   |生成上述两个文件的makefile代码是
|   |   |bootfiles = $(call listf_cc,boot)
|   |   |$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
|   |   |生成上述两个.o目标文件需要执行boot/bootasm.S，bootmain.c文件
|   |   |生成bootasm.o实际命令为
|   |   |gcc -Iboot/ -march=i686 -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
|	|	| 其中关键的参数为
|	|	| 	-ggdb  生成可供gdb使用的调试信息。这样才能用qemu+gdb来调试bootloader or ucore。
|	|	|	-m32  生成适用于32位环境的代码。我们用的模拟硬件是32bit的80386，所以ucore也要是32位的软件。
|	|	| 	-gstabs  生成stabs格式的调试信息。这样要ucore的monitor可以显示出便于开发者阅读的函数调用栈信息
|	|	| 	-nostdinc  不使用标准库。标准库是给应用程序用的，我们是编译ucore内核，OS内核是提供服务的，所以所有的服务要自给自足。
|	|	|	-fno-stack-protector  不生成用于检测缓冲区溢出的代码。这是for 应用程序的，我们是编译内核，ucore内核好像还用不到此功能。
|	|	| 	-Os  为减小代码大小而进行优化。根据硬件spec，主引导扇区只有512字节，我们写的简单bootloader的最终大小不能大于510字节。
|	|	| 	-I<dir>  添加搜索头文件的路径
|	|	| 
|   |   |生成bootmain.o实际命令为
|   |   |gcc -Iboot/ -march=i686 -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o
|   |>bin/sign
|   |   |生成sign的makefile函数为
|   |   |$(call add_files_host,tools/sign.c,sign,sign)
|   |   |$(call create_target_host,sign,sign)
|   |   |实际命令为
|   |   |gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
|   |   |gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
|   |生成bootblocker
|   |   |ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
 拷贝二进制代码bootblock.o到bootblock.out
|	| objcopy -S -O binary obj/bootblock.o obj/bootblock.out
|	| 其中关键的参数为
|	|	-S  移除所有符号和重定位信息
|	|	-O <bfdname>  指定输出格式
|	|
|	| 使用sign工具处理bootblock.out，生成bootblock
|	| bin/sign obj/bootblock.out bin/bootblock
|
|>bin/kernel
|   |$(kernel): tools/kernel.ld
|   |$(kernel): $(KOBJS)
|   |   @echo + ld $@
|   |   $(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
|   |   @$(OBJDUMP) -S $@ > $(call asmfile,kernel)
|   |   @$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
|   |从中可知道，生成kernel文件需要kernel.ld文件，这个文件在tool文件夹里。
|   |观察make执行文件内容，可知道.ld文件时用来生成kernel文件的，具体过程是将众多.o文件链接起来
|   |所需要的.o文件有obj/kern/init/init.o obj/kern/libs/stdio.o obj/kern/libs/readline.o obj/kern/debug/panic.o 
|   |obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/picirq.o obj/kern/driver/intr.o 
|   |obj/kern/trap/trap.o obj/kern/trap/vectors.o obj/kern/trap/trapentry.o obj/kern/mm/pmm.o  
|   |obj/libs/string.o obj/libs/printfmt.o
|   |生成这些.o文件的makefile代码是
|   |$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))
|   |下面以其中一个.o文件生成过程为例，其他文件过程类似
|   |>obj/kern/init/init.o
|   |以init.c来编译，实际命令
|   |gcc -Ikern/init/ -march=i686 -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/init/init.c -o obj/kern/init/init.o
|
| 生成一个有10000个块的文件，每个块默认512字节，用0填充
| dd if=/dev/zero of=bin/ucore.img count=10000
|
| 把bootblock中的内容写到第一个块，不截断
| dd if=bin/bootblock of=bin/ucore.img conv=notrunc
|
| 从第二个块开始写kernel中的内容，也同样不截断
| dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
```

[练习1.2] 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?
```
从sign.c的代码来看，主引导扇区的大小应该是512字节，其中511（倒数第一个）字节是0xAA;510(倒数第二个)字节是0x55。
```
## [练习2]

[练习2.1] 从 CPU 加电后执行的第一条指令开始,单步跟踪 BIOS 的执行。
练习2可以单步跟踪，方法如下：
 
1 修改 lab1/tools/gdbinit,内容为:
```
set architecture i8086
target remote :1234
```

2 在 lab1目录下，执行
```
make debug
```

3 在看到gdb的调试界面(gdb)后，在gdb调试界面下执行如下命令
```
si
```
即可单步跟踪BIOS了。

4 在gdb界面下，可通过如下命令来看BIOS的代码
```
 x /2i $pc  //显示当前eip处的汇编指令
```

[练习2.2] 在初始化位置0x7c00 设置实地址断点,测试断点正常。

在tools/gdbinit结尾加上
```
    set architecture i8086  //设置当前调试的CPU是8086
	b *0x7c00  //在0x7c00处设置断点。此地址是bootloader入口点地址，可看boot/bootasm.S的start地址处
	c          //continue简称，表示继续执行
	x /10i $pc  //显示当前eip处的汇编指令，10是显示行数
	set architecture i386  //设置当前调试的CPU是80386
```
	
运行"make debug"便可得到

```
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

[练习2.3] 在调用qemu 时增加-d in_asm -D q.log 参数，便可以将运行的汇编指令保存在q.log 中。
将执行的汇编代码与bootasm.S 和 bootblock.asm 进行比较，看看二者是否一致。
```
改写Makefile文件
	debug: $(UCOREIMG)
		$(V)$(TERMINAL) -e "$(QEMU) -S -s -d in_asm -D $(BINDIR)/q.log -parallel stdio -hda $< -serial null"
		$(V)sleep 2
		$(V)$(TERMINAL) -e "gdb -q -tui -x tools/gdbinit"
```
```
在调用qemu时增加`-d in_asm -D q.log`参数，便可以将运行的汇编指令保存在q.log中。
为防止qemu在gdb连接后立即开始执行，删除了`tools/gdbinit`中的`continue`行。
```
得到代码如下，比对后一致：
```
----------------
IN: 
0xfffffff0:  ljmp   $0xf000,$0xe05b

----------------
IN: 
0x000fe05b:  cmpl   $0x0,%cs:0x70c8
0x000fe062:  jne    0xfd414

----------------
IN: 
0x000fe066:  xor    %dx,%dx
0x000fe068:  mov    %dx,%ss

----------------
IN: 
0x000fe06a:  mov    $0x7000,%esp

----------------
IN: 
0x000fe070:  mov    $0xf2d4e,%edx
0x000fe076:  jmp    0xfff00

----------------
IN: 
0x000fff00:  cli    
0x000fff01:  cld    
0x000fff02:  mov    %eax,%ecx
0x000fff05:  mov    $0x8f,%eax
0x000fff0b:  out    %al,$0x70
0x000fff0d:  in     $0x71,%al
0x000fff0f:  in     $0x92,%al
0x000fff11:  or     $0x2,%al
0x000fff13:  out    %al,$0x92
0x000fff15:  mov    %ecx,%eax
0x000fff18:  lidtw  %cs:0x70b8
0x000fff1e:  lgdtw  %cs:0x7078
0x000fff24:  mov    %cr0,%ecx
0x000fff27:  and    $0x1fffffff,%ecx
0x000fff2e:  or     $0x1,%ecx
0x000fff32:  mov    %ecx,%cr0
。。。。。。。省略
```

[练习2.4]设置断点查看练习


## [练习3]
分析bootloader 进入保护模式的过程。

加载到磁盘后，运行实模式从%cs=0 %ip=7c00开始。

标志和寄存器置零
```
start:
.code16                                        
    cli                                            
    cld                                             

    # Set up the important data segment registers (DS, ES, SS).
    xorw %ax, %ax                                   
    movw %ax, %ds                                   
    movw %ax, %es                                   
    movw %ax, %ss                                   
```


开启A20：通过将键盘控制器上的A20线置于高电位，全部32条地址线可用，
可以访问4G的内存空间。
```
	seta20.1:               # 等待8042键盘控制器不忙
	    inb $0x64, %al      # 
	    testb $0x2, %al     #
	    jnz seta20.1        #
	
	    movb $0xd1, %al     # 发送写8042输出端口的指令
	    outb %al, $0x64     #
	
	seta20.1:               # 等待8042键盘控制器不忙
	    inb $0x64, %al      # 
	    testb $0x2, %al     #
	    jnz seta20.1        #
	
	    movb $0xdf, %al     # 打开A20
	    outb %al, $0x60     # 
```
初始化GDT表：一个简单的GDT表和其描述符已经静态储存在引导区中，载入即可
```
	    lgdt gdtdesc
```

进入保护模式：通过将cr0寄存器PE位置1便开启了保护模式
```
	    movl %cr0, %eax
	    orl $CR0_PE_ON, %eax
	    movl %eax, %cr0
```

通过长跳转更新cs的基地址
```
	 ljmp $PROT_MODE_CSEG, $protcseg
	.code32
	protcseg:
```

设置段寄存器，并建立堆栈
```
	    movw $PROT_MODE_DSEG, %ax
	    movw %ax, %ds
	    movw %ax, %es
	    movw %ax, %fs
	    movw %ax, %gs
	    movw %ax, %ss
	    movl $0x0, %ebp
	    movl $start, %esp
```
转到保护模式完成，进入boot主程序
```
	    call bootmain
```

## [练习4]
分析bootloader加载ELF格式的OS的过程。

readsect函数：
`readsect`从设备的第secno扇区读取数据到dst位置
```
readsect(void *dst, uint32_t secno) {
    // 等待磁盘准备好
    waitdisk();

    outb(0x1F2, 1);                        // 读取数目为1
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
	// 上面四条指令联合制定了扇区号
	        // 在这4个字节线联合构成的32位参数中
	        //   29-31位强制设为1
	        //   28位(=0)表示访问"Disk 0"
	        //   0-27位是28位的偏移量
    outb(0x1F7, 0x20);              // 0x20命令，读取扇区

    // wait for disk to be ready
    waitdisk();

    // read a sector
    insl(0x1F0, dst, SECTSIZE / 4);
}
```
readseg函数：
简单包装了readsect，可以从设备读取任意长度的内容。
```
readseg(uintptr_t va, uint32_t count, uint32_t offset) {
    uintptr_t end_va = va + count;

    //标定段边界位置
    va -= offset % SECTSIZE;

    // 数码到段的翻译；kernel从1开始
    uint32_t secno = (offset / SECTSIZE) + 1;

    //为了快速写入ELF文件，可以超前加载
    for (; va < end_va; va += SECTSIZE, secno ++) {
        readsect((void *)va, secno);
    }
}
```
bootmain函数：
进入bootloader
```
void
	bootmain(void) {
	    // 首先读取ELF的头部
	    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
	
	    // 通过储存在头部的幻数判断是否是合法的ELF文件
	    if (ELFHDR->e_magic != ELF_MAGIC) {
	        goto bad;
	    }
	
	    struct proghdr *ph, *eph;
	
	    // ELF头部有描述ELF文件应加载到内存什么位置的描述表，
	    // 先将描述表的头地址存在ph
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

## [练习5] 
实现函数调用堆栈跟踪函数 

我们需要在lab1中完成kdebug.c中函数print_stackframe的实现，可以通过函数print_stackframe来跟踪函数调用堆栈中记录的返回地址。
```
     uint32_t ebp=read_ebp(),eip=read_eip();

	  int i,j;
	  for(i=0;ebp!=0&&i<STACKFRAME_DEPTH;i++)
	  	{
	  	cprintf("ebp=0x%08x; eip=0x%08x args:",ebp,eip);
	  	uint32_t *args=(uint32_t *)ebp+2;
		for(j=0;j<4;j++)
			{
			cprintf(" 0x%08x",args[j]);
			}
		cprintf("\n");
		print_debuginfo(eip-1);
		eip=((uint32_t *)ebp)[1];
		ebp=((uint32_t *)ebp)[0];
	  	}
```

在如果能够正确实现此函数，可在lab1中执行 “make qemu”后，在qemu模拟器中得到类似如下的输出：
```
Kernel executable memory footprint: 64KB
ebp=0x00007b28; eip=0x00100a63 args: 0x00010094 0x00010094 0x00007b58 0x00100092
    kern/debug/kdebug.c:305: print_stackframe+21
ebp=0x00007b38; eip=0x00100d4d args: 0x00000000 0x00000000 0x00000000 0x00007ba8
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp=0x00007b58; eip=0x00100092 args: 0x00000000 0x00007b80 0xffff0000 0x00007b84
    kern/init/init.c:48: grade_backtrace2+33
ebp=0x00007b78; eip=0x001000bc args: 0x00000000 0xffff0000 0x00007ba4 0x00000029
    kern/init/init.c:53: grade_backtrace1+38
ebp=0x00007b98; eip=0x001000db args: 0x00000000 0x00100000 0xffff0000 0x0000001d
    kern/init/init.c:58: grade_backtrace0+23
ebp=0x00007bb8; eip=0x00100101 args: 0x001032dc 0x001032c0 0x0000130a 0x00000000
    kern/init/init.c:63: grade_backtrace+34
ebp=0x00007be8; eip=0x00100055 args: 0x00000000 0x00000000 0x00000000 0x00007c4f
    kern/init/init.c:28: kern_init+84
ebp=0x00007bf8; eip=0x00007d72 args: 0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8
    <unknow>: -- 0x00007d71 --
++ setup timer interrupts
```

解释最后一行各个数值的含义。
```
ebp=0x00007bf8; eip=0x00007d72 args: 0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8
```
其对应的是第一个使用堆栈的函数，bootmain.c中的bootmain。
bootloader设置的堆栈从0x7c00开始，使用"call bootmain"转入bootmain函数。
call指令压栈，所以bootmain中ebp为0x7bf8。

## [练习6]
完善中断初始化和处理

[练习6.1] 中断向量表中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

中断向量表一个表项占用8字节，其中2-3字节是段选择子，0-1字节和6-7字节拼成位移，
两者联合便是中断处理程序的入口地址。4-5字节是系统权限和置零设置。

[练习6.2] 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。

见代码

[练习6.3] 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数

见代码


## [练习7]

增加syscall功能，即增加一用户态函数（可执行一特定系统调用：获得时钟计数值），
当内核初始完毕后，可从内核态返回到用户态的函数，而用户态的函数又通过系统调用得到内核态的服务

见代码


