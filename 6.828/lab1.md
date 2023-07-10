

# The PC's Physical Address Space

<img src="E:\Typora\notes\6.828\pics\mem layout.JPG" alt="内存空间布局" style="zoom: 80%;" />

第一批基于16位Intel 8088处理器的PC只能寻址1MB的物理内存。因此，早期PC的物理地址空间将以0x00000000开始，但以0x000FFFFF结束，而不是0xFFFFFF。标记为“地位地址”的640KB区域是早期PC唯一可以使用的随机存取存储器（RAM）；事实上，最早的PC只能配置16KB、32KB或64KB的RAM！

从0x000A0000到0x000FFFFF的384KB区域由硬件保留，用于特殊用途，如视频显示缓冲区和非易失性的固件。这个保留区域中最重要的部分是基本输入/输出系统（BIOS），它占用从0x000F0000到0x000FFFFF的64KB区域。在早期的个人电脑中，BIOS保存在真正的只读存储器（ROM）中，但当前的个人电脑将BIOS存储在可更新的闪存中。BIOS负责执行基本的系统初始化，如激活显卡和检查安装的内存量。执行此初始化后，BIOS将从软盘、硬盘、CD-ROM或网络等适当位置加载操作系统，并将机器的控制权传递给操作系统。

当英特尔最终用80286和80386处理器“打破了1兆字节的壁垒”时，PC架构师们仍然保留了原来的1兆物理地址空间布局，以确保与现有软件的向后兼容性。因此，现代PC在物理内存中有一个从0x000A0000到0x00100000的“洞”，将RAM分为“低”或“常规内存”（前640KB）和“扩展内存”（其他所有内存）。此外，在PC的32位物理地址空间的最上面的一些空间，在所有物理RAM之上，通常由BIOS保留供32位PCI设备使用。

# The ROM BIOS

```[f000:fff0] 0xffff0:	ljmp   $0xf000,$0xe05b```是GDB对要执行的第一条指令的反汇编。从这个输出可以得出以下结论：

IBM PC在物理地址0x000ffff0处开始执行，该地址位于为rombios保留的64KB区域的最顶端。
PC开始执行时CS  =  0xf000，IP  =  0xfff0。
要执行的第一条指令是jmp指令，它跳转到分段地址CS=0xf000和IP=0xe05b。

在实模式（PC启动的模式）下，地址转换的工作原理是：物理地址=16*段+偏移量。因此，当PC将CS设置为0xf000，IP设置为0xfff0时，引用的物理地址为：

16  *  0xf000  +  0xfff0

=  0xf0000  +  0xfff0 

=  0xffff0

当BIOS运行时，它会设置一个中断描述符表，并初始化各种设备，如VGA显示器。然后会在QEMU窗口中看到的“Starting SeaBIOS”的消息。
**在初始化PCI总线和BIOS知道的所有重要设备之后，它会搜索可引导设备，如软盘、硬盘或CD-ROM。最终，当它找到可引导磁盘时，BIOS会从磁盘读取引导加载程序，并将控制权传输给它。**

# The Boot Loader



如果磁盘是可引导的，则第一个扇区称为引导扇区，因为这是引导加载程序代码所在的位置。当BIOS找到可引导软盘或硬盘时，它将512字节的引导扇区加载到物理地址0x7c00到0x7dff的内存中，然后使用jmp指令将CS:IP设置为0000:7c00，将控制权传递给引导加载程序。

首先，引导加载程序将处理器从实模式切换到32位保护模式，因为只有在这种模式下，软件才能访问处理器物理地址空间中1MB以上的所有内存。分段地址（段：偏移对）到物理地址的转换在保护模式下的发生方式不同，转换后的偏移量是32位而不是16位。

其次，引导加载程序通过x86的特殊I/O指令直接访问IDE磁盘设备寄存器，从硬盘读取内核。

查看`boot/boot.S`，`boot/main.c`，` obj/boot/boot.asm`，`obj/kern/kernel.asm`，并跟踪引导程序的执行，回答以下问题。

```
At what point does the processor start executing 32-bit code? What exactly causes the switch from 16- to 32-bit mode?
What is the last instruction of the boot loader executed, and what is the first instruction of the kernel it just loaded?
Where is the first instruction of the kernel?
How does the boot loader decide how many sectors it must read in order to fetch the entire kernel from disk? Where does it find this information?
```



