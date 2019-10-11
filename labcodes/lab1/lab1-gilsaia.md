# 练习1

## 练习1.1

通过`make "V="`命令，我们可以观察到make实际执行了哪些命令。

仔细观察其实可以看出来，加的后缀实际上只是把原来V的默认赋值“@”去掉...把一部分命令显示出来了...

首先可以看到的是一系列的编译`kern`目录下的文件到目标文件的命令，仔细观察后发现相关代码段如下
```
# kernel

KINCLUDE	+= kern/debug/ \
			   kern/driver/ \
			   kern/trap/ \
			   kern/mm/

KSRCDIR		+= kern/init \
			   kern/libs \
			   kern/debug \
			   kern/driver \
			   kern/trap \
			   kern/mm

KCFLAGS		+= $(addprefix -I,$(KINCLUDE))

$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))

KOBJS	= $(call read_packet,kernel libs)

# create kernel target
kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)
```
其中`add_files_cc`是在上方定义的函数，并调用了在`function.mk`中定义的部分，会对给定目录下的文件进行编译，并输出类似`+ cc dir/dir`类似的信息。并进行gcc的编译，其中gcc的命令参数在`KCFLAGS`中被定义，其中
| 参数 | 含义 |
|---|---|
|-march=i686|生成适合对应cpu平台的代码|
|-fno-builtin|不接受不是两个下划线开头的内建函数|
|-fno-PIC|-fPIC的相反形式，不生成与位置无关的代码|
|-Wall|对常见错误发出警告|
|-ggdb|以本地格式输出调试信息，尽可能包括GDB扩展|
|-m32|生成32位汇编代码|
|-gstabs|以stabs格式输出调试信息，不包括GDB扩展|
|-nostdinc| 不在标准系统目录中寻找头文件，而只搜索-I制定目录|
|-I *dir*|将*dir*目录添加到头文件搜索路径|
|-fno-stack-protector|禁用栈保护措施|
|-c|执行至生成目标文件阶段|
|-o *filename*|输出文件名|
从而对`KSRCDIR`定义的目录下文件进行编译，中间会有部分警告包括定义但未使用的函数，可能错误的命名
等等，而后执行链接操作，`-T`代表使用指定脚本，`-m`表示指定平台，这样将之前生成的目标文件链接生成*bin/kernal*文件

而后生成的是bootblock块，相关代码与kernal块类似
```
# create bootblock
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

bootblock = $(call totarget,bootblock)

$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

$(call create_target,bootblock)
```
先对boot目录下文件生成目标文件，这里多出的后缀
`-Os`是为了优化文件大小的设置，还生成了`sign`的文件，用于生成最后的bootblock，其中`-N`为设置代码段和数据段均可读写，`-e`设置入口，**`-Ttext`设置代码段开始位置(加粗因为问的时候忘了=_=)**

最后执行的是生成ucore.img相关的dd命令
```
# create ucore.img
UCOREIMG	:= $(call totarget,ucore.img)

$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)
```
`dd if=/dev/zero of=bin/ucore.img count=10000`生成一万个块的空文件ucore.img
`dd if=bin/bootblock of=bin/ucore.img conv=notrunc`将之前生成的bootblock块不截短写入文件
`dd if=bin/bootblock of=bin/ucore.img conv=notrunc`将生成的kernal内核跳过一个块放入ucore.img，即跳过bootblock块

这样最后的ucore.img文件就生成了

## 练习1.2

经过阅读相关资料后得知，BIOS在进行完硬件自检以及初始化后，会将启动设备的第一扇区也就是主引导扇区放到内存的0x7c00处，在ucore中也就是将我们之前生成的bootblock块拷贝入内存，而之前生成bootblock块的命令也可以证实这一点，而显示的信息以及makefile的命令都指出链接生成bootblock后还经过了sign程序，从sign的报错提示来看bootblock块也就是主引导扇区大小应该不能超过512字节，而最后两个字节应已规定好不能写入其他，倒数第二个字节是0x55,倒数第一个字节是0xAA

