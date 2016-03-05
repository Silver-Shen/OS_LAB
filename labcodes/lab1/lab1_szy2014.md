#OS Lab1 Report

##练习一
####1.操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)
>分析过程分为两步，首先根据Makefile，从最后一步开始反向追溯ucore.img生成的过程，其次通过make V=显示编译过程的细节，正向验证可执行文件的产生过程<br>
（1）直接生成ucore.img的部分为如下命令段

```
UCOREIMG	:= $(call totarget,ucore.img)

$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)
```
>ucore硬盘映像总共有10000个扇区，bootblock放在第一块，kernel从第二块开始放（文件指针seek=1）。在ucore.img生成的过程中有两个依赖，一个是kernel、另一个是bootblock，这两个也都是可执行文件，下面讨论这两个文件的生成。<br>
（2）生成bootblock的命令段如下

```
bootblock = $(call totarget,bootblock)

$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

$(call create_target,bootblock)
```
>bootblock的生成依赖于bootfiles的目标文件(.o文件)以及sign的可执行文件，bootfiles是boot文件夹下的文件（bootasm.S和bootmain.c）

```
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
```
>生成.o文件是通过脚本语句foreach批量生成，编译命令由宏CC定义，编译选项由CFLAGS定义，还有一些诸如-Os的编译优化选项，f遍历bootfiles中的代码文件生成两个目标文件bootasm.o和bootmain.o，生成sign的代码入下

```
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)
```
>kernel的生成来自如下命令

```
kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)
```
>生成kernel依赖的文件为kernel.ld和KOBJS，其中KOBJS在之前有过宏定义，含义为kern文件夹下所有.c文件生成的目标文件。综上，完成了对ucore.img生成过程的回溯。<br>
`下面通过make的编译过程验证一下，在lab1中进行编译`

```
    make V=
```
>首先是kernel的部分(只列出一部分)

```
+ cc kern/libs/stdio.c
gcc -Ikern/libs/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/libs/stdio.c -o obj/kern/libs/stdio.o
+ cc kern/libs/readline.c
gcc -Ikern/libs/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/libs/readline.c -o obj/kern/libs/readline.o
+ cc kern/debug/panic.c
gcc -Ikern/debug/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/debug/panic.c -o obj/kern/debug/panic.o
...
...
...
+ ld bin/kernel
ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel  obj/kern/init/init.o obj/kern/libs/stdio.o obj/kern/libs/readline.o obj/kern/debug/panic.o obj/kern/debug/kdebug.o obj/kern/debug/kmonitor.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/picirq.o obj/kern/driver/intr.o obj/kern/trap/trap.o obj/kern/trap/vectors.o obj/kern/trap/trapentry.o obj/kern/mm/pmm.o  obj/libs/string.o obj/libs/printfmt.o
```
>然后是bootblock部分

```
+ cc boot/bootasm.S
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
+ cc boot/bootmain.c
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o
+ cc tools/sign.c
gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
+ ld bin/bootblock
ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
```
>最后是生成ucore.img，可以看出bootloader大小是512字节一个扇区

```
dd if=/dev/zero of=bin/ucore.img count=10000
10000+0 records in
10000+0 records out
5120000 bytes (5.1 MB) copied, 0.0218658 s, 234 MB/s
dd if=bin/bootblock of=bin/ucore.img conv=notrunc
1+0 records in
1+0 records out
512 bytes (512 B) copied, 0.000214607 s, 2.4 MB/s
dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
146+1 records in
146+1 records out
74864 bytes (75 kB) copied, 0.000527022 s, 142 MB/s

```

####2.一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？
由学堂在线第三讲 3.1 BIOS小节可知，一个主引导扇区应该以0x55AA作为结尾<br>
阅读tools/sign.c，也可以得到相同的结论(buf[510]=0x55,buf[511]=0xAA)

```
char buf[512];
    memset(buf, 0, sizeof(buf));
    FILE *ifp = fopen(argv[1], "rb");
    int size = fread(buf, 1, st.st_size, ifp);
    if (size != st.st_size) {
        fprintf(stderr, "read '%s' error, size is %d.\n", argv[1], size);
        return -1;
    }
    fclose(ifp);
    buf[510] = 0x55;
    buf[511] = 0xAA;
    FILE *ofp = fopen(argv[2], "wb+");
    size = fwrite(buf, 1, 512, ofp);
```

