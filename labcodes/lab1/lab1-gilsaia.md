# 练习1

## 练习1.1

通过`make "V="`命令，我们可以观察到make实际执行了哪些命令。

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
`-Os`是为了优化文件大小的设置，还生成了`sign`的文件，都生成后链接生成bootblock，其中`-N`为设置代码段和数据段均可读写，`-e`设置入口，`-Ttext`设置代码段开始位置

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

经过阅读相关资料后得知，BIOS在进行完硬件自检以及初始化后，会将启动设备的第一扇区也就是主引导扇区放到内存的0x7c00处，在ucore中也就是将我们之前生成的bootblock块拷贝入内存，而之前生成bootblock块的命令也可以证实这一点，而显示的信息以及makefile的命令都指出链接生成bootblock后还经过了sign程序，从sign的报错来看bootblock块也就是主引导扇区大小应该不能超过512字节，而最后两个字节应已规定好不能写入其他，倒数第二个字节是0x55,倒数第一个字节是0xAA

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
，`si`可以让cpu单步执行从而观察BIOS指令

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
进行长跳转指令将处理器切换到32位模式
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