找到地方之后发现写的确实挺明显的...然而找到这个文件可是够费劲的...

# 练习2

## 练习2.1

根据网站上的参考资料，我们在tools下建立gdbinit_2
```
set architecture i8086
target remote :1234
```
并修改makefile中debug指令gdb调试指定脚本，从而从BIOS开始单步调试
执行
```
x /i 0xffff0
```
即可看到执行的第一条长跳转指令
```
ljmp $0x3630,$0xf000e05b
```
`si`可以让cpu单步执行从而观察BIOS指令

## 练习2.2

经查阅后得知gdb添加断点即在脚本加上
```
b *0x7c00
```
之后让其继续运行即可
而后就可观察到
```
(gdb)Breakpoint 1, 0x00007c00 in ?? ()
(qemu)Booting from Hard Disk
```

## 练习2.3

在上一问基础上，当断点停下时观察附近的指令，如下
```
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
   0x7c12:      out    %al,$0x64
   0x7c14:      in     $0x64,%al
   0x7c16:      test   $0x2,%al
   0x7c18:      jne    0x7c14
   0x7c1a:      mov    $0xdf,%al
   0x7c1c:      out    %al,$0x60
   0x7c1e:      lgdtl  (%esi)
   0x7c21:      insb   (%dx),%es:(%edi)
   0x7c22:      jl     0x7c33
   0x7c24:      and    %al,%al
   0x7c26:      or     $0x1,%ax
   0x7c2a:      mov    %eax,%cr0
   0x7c2d:      ljmp   $0xb866,$0x87c32
   0x7c34:      adc    %al,(%eax)
   0x7c36:      mov    %eax,%ds
   0x7c38:      mov    %eax,%es
   0x7c3a:      mov    %eax,%fs
   0x7c3c:      mov    %eax,%gs
   0x7c3e:      mov    %eax,%ss
   0x7c40:      mov    $0x0,%ebp
   0x7c45:      mov    $0x7c00,%esp
   0x7c4a:      call   0x7d11
```
经观察可以发现与bootasm一致

## 练习2.4

在调用bootmain的地方`0x7d11`添加断点，可观察到之后如下
```
=> 0x7d11:      push   %ebp
   0x7d12:      xor    %ecx,%ecx
   0x7d14:      mov    %esp,%ebp
   0x7d16:      mov    $0x1000,%edx
   0x7d1b:      push   %esi
   0x7d1c:      mov    $0x10000,%eax
   0x7d21:      push   %ebx
   0x7d22:      call   0x7c72
   0x7d27:      cmpl   $0x464c457f,0x10000
   0x7d31:      jne    0x7d72

```

# 练习3

通过阅读bootasm.S和相关资料，大致分析如下
```
    cli                                             # Disable interrupts
    cld                                             # String operations increment

    # Set up the important data segment registers (DS, ES, SS).
    xorw %ax, %ax                                   # Segment number zero
    movw %ax, %ds                                   # -> Data Segment
    movw %ax, %es                                   # -> Extra Segment
    movw %ax, %ss                                   # -> Stack Segment
```
cli屏蔽中断，cld复位操作方向标志，然后将各个段寄存器清零
```
seta20.1:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port
```
首先等待读入缓存为空，然后发送写指令
```
seta20.2:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
```
等待为空后写入将P2引脚位1置1,从而开启A20，避免地址“回卷”的情况出现