##练习二
####1.从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行
参考实验指导书附录“启动后第一条执行的指令”，步骤如下<br>
1 修改 lab1/tools/gdbinit, 内容为:

```
set architecture i8086
target remote :1234
```
2 修改makefile中debug的宏，增加日志记录的功能

```
debug: $(UCOREIMG)
		$(V)$(TERMINAL) -e "$(QEMU) -S -s -d in_asm -D $(BINDIR)/q.log -parallel stdio -hda $< -serial null"
		$(V)sleep 2
		$(V)$(TERMINAL) -e "gdb -q -tui -x tools/gdbinit"
```
运行过的汇编指令将被记录在q.log中

3 在lab1的目录下，执行

```
make debug
```
4 在gdb下可以查看BIOS的最初的指令

```
(gdb) x /2i 0xffff0
   0xffff0:     ljmp   $0xf000,$0xe05b
   0xffff5:     xor    %dh,0x322f

``` 
可以看到最初的一条指令是长跳转，跳转到0xfe05b去继续执行，查看此时的寄存器

```
(gdb) info register cs
cs             0xf000   61440
(gdb) info register eip
eip            0xfff0   0xfff0
```
和原理描述的是相符的

5 执行si可以继续单步调试，观察BIOS执行的过程

####2.在初始化位置0x7c00设置实地址断点,测试断点正常 
参考lab1init, 修改tools/gdbinit

```
target remote :1234
set architecture i8086  
b *0x7c00  
c
x /10i $pc
set architecture i386

```
0x7c00处设置断点，并在执行到此处（bootloader起始点）时切换回32位模式显示

之后运行 make debug可以得到

```
The target architecture is assumed to be i8086
Breakpoint 1 at 0x7c00

Breakpoint 1, 0x00007c00 in ?? ()
=> 0x7c00:      cli
   0x7c01:      cld
   0x7c02:      xor    %ax,%ax
   0x7c04:      mov    %ax,%ds
   0x7c06:      mov    %ax,%es
   0x7c08:      mov    %ax,%ss
   0x7c0a:      in     $0x64,%al
   0x7c0c:      test   $0x2,%al
   0x7c0e:      jne    0x7c0a
   0x7c10:      mov    $0xd1,%al
The target architecture is assumed to be i386
```
这就完成了断点的测试

####3.从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较
在练习2.1中，我们修改了debug的宏，加上了日志记录的功能，故运行make debug之后，我们可以在q.log文件中看到我们反汇编出来的代码，以0x7c00开始的代码为例

```
IN: 
0x00007c00:  cli    

----------------
IN: 
0x00007c01:  cld    

----------------
IN: 
0x00007c02:  xor    %ax,%ax

----------------
IN: 
0x00007c04:  mov    %ax,%ds

----------------
IN: 
0x00007c06:  mov    %ax,%es

----------------
IN: 
0x00007c08:  mov    %ax,%ss

----------------
IN: 
0x00007c0a:  in     $0x64,%al

----------------
IN: 
0x00007c0c:  test   $0x2,%al

----------------
IN: 
0x00007c0e:  jne    0x7c0a

----------------
IN: 
0x00007c10:  mov    $0xd1,%al

----------------
IN: 
0x00007c12:  out    %al,$0x64

----------------
IN: 
0x00007c14:  in     $0x64,%al

----------------
IN: 
0x00007c16:  test   $0x2,%al

----------------
IN: 
0x00007c18:  jne    0x7c14

----------------
IN: 
0x00007c1a:  mov    $0xdf,%al

----------------
IN: 
0x00007c1c:  out    %al,$0x60

----------------
IN: 
0x00007c1e:  lgdtw  0x7c6c

----------------
IN: 
0x00007c23:  mov    %cr0,%eax

----------------
IN: 
0x00007c26:  or     $0x1,%eax

----------------
IN: 
0x00007c2a:  mov    %eax,%cr0

----------------
IN: 
0x00007c2d:  ljmp   $0x8,$0x7c32

----------------
IN: 
0x00007c32:  mov    $0x10,%ax

----------------
IN: 
0x00007c36:  mov    %eax,%ds

----------------
IN: 
0x00007c38:  mov    %eax,%es

----------------
IN: 
0x00007c3a:  mov    %eax,%fs

----------------
IN: 
0x00007c3c:  mov    %eax,%gs

----------------
IN: 
0x00007c3e:  mov    %eax,%ss

----------------
IN: 
0x00007c40:  mov    $0x0,%ebp

----------------
IN: 
0x00007c45:  mov    $0x7c00,%esp

----------------
IN: 
0x00007c4a:  call   0x7d0d
```
最后call的是bootmain函数，和bootasm.S和bootblock.asm中的代码是一致的。

