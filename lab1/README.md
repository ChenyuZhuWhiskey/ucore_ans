# Lab1
## exercise
### exercise 1：理解通过make生成执行文件的过程

- 1.操作系统镜像文件ucore.img是如何一步步生成的？
- 2.一个被系统认为符合规范的硬盘主引导扇区的特征是什么？
  
1.这个要读makefile,在create ucore.img注释下面：
```Makefile
UCOREIMG := $(call totarget, ucore.img)
```
可以看到生成ucore.img调用了一个`totarget`函数，这个函数定义在了`tools/function.mk`里：
```Makefile
totarget = $(addprefix $(BINDIR)$(SLASH),$(1))
```
这个函数只是把ucore.img加一个prefix，结果就是bindir/ucore.img，也就是生成了镜像文件的位置
然后就是具体的生成步骤：
```Makefile
$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
```
因此具体的过程就是用3次dd命令，用/dev/zer创建了一块大的匿名的虚拟内存，把kernel和bootblock复制到文件中就制作好了镜像。

UCOREIMG的依赖是kernel和bootlock，那我们来看一下目标kernel是怎么生成的：
```Makefile
kernel = $(call totarget,kernel)
$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
```

就是一个借助tools/kernel.ld的链接脚本进行链接的过程，读懂之前我们要先看一下这些宏定义是什么：
- `V:= @`，这一行不打印
- `LD:= $(GCCPREFIX)ld` 链接命令
- `KOBJS=$(call read_packet,kernel libs)`

kernel的依赖是KOBJS，因此在生成kernel前会先生成KOBJS，`read_packet`函数在`tools/function.mk`中定义：
```Makefile
read_packet = $(foreach p,$(call packetname,$(1)),$($(p)))
# change $(name) to $(OBJPREFIX)$(name): (#names)
packetname = $(if $(1),$(addprefix $(OBJPREFIX),$(1)),$(OBJPREFIX))
```
最后生成的KOBJS就是这些：
```
obj/kern/init/init.o 
obj/kern/libs/stdio.o 
obj/kern/libs/readline.o 
obj/kern/debug/panic.o 
obj/kern/debug/kdebug.o 
obj/kern/debug/kmonitor.o 
obj/kern/driver/clock.o 
obj/kern/driver/console.o 
obj/kern/driver/picirq.o 
obj/kern/driver/intr.o 
obj/kern/trap/trap.o 
obj/kern/trap/vectors.o 
obj/kern/trap/trapentry.o 
obj/kern/mm/pmm.o 
obj/libs/string.o 
obj/libs/printfmt.o
```
而这些.o文件又由这两行命令进行编译：
```Makefile
$(call add_files_cc,$(call listf_cc,$(LIBDIR)),libs,)
$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))
```
(说实话这里的定义展开后没看懂，但确实做到了将这些文件编译)
生成bootlock的代码如下：
```Makefile
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

bootblock = $(call totarget,bootblock)

$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJDUMP) -t $(call objfile,bootblock) | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
```

将bootfiles编译，然后进行链接，flag -Ttext 0x7C00表示这段代码从0x7c00开始执行，也就是mbr的开始地址。

2.一个符合规范的主引导扇区标准是最后两个字节为0x55aa

### exercise 2：使用qemu执行并调试lab1中的软件
为了从mbr开始调试，首先修改tools/gdbinit:
```
set architecture i8086
target remote:1234
break *0x7c00
continue
```
然后按照提示`make lab1-mon`，编译后弹出debug的窗口，显示第一行mbr的命令在0x7c00处：
```
Breakpoint 1, 0x00007c00 in ?? ()
=> 0x7c00:	cli    
   0x7c01:	cld 
```
然后接着调试，代码和boot/bootasm.S是一样的
### exercise 3：分析bootloader进入保护模式的过程
- 打开A20
- 加载gdt
- 设置CR0首位为1

### exercise 4：分析bootloader加载ELF格式OS的过程
1.bootloader是如何读取硬盘扇区的？
见`readsect()`函数：
```c
/* readsect - read a single sector at @secno into @dst */
static void
readsect(void *dst, uint32_t secno) {
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
}
```