通过阅读注释以及查询
```
    lgdt gdtdesc
```
加载全局描述符表GDTR
```
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0
```
cr0是系统内的控制寄存器，第一位用来表示保护模式的开启与否，所以这几步的意义就是将cr0
寄存器第一位置一从而开启保护模式
```
    # Jump to next instruction, but in 32-bit code segment.
    # Switches processor into 32-bit mode.
    ljmp $PROT_MODE_CSEG, $protcseg
```
进行长跳转指令将处理器切换到32位模式(好像是会迫使处理器完成管道流水中不同状态啥啥啥的...)
```
    # Set up the protected-mode data segment registers
    movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
    movw %ax, %ds                                   # -> DS: Data Segment
    movw %ax, %es                                   # -> ES: Extra Segment
    movw %ax, %fs                                   # -> FS
    movw %ax, %gs                                   # -> GS
    movw %ax, %ss                                   # -> SS: Stack Segment
```
设置好段寄存器
```
    # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
    movl $0x0, %ebp
    movl $start, %esp
    call bootmain
```
设置好堆栈进入bootmain主方法

# 练习4

进入bootmain方法后，首先读取elf文件头
```
    // read the 1st page off disk
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
```
这里调用的函数可以读取硬盘中从某一点开始任意长度的内容
```
    // If this is too slow, we could read lots of sectors at a time.
    // We'd write more to memory than asked, but it doesn't matter --
    // we load in increasing order.
    for (; va < end_va; va += SECTSIZE, secno ++) {
        readsect((void *)va, secno);
    }
```
函数中真正读取硬盘的是这一段循环，读取某一扇区至某一位置，注释中说明了这样会将整个扇区读出导致多读出内容，但是并不重要

对应的实际读取硬盘扇区的函数
```
    // wait for disk to be ready
    waitdisk();

    outb(0x1F2, 1);                         // count = 1
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors

    // wait for disk to be ready
    waitdisk();

    // read a sector
    insl(0x1F0, dst, SECTSIZE / 4);
```
读取相关参考资料可以知道，读取硬盘首先要等待硬盘准备好，也就是`waitdisk`函数所做的事情，然后将各IO地址写好参数后一样等待硬盘准备好，而后读取相关数据

获取到elf文件头，要先判断是不是合法的elf文件

神奇的魔法数字～
```
    // is this a valid ELF?
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }
```
根据elf头部信息，将文件读入内存
```
    // load each program segment (ignores ph flags)
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }
```
最后找到入口，进入入口
```
    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
```

# 练习5

基本按照注释以及参考书写，其中几个对我来说的要点如下：
* uint32_t只是对int的类型重写，起到统一类型，保证安全的作用
* 对无符号十六进制整数输出用%x，%08x表示若数不足补至八位用零补
* 取到的ebp与eip是一个地址，在找栈的下一个地址或找相应参数时要将其强制类型转换为指针，将它的值作为地址来寻找
而后得到的输出如下
```
ebp:0x00007b28 eip:0x00100a63 args:0x00007b30 0x00007b34 0x00007b38 0x00007b3c 
    kern/debug/kdebug.c:294: print_stackframe+21
ebp:0x00007b38 eip:0x00100d42 args:0x00007b40 0x00007b44 0x00007b48 0x00007b4c 
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp:0x00007b58 eip:0x00100092 args:0x00007b60 0x00007b64 0x00007b68 0x00007b6c 
    kern/init/init.c:48: grade_backtrace2+33
ebp:0x00007b78 eip:0x001000bc args:0x00007b80 0x00007b84 0x00007b88 0x00007b8c 
    kern/init/init.c:53: grade_backtrace1+38
ebp:0x00007b98 eip:0x001000db args:0x00007ba0 0x00007ba4 0x00007ba8 0x00007bac 
    kern/init/init.c:58: grade_backtrace0+23
ebp:0x00007bb8 eip:0x00100101 args:0x00007bc0 0x00007bc4 0x00007bc8 0x00007bcc 
    kern/init/init.c:63: grade_backtrace+34
ebp:0x00007be8 eip:0x00100055 args:0x00007bf0 0x00007bf4 0x00007bf8 0x00007bfc 
    kern/init/init.c:28: kern_init+84
ebp:0x00007bf8 eip:0x00007d72 args:0x00007c00 0x00007c04 0x00007c08 0x00007c0c 
    <unknow>: -- 0x00007d71 --
```
其中每次输出地址及参数后的下一行是按照注释调用`print_debuginfo`函数的输出，阅读相关后发现输出当前指向的函数名及相对位置