##exercise 4.

运行代码，请确保了解打印的第1行和第6行中的指针地址来自何处，打印的第2行到第4行中的所有值是如何到达那里的，以及为什么打印的第5行中的值看起来已损坏。

```c
#include <stdio.h>
#include <stdlib.h>

void f(void) {
    int a[4];
    int *b = malloc(16);
    int *c;
    int i;

    printf("1: a = %p, b = %p, c = %p\n", a, b, c);

    c = a;
    for (i = 0; i < 4; i++)
        a[i] = 100 + i;
    c[0] = 200;
    printf("2: a[0] = %d, a[1] = %d, a[2] = %d, a[3] = %d\n", a[0], a[1], a[2],
           a[3]);

    c[1] = 300;
    *(c + 2) = 301;
    3 [c] = 302;
    printf("3: a[0] = %d, a[1] = %d, a[2] = %d, a[3] = %d\n", a[0], a[1], a[2],
           a[3]);

    c = c + 1;
    *c = 400;
    printf("4: a[0] = %d, a[1] = %d, a[2] = %d, a[3] = %d\n", a[0], a[1], a[2],
           a[3]);

    c = (int *)((char *)c + 1);
    *c = 500;
    printf("5: a[0] = %d, a[1] = %d, a[2] = %d, a[3] = %d\n", a[0], a[1], a[2],
           a[3]);

    b = (int *)a + 1;
    c = (int *)((char *)a + 1);
    printf("6: a = %p, b = %p, c = %p\n", a, b, c);
}

int main(int ac, char **av) {
    f();
    return 0;
}
```

代码输出

```
1: a = 0x7ffe391424c0, b = 0x5619501c7260, c = 0xf0b6ff
2: a[0] = 200, a[1] = 101, a[2] = 102, a[3] = 103
3: a[0] = 200, a[1] = 300, a[2] = 301, a[3] = 302
4: a[0] = 200, a[1] = 400, a[2] = 301, a[3] = 302
5: a[0] = 200, a[1] = 128144, a[2] = 256, a[3] = 302
6: a = 0x7ffe391424c0, b = 0x7ffe391424c4, c = 0x7ffe391424c1
```

输出5解析：c的值为`0x7ffe391424c5`，500的16进制为`0x000001f4`，400的16进制为`0x00000190`，301的16进制为`0x0000012C`，256的16进制为`0x00000100`，本机为数据存储为小端模式。如500的16进制小端模式存储为`0xf4010000`，

| `0x7ffe391424c4` | `0x7ffe391424c5` | `0x7ffe391424c6` | `0x7ffe391424c7` | `0x7ffe391424c8` | `0x7ffe391424c9` | `0x7ffe391424c5A` | `0x7ffe391424cB` |
| :--------------: | :--------------: | :--------------: | :--------------: | :--------------: | :--------------: | :---------------: | :--------------: |
|        90        |        01        |        00        |        00        |       `2D`       |        01        |        00         |        00        |
|        90        |       `F4`       |        01        |        00        |        00        |        01        |        00         |        00        |

从`0x7ffe391424c4`到`0x7ffe391424c7`为400按小端模式存储，从`0x7ffe391424c8`到`0x7ffe391424cB`为301按小端模式存储。然后将500按小端模式存储到`0x7ffe391424c5`到`0x7ffe391424c8`。所以然后a[1]的16进制表示为`0x0001f490`，10进制为128144，a[1]的16进制表示为`0x00000100`，10进制为256。



#Formatted Printing to the Console

##exercise 8

```c
printfmt.c文件里的Vprintfmt函数里的switch里的一个case

case 'o':
			// Replace this with your code.
			// putch('X', putdat);
			// putch('X', putdat);
			// putch('X', putdat);
			num = getuint(&ap,lflag);
			base = 8;
			goto number;
```

1. cputchar函数，被putch函数调用

2. 在屏幕被占满之后，整个屏幕向上移动一行，即屏幕的第一行消失，最后面一行空白

3. 在kern/monitor.c的mon_backtrace里添加以下代码并执行。

   ```c
   	int x = 1, y = 3, z = 4;
   	cprintf("x %d, y %x, z %d\n", x, y, z);
   ```