0x1F2端口表示要读的扇区数，这里是1。

0x1F3-0x1F6是LBA的地址，前三个端口8位，最后一个端口是24-27位。

0x1F7是命令端口，0x20就是如果端口0x1F0不忙就执行读命令

2.bootloader是如何加载ELF格式的OS？
见`bootmain()`函数：
```c
void
bootmain(void) {
    // read the 1st page off disk
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    // is this a valid ELF?
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

    struct proghdr *ph, *eph;

    // load each program segment (ignores ph flags)
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();

bad:
    outw(0x8A00, 0x8A00);
    outw(0x8A00, 0x8E00);

    /* do nothing */
    while (1);
}
```

可以看出步骤如下：
- 从硬盘读8个扇区内容，强制转换为elfheader使用
- 检查magic
- 根据偏移量把相关内容加载到内存中运行

### 练习5：实现函数调用堆栈跟踪函数
函数调用栈的布局如下：
```
    |第n个变量    |         高地址
    ...
    |第2个变量    | 
    |第1个变量    | 
    |原来的ebp值  | <- ebp
    |函数栈局部变量|
    |函数栈局部变量| <- esp  低地址
```

要实现堆栈跟踪，就要拿到ebp的值，这样就能得到前一个ebp的值和返回地址。
```c
void
print_stackframe(void) {
     /* LAB1 YOUR CODE : STEP 1 */
     /* (1) call read_ebp() to get the value of ebp. the type is (uint32_t);
      * (2) call read_eip() to get the value of eip. the type is (uint32_t);
      * (3) from 0 .. STACKFRAME_DEPTH
      *    (3.1) printf value of ebp, eip
      *    (3.2) (uint32_t)calling arguments [0..4] = the contents in address (unit32_t)ebp +2 [0..4]
      *    (3.3) cprintf("\n");
      *    (3.4) call print_debuginfo(eip-1) to print the C calling function name and line number, etc.
      *    (3.5) popup a calling stackframe
      *           NOTICE: the calling funciton's return addr eip  = ss:[ebp+4]
      *                   the calling funciton's ebp = ss:[ebp]
      */
    uint32_t ebp = read_ebp(), eip = read_eip();
    for (int i = 0; i < STACKFRAME_DEPTH && ebp != 0; i++) {
        cprintf("ebp: 0x%08x eip: 0x%08x args:", ebp, eip);
        for (int ij= 0; j < 4; j++) {
            cprintf(" 0x%08x", ((uint32_t*)(ebp + 2))[j]);
        }
        cprintf("\n");
        print_debuginfo(eip - 1);
        eip = *((uint32_t*) ebp + 1);
        ebp = *((uint32_t*) ebp);
    }
}
```
### exercise 6: 完善中断初始化和处理
1.中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

中断描述符表一个表项8字节，其中16-31位是中断历程的段选择子，0到15位和第48到63位分别为偏移量的地位和高位

2.请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。在idt_init函数中，依次对所有中断入口进行初始化。使用mmu.h中的SETGATE宏，填充idt数组内容。每个中断的入口由tools/vectors.c生成，使用trap.c中声明的vectors数组即可。

```c
/* idt_init - initialize IDT to each of the entry points in kern/trap/vectors.S */
void
idt_init(void) {
    // (1) 拿到外部变量 __vector
    extern uintptr_t __vectors[];
    // (2) 使用SETGATE宏，对中断描述符表中的每一个表项进行设置
    for (int i = 0; i < 256; i++) {
        uint16_t istrap = 0, off = 0, dpl = 3;
        SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
    }
    // set for switch from user to kernel 
    SETGATE(idt[T_SWITCH_TOU], 0, GD_KTEXT, __vectors[T_SWITCH_TOU], DPL_USER);
    SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);
    // (3) 调用lidt函数，设置中断描述符表
    lidt(&idt_pd);
}
```

3.请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字”100 ticks”。

声明一个静态变量
```c
static int32_t tick_count = 0;
```
然后在时间中断`IRQ_OFFSET + IRQ_TIMER`的case中添加打印条件：
```c
tick_count++;
        if (0 == (tick_count % TICK_NUM)) {
            print_ticks();
        }
```