注释真的太太太全了orz

# 练习6

## 练习6.1

通过阅读相关资料可以看到，中断向量表每项8字节，3、4字节存储了段选择子，通过查询段描述符得到的段基址加上1、2,7、8字节存储的偏移位置就可以得到结果也就是中断向量的入口

## 练习6.2

在对应位置通过阅读注释可以发现如何获得中断服务程序偏移量数组`__vectors[]`，而为了填充中断描述符表我们阅读`mmu.h`中的SETGATE宏，发现还需要自行确定的参数有
* 门的类型
* 中断服务程序的段选择子
* 特权等级

而中断服务程序的段选择子比较难以寻找，在经过阅读相关资料以及参考答案并仔细对比之后，可以发现vectors.S的程序被放在.data段下面，在通过生成vectors.S的vectors.c可以找到在文件`memlayout.h`中对相关代码段有所定义，特权等级根据描述除了系统调用为用户态以外其他均为内核态，而Gate在系统调用处用了Trap Gate而其他皆为Interrupt Gate

最后按照注释调用lidt指令加载idt即可

## 练习6.3

按照注释添加即可

# 扩展练习

## 扩展练习1

为了能够在用户态切换回系统态，我们必须要注意的是要将切换内核态的中断的特权等级必须是用户态，即`     SETGATE(idt[T_SWITCH_TOK],1,GD_KTEXT,__vectors[T_SWITCH_TOK],DPL_USER);`，而后在处理中断时在对应位置加上处理代码，即根据当前的trapframe进行处理，该结构定义在`trap.h`中，其中的`__attribute__((packed))`是为了避免c中结构体的自动对齐等问题而写出的，结构中的很多`uint16_t`是为了填充空隙并保持对齐的

栈是从高地址向低地址发展，而c中给定结构体地址却是从低地址向高地址中看，所以定义的顺序与实际压栈的顺序正好相反，在用户态向内核态转换时会自动压入ss和esp所以在内核态向用户态转换时要在最开始留出额外空间从而与结构体中的ss和esp位置对应，另外x8086体系中是小端对齐，填充的空隙补满之后可以直接取后半部分作为16位寄存器修改或者类型转换为32位作为32位寄存器修改

这里在后来才发现kernel会把系统的代码段放在ts寄存器里在恢复时可以直接使用，所以这个可以自动压入而反之不行orz

在压入eflags，cs，eip等后进入中断向量，即vectors中，观察及阅读可以看到压入了erroeCode、trapno，然后进入统一处理中断函数即`trapentry.S`中，观察可以发现压入各个段寄存器后通过`pushal`指令存入其他普通寄存器值，而后压入esp作为参数而后调用`trap.c`中的trap函数而后即进入我们定义的处理函数

我们为了实现用户态和内核态的转换即将栈中的段寄存器更改为对应的段位置即可，而在`init.c`中规定的调用函数我们用了gcc的扩展内联汇编代码，每一行后跟换行符与制表符，用的操作数用常量替换用i表示，其中向用户态切换的如前文所说要提前留出空间

参考答案的做法是建了新的用户栈，然而经过尝试我们发现...改段寄存器就完了=_=

这样修改完成后进行尝试会发现在尝试向用户态转换时就会卡死，经过仔细寻找并参考了答案的设置后，我们发现是eflag中有两位是io控制字段，在用户态时由于权限等级不够导致卡死，所以我们只要在状态切换时更改eflag将这两位修改为三即可`FL_IOPL_MASK`实际与`FL_IOPL_3`效果相同

扩展练习真的是扩展...啥提示都没有啊...搜这搜那看了一天啊...加上临近国庆期间形势严峻...谷歌差点不能用了orz

## 扩展练习2

键盘中断的位置也同样在`trap.c`中，只要对得到的字符判断并与之前的中断方案类似即可