4.  ```
      unsigned int i = 0x00646c72;
      cprintf("H%x Wo%s", 57616, &i);
    ```

   57616的16进制表示为e110，0x00646c72的按自己转化为char型后为\0dlr，又因为是小端模式，实际按地址从低到高为726c6400，按%s输出为rld\0，所以cprintf的输出时He110 World。  大端模式时i的值应设为0x726c6400，57616不用改变。

5. 。。

6. 。。

# The Stack

编写一个有用的新的内核监视器函数，该函数打印堆栈的回溯：一个从嵌套调用指令（导致当前执行点）保存的指令指针（IP）值的列表。

##exercise 9

```Determine where the kernel initializes its stack, and exactly where in memory its stack is located. How does the kernel reserve space for its stack? And at which "end" of this reserved area is the stack pointer initialized to point to? 栈指针指向保留区域的哪一端？```

```
	# Clear the frame pointer register (EBP)
	# so that once we get into debugging C code,
	# stack backtraces will be terminated properly.
	movl	$0x0,%ebp			# nuke frame pointer

	# Set the stack pointer
	movl	$(bootstacktop),%esp
	# now to C code
	call	i386_init
```

在entry.S的71行之后，显示如上汇编代码。在初始化i386之前，把0x0作为ebp栈底，保证栈追踪函数可以顺利结束，把$(bootstacktop)作为栈顶。x86堆栈指针（esp寄存器）指向当前正在使用的堆栈上的最低位置。在为堆栈保留的区域中，该位置下方的所有内容都是可用的。将值复制到堆栈上需要减少堆栈指针，然后将值写入堆栈指针所指向的位置。从堆栈弹出一个值涉及读取堆栈指针指向的值，然后增加堆栈指针。栈顶指针指向保留区域的上面，因为栈向下增长。

##exercise 10

```To become familiar with the C calling conventions on the x86, find the address of the `test_backtrace` function in `obj/kern/kernel.asm`, set a breakpoint there, and examine what happens each time it gets called after the kernel starts. How many 32-bit words does each recursive nesting level of `test_backtrace` push on the stack, and what are those words?```

test_backtrace函数是一个递归函数，输入一个参数，当参数减到0时调用跟踪函数。

```c
void
test_backtrace(int x)
{
	cprintf("entering test_backtrace %d\n", x);
	if (x > 0)
		test_backtrace(x-1);
	else
		mon_backtrace(0, 0, 0);
	cprintf("leaving test_backtrace %d\n", x);
}

```

## exercise 11

```Implement the backtrace function.```

要求按一下格式输出:

```
Stack backtrace:
  ebp f0109e58  eip f0100a62  args 00000001 f0109e80 f0109e98 f0100ed2 00000031
  ebp f0109ed8  eip f01000d6  args 00000000 00000000 f0100058 f0109f28 00000061
```

ebp的值为当前函数的栈底指针，*ebp，即栈底指针所指地址保存的值为调用当前函数的函数的栈底指针。eip的值为当前函数执行完毕之后的返回地址，也就是调用当前函数的call指令之后的那条指令的地址。args为当前函数的参数，当参数没有5个时，多余显示的数值就不是参数。

Q. 那么为什么跟踪函数无法检测出有多少个参数？怎么解决这个问题？

A. 

Q. 跟踪函数在怎么结束？

A. 在exercise9里提到最开始ebp被初始化为0x0，所以当ebp的值为0时跟踪函数就结束。

```c
int mon_backtrace(int argc, char **argv, struct Trapframe *tf) {
    // Your code here.
    uint32_t now_ebp = read_ebp();
    while (now_ebp) {
        uint32_t last_pc = now_ebp + 4;
        uint32_t last_param = last_pc + 4; 
        cprintf("ebp %x  eip %x  args", now_ebp, *((uint32_t *)last_pc));
        for (int i = 0; i < 20; i += 4) {
            cprintf(" %08x", *((uint32_t *)(last_param + i)));
        }
        cprintf("\n");
        now_ebp = *((uint32_t *)now_ebp);
    }
    return 0;
}
```