##练习三
####分析bootloader进入保护模式的过程
主要方法为阅读启动代码bootasm.S
####1.开启A20
>代码首先定义了保护模式下代码段和数据段寄存器的值以及保护模式下CR0寄存器的flag值，用于切换时的修改

```
.set PROT_MODE_CSEG,        0x8                     
.set PROT_MODE_DSEG,        0x10                   
.set CR0_PE_ON,             0x1                    
```
>然后bootAsm重置了一些段寄存器的值，进入开启A20模块；A20位于8042键盘控制器的一个状态位上，故需要通过和键盘控制器的交互来开启A20；这里CPU和键盘是通过轮询（PIO）的方式进行交互（由于还没有中断处理机制，虽然效率很低，但编程较为容易），CPU反复查看键盘状态字，判断是否忙，循环等待直到缓冲区空了，将修改A20后的状态字写入，完成了开启A20的工作

```
seta20.1:
    inb $0x64, %al                                  
    testb $0x2, %al
    jnz seta20.1  #判断是否忙
    movb $0xd1, %al                                 
    outb %al, $0x64                                 

seta20.2:
    inb $0x64, %al                                 
    testb $0x2, %al
    jnz seta20.2
    movb $0xdf, %al   #A20置位                              
    outb %al, $0x60
```
####2.初始化DGT表
>DGT表项以及GDTR（dgtdesc）已经由bootasm初始化（三个表项）

```
.p2align 2             # force 4 byte alignment
gdt:
    SEG_NULLASM        # null seg
    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)           
    # code seg for bootloader and kernel
    SEG_ASM(STA_W, 0x0, 0xffffffff)                 
    # data seg for bootloader and kernel

gdtdesc:
    .word 0x17          # sizeof(gdt) - 1
    .long gdt           # address gdt
```
通过ldgt特权指令就可以导入全局描述符表的选择子

```
lgdt gdtdesc
```

####3.进入保护模式
>进入保护模式首先需要cr0寄存器的置位

```
	movl %cr0, %eax
    orl $CR0_PE_ON, %eax #开启保护位
    movl %eax, %cr0
```
>然后需要初始化在32位模式的下各个段寄存器

```
.code32                  # Assemble for 32-bit mode
protcseg:
    # Set up the protected-mode data segment registers
    movw $PROT_MODE_DSEG, %ax                           	movw %ax, %ds                                       	movw %ax, %es                                       
    movw %ax, %fs                                   
    movw %ax, %gs                                   
    movw %ax, %ss                                   
```
>然后准备栈帧，调用主函数

```
    # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
    movl $0x0, %ebp
    movl $start, %esp
    call bootmain
```

##练习四
####分析bootloader加载ELF格式的OS的过程
>主要方法为阅读bootloader的主函数所在处bootmain.c文件，elf格式的文件定义在libs/elf.h中
####1.bootloader如何读取硬盘扇区的
>bootmain中有两个函数用于读取硬盘扇区readsect和readseg，还有一个查看硬盘状态的辅助函数waitdisk，调用关系为bootmain->readseg->readsect<br>
readsect函数为底层的读取硬盘的函数，通过IO地址寄存器与硬盘进行交互

```
IO地址	功能
0x1f0	读数据，当0x1f7不为忙状态时，可以读。
0x1f2	要读写的扇区数，每次读写前，你需要表明你要读写几个扇区。最小是1个扇区
0x1f3	如果是LBA模式，就是LBA参数的0-7位
0x1f4	如果是LBA模式，就是LBA参数的8-15位
0x1f5	如果是LBA模式，就是LBA参数的16-23位
0x1f6	第0~3位：如果是LBA模式就是24-27位 第4位：为0主盘；为1从盘
0x1f7	状态和命令寄存器。操作时先给命令，再读取，如果不是忙状态就从0x1f0   端口读数据
```
>通过操作IO寄存器0x1F0~0x1F7设定读取扇区的数量和扇区号，并读取到指定的内存地址

