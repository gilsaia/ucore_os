# 练习一

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

final test