lab1 报告

练习1.1

生成ucore.img的代码如下

# create ucore.img
UCOREIMG	:= $(call totarget,ucore.img)

$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)

生成ucore.img 依赖于kernel和bootblock的生成

# create kernel target
kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)

# -------------------------------------------------------------------

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

用make V= 可以看到
为了生成bootblock，首先需要生成bootasm.o、bootmain.o、sign

练习1.2

sign.c中部分代码如下

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
    if (size != 512) {
        fprintf(stderr, "write '%s' error, size is %d.\n", argv[2], size);
        return -1;
    }
可以看到，一个磁盘主引导扇区要512字节。且第510个字节是0x55，第511个字节是0xAA。

练习2

在lab1/tools/gdbinit中添加
set architecture i8086

然后执行
make debug

b 0x7c00插入断电
c运行到断电处
s单布调试
截图见 练习2.png

练习3

从`%cs=0 $pc=0x7c00`，进入保护模式后

首先将flag置0和将段寄存器置0
.code16                                             # Assemble for 16-bit mode
    cli                                             # Disable interrupts
    cld                                             # String operations increment

    xorw %ax, %ax                                   # Segment number zero
    movw %ax, %ds                                   # -> Data Segment
    movw %ax, %es                                   # -> Extra Segment
    movw %ax, %ss                                   # -> Stack Segment


开启A20：通过将键盘控制器上的A20线置于高电位，全部32条地址线可用，
可以访问4G的内存空间。

seta20.1:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port

seta20.2:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1

初始化GDT表
    lgdt gdtdesc
进入保护模式
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0


通过长跳转更新cs的基地址
    jmp $PROT_MODE_CSEG, $protcseg

设置段寄存器，并建立堆栈
    movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
    movw %ax, %ds                                   # -> DS: Data Segment
    movw %ax, %es                                   # -> ES: Extra Segment
    movw %ax, %fs                                   # -> FS
    movw %ax, %gs                                   # -> GS
    movw %ax, %ss 

转到保护模式完成，进入boot主方法 
    movl $0x0, %ebp
    movl $start, %esp
    call bootmain

练习4

从bootmain函数中可以看到
首先是readseg函数读取硬盘扇区
而readseg函数是调用了readsect函数
readsect(void *dst, uint32_t secno) {
    waitdisk(); // 等待硬盘就绪
    // 写地址0x1f2~0x1f5,0x1f7,发出读取磁盘的命令
    outb(0x1F2, 1);
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);
    waitdisk();
    insl(0x1F0, dst, SECTSIZE / 4);//读取一个扇区
}



在bootmain函数中，
bootmain(void) {
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
    //首先判断是不是ELF
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;                 
    }
    struct proghdr *ph, *eph;

    //ELF头部有描述ELF文件应加载到内存什么位置的描述表，这里读取出来将之存入ph
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;

    //按照程序头表的描述，将ELF文件中的数据载入内存
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }
    //根据ELF头表中的入口信息，找到内核的入口并开始运行 
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
	
    bad:
        outw(0x8A00, 0x8A00);
        outw(0x8A00, 0x8E00);
        while (1);
	}

练习5

代码中循环时需要判断ebp是否为0

qemu 模拟器输出：
ebp:0x00007b38 eip:0x00100bcc args:0x00010094 0x0010e950 0x00007b68 0x001000a2 
    kern/debug/kdebug.c:306: print_stackframe+33
ebp:0x00007b48 eip:0x00100f7f args:0x00000000 0x00000000 0x00000000 0x0010008d 
    kern/debug/kmonitor.c:125: mon_backtrace+23
ebp:0x00007b68 eip:0x001000a2 args:0x00000000 0x00007b90 0xffff0000 0x00007b94 
    kern/init/init.c:48: grade_backtrace2+32
ebp:0x00007b88 eip:0x001000d1 args:0x00000000 0xffff0000 0x00007bb4 0x001000e5 
    kern/init/init.c:53: grade_backtrace1+37
ebp:0x00007ba8 eip:0x001000f8 args:0x00000000 0x00100000 0xffff0000 0x00100109 
    kern/init/init.c:58: grade_backtrace0+29
ebp:0x00007bc8 eip:0x00100124 args:0x00000000 0x00000000 0x00000000 0x001037d8 
    kern/init/init.c:63: grade_backtrace+37
ebp:0x00007be8 eip:0x00100066 args:0x00000000 0x00000000 0x00000000 0x00007c4f 
    kern/init/init.c:28: kern_init+101
ebp:0x00007bf8 eip:0x00007d6e args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8 
    <unknow>: -- 0x00007d6d --

最后一行为最深层堆栈，所以应该是bootmain.c中的bootmain函数。
bootloader设置的堆栈从0x7c00开始，使用"call bootmain"转入bootmain函数。所以0x00007bf8,是bootmain的ebp 。

练习6

练习6.1
中断描述符表一个表项占8字节。其中0~15位和48~63位分别为offset的低16位和高16位。16~31位为段选择子。通过段选择子获得段基址，加上段内偏移量即可得到中断处理代码的入口。

练习6.2

见代码
lidt函数用于加载中断描述符

这里的SETGATE的定义在mmu.h中
#define SETGATE(gate, istrap, sel, off, dpl) 
gate：处理函数的入口地址
istrap：系统段设置为1，中断门设置为0
sel：段选择子 ？？
off：为__vectors[]数组内容
dpl：设置特权级


练习6.3

见代码