```
	outb(0x1F2, 1);                        
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);                      
    waitdisk();
    insl(0x1F0, dst, SECTSIZE / 4);
```
readseg简单地做一些预处理，然后通过调用readsect进行多个扇区的读取（kernel可能会持续多个扇区，而不像bootloader一样仅有一个扇区那么大）
####2.bootloader是如何加载ELF格式的OS
>主要阅读bootmain函数，函数首先读入了elf的第一页（8个扇区），即elf文件的header部分，这个header里包含了文件的许多信息(具体的定义参见libs/elf.h)，其中用来判断文件是否合法字段是e_magic（幻数），bootloader在一开始首先根据幻数来判断是否合法，如果不合法则输出提示并陷入死循环

```
	readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    // is this a valid ELF?
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }
```   
>如果elf文件合法，那么进而读取其中所含的每一个程序段

```
	struct proghdr *ph, *eph;

    // load each program segment (ignores ph flags)
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }
```      
>最后跳转到（调用）OS内核中的主函数，将控制权交给OS kernel

```
// call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
```

##练习五
####实现函数调用堆栈跟踪函数 
>主要实现思路为，首先获取当前的ebp和eip，因为调用栈都是递归嵌套的，故可以通过当前的ebp来追溯之前的调用函数。阅读“函数堆栈章节”之后，可以了解到函数栈的基本构成：ss:[ebp]=old ebp; ss:[ebp+1]=RA; ss:[ebp+N]=args(N>1),故可以逐层打印出函数的信息。完成print_stackframe函数后，qemu模拟器中的显示结果如下

```
ebp:0x00007b38 eip:0x00100a27 args:0x00010094 0x00010094 0x00007b68 0x0010007f 
    kern/debug/kdebug.c:307: print_stackframe+21
ebp:0x00007b48 eip:0x00100d21 args:0x00000000 0x00000000 0x00000000 0x00007bb8 
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp:0x00007b68 eip:0x0010007f args:0x00000000 0x00007b90 0xffff0000 0x00007b94 
    kern/init/init.c:48: grade_backtrace2+19
ebp:0x00007b88 eip:0x001000a1 args:0x00000000 0xffff0000 0x00007bb4 0x00000029 
    kern/init/init.c:53: grade_backtrace1+27
ebp:0x00007ba8 eip:0x001000be args:0x00000000 0x00100000 0xffff0000 0x00100043 
    kern/init/init.c:58: grade_backtrace0+19
ebp:0x00007bc8 eip:0x001000df args:0x00000000 0x00000000 0x00000000 0x00103260 
    kern/init/init.c:63: grade_backtrace+26
ebp:0x00007be8 eip:0x00100050 args:0x00000000 0x00000000 0x00000000 0x00007c4f 
    kern/init/init.c:28: kern_init+79
ebp:0x00007bf8 eip:0x00007d6e args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8 
    <unknow>: -- 0x00007d6d --

```
>和实验指导书大致一致，最后一行是第一个被调用的函数（也即第一个使用栈空间的函数），bootloader为函数分配了0x0~0x7bff的栈空间，第一条指令从0x7c00开始存放，bootasm中调用call bootmain首次使用栈空间，call指令执行两个操作，首先把返回地址压栈，其次把原始ebp（即0x0）压栈，故bootmain函数的ebp为0x7bf8，eip为bootmain第一条指令所处的地址。

##练习六
####完善中断初始化和处理
####1.中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？
>中断描述符表一个表项占8个字节，其中第0~1字节以及第6~7字节联合构成了32位的段内偏移；第2~3位是段选择子，这两个信息结合段机制可以给出中断处理代码的入口

####2.请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。在idt_init函数中，依次对所有中断入口进行初始化。使用mmu.h中的SETGATE宏，填充idt数组内容。每个中断的入口由tools/vectors.c生成，使用trap.c中声明的vectors数组即可
>大致思路为调用mmu.h中声明的宏SETGATE来初始化中断描述附表中的每一个表项，遍历整个idt数组完成初始化，偏移量通过__vectors获得，选择子为内核代码段；值得注意的是，T_SYSCALL中断因为可以由用户程序调用，故特权级需要修改。

####3.请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字”100 ticks”
>这部分比较简单，clock.c中有一个全局变量ticks用来记录每次时钟中断，在中断处理函数中累加这个计数器，每到100就打印提示即可。