mon_traceback的实现如上，使用read_ebp()函数得到就能得到当前执行函数的栈底指针，ebp的值+4是返回地址eip，再+4就是参数的地址了。然后将ebp的值设为上层调用函数的ebp即可追踪到所以调用函数。

## exercise 12

```
Modify your stack backtrace function to display, for each eip, the function name, source file name, and line number corresponding to that eip.

Complete the implementation of debuginfo_eip by inserting the call to stab_binsearch to find the line number for an address.

Add a backtrace command to the kernel monitor, and extend your implementation of mon_backtrace to call debuginfo_eip and print a line for each stack frame of the form:

K> backtrace
Stack backtrace:
  ebp f010ff78  eip f01008ae  args 00000001 f010ff8c 00000000 f0110580 00000000
         kern/monitor.c:143: monitor+106
  ebp f010ffd8  eip f0100193  args 00000000 00001aac 00000660 00000000 00000000
         kern/init.c:49: i386_init+59
  ebp f010fff8  eip f010003d  args 00000000 00000000 0000ffff 10cf9a00 0000ffff
         kern/entry.S:70: <unknown>+0
K> 
```

新增加的打印要求的值分别是调用当前函数的函数所在的文件、在调试符号表中eip和调用函数之间的行数差、调用函数名和eip和调用函数的地址之间的字节差。

为了帮助实现这一功能，使用函数debuginfo_ eip()，它在symbol表中查找eip并返回该地址的调试信息。这个函数在kern/kdebug.c中定义。

debuginfo_eip的参数为eip的值，即一个地址，一个Eipdebuginfo类型的指针。Eipdebuginfo结构的定义如下。

```c
// Debug information about a particular instruction pointer
struct Eipdebuginfo {
	const char *eip_file;		// Source code filename for EIP
	int eip_line;			// Source code linenumber for EIP

	const char *eip_fn_name;	// Name of function containing EIP
					//  - Note: not null terminated!
	int eip_fn_namelen;		// Length of function name
	uintptr_t eip_fn_addr;		// Address of start of function
	int eip_fn_narg;		// Number of function arguments
};


//下面时stab的结构体
struct Stab {
	uint32_t n_strx;	// index into string table of name
	uint8_t n_type;         // type of symbol
	uint8_t n_other;        // misc info (usually empty)
	uint16_t n_desc;        // description field
	uintptr_t n_value;	// value of symbol，地址值
};
```

debuginfo_eip函数获取一个eip地址的Eipdebuginfo中的信息依靠调试符号表。在stabs-stabs_end之间进行二分搜索，找到存储相应信息的stabs[i]，将部分值赋给info的成员。该函数实现中计算eip地址与该指令所在函数首地址的行数偏移功能没有实现，具体实现如下。

```
	// Your code here.
	stab_binsearch(stabs,&lline,&rline,N_SLINE,addr);
	if(lline > rline)
		return -1;
	info->eip_line =lline - lfun;
```

调用debuginfo_eip()函数后，按要求输出info的成员。

```c
int mon_backtrace(int argc, char **argv, struct Trapframe *tf) {
    // Your code here.
    uint32_t now_ebp = read_ebp();
    while (now_ebp) {
        uint32_t last_pc = now_ebp + 4;
        uint32_t last_param = last_pc + 4; // args %x\n
        cprintf("ebp %x  eip %x  args", now_ebp, *((uint32_t *)last_pc));
        for (int i = 0; i < 20; i += 4) {
            cprintf(" %08x", *((uint32_t *)(last_param + i)));
        }
        cprintf("\n");
        struct Eipdebuginfo info;
        debuginfo_eip(*((uint32_t *)last_pc), &info);
        cprintf("%s:%d: %.*s", info.eip_file, info.eip_line,
                info.eip_fn_namelen, info.eip_fn_name);
        cprintf("+%d\n", -info.eip_fn_addr + *((uint32_t *)last_pc));

        now_ebp = *((uint32_t *)now_ebp);
    }

    return 0;
}
```

执行```make clean; make; make grade;```命令后，输出以下信息，拿到满分。

![lab1打分](E:\Typora\notes\6.828\pics\lab1评分.JPG)

执行```make qemu```得到的输出如下图。

![lab1make qemu](E:\Typora\notes\6.828\pics\lab1_make_qemu.JPG)

