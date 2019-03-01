# lec2：lab0 SPOC思考题

## **提前准备**
（请在上课前完成，option）

- 完成lec2的视频学习
- git pull ucore_os_lab, os_tutorial_lab, os_course_exercises  in github repos。这样可以在本机上完成课堂练习。
- 了解代码段，数据段，执行文件，执行文件格式，堆，栈，控制流，函数调用,函数参数传递，用户态（用户模式），内核态（内核模式）等基本概念。思考一下这些基本概念在不同操作系统（如linux, ucore,etc.)与不同硬件（如 x86, riscv, v9-cpu,etc.)中是如何相互配合来体现的。
- 安装好ucore实验环境，能够编译运行ucore labs中的源码。
- 会使用linux中的shell命令:objdump，nm，file, strace，gdb等，了解这些命令的用途。
- 会编译，运行，使用v9-cpu的dis,xc, xem命令（包括启动参数），阅读v9-cpu中的v9\-computer.md文档，了解汇编指令的类型和含义等，了解v9-cpu的细节。
- 了解基于v9-cpu的执行文件的格式和内容，以及它是如何加载到v9-cpu的内存中的。
- 在piazza上就学习中不理解问题进行提问。

---

## 思考题

- 你理解的对于类似ucore这样需要进程/虚存/文件系统的操作系统，在硬件设计上至少需要有哪些直接的支持？至少应该提供哪些功能的特权指令？  
答：  
进程：时钟中断、用户态和内核态之间的切换；特权指令是从内核态切换到用户态的指令。  
虚存：段页式内存管理；特权指令是设置段基地址的指令，以及设置页目录起始地址的指令。  
文件系统：对磁盘的读取；特权指令是读取磁盘扇区内容的指令。

- 你理解的x86的实模式和保护模式有什么区别？物理地址、线性地址、逻辑地址的含义分别是什么？  
答：  
x86的实模式下只能使用20位的地址线（即寻址空间为1MB），且程序中使用的地址为物理地址，用户程序很容易破坏操作系统；  
保护模式下可以使用全部32位地址线（即寻址空间为4GB），支持段页式内存管理，支持多个特权级，能够防止用户程序破坏操作系统。  
物理地址的含义是数据在物理内存中存放的实际地址。  
线性地址是逻辑地址加上段基地址。  
逻辑地址是程序员在程序中使用的地址。

- 你理解的risc-v的特权模式有什么区别？不同模式在地址访问方面有何特征？  
答：  
RISC-V定义了四种特权级模式。  
机器级是最高级特权，也是RISC-V硬件平台唯一必须的特权级。运行于机器模式（M-mode）下的代码是固有可信的，因为它可以在低层次访问机器的实现。  
用户模式（U-mode）用于传统应用程序。  
管理员模式（S-mode）用于操作系统。  
Hypervisor 模式（H-mode）则是为了支持虚拟机监视器。 


- 理解ucore中list_entry双向链表数据结构及其4个基本操作函数和ucore中一些基于它的代码实现（此题不用填写内容）

- 对于如下的代码段，请说明":"后面的数字是什么含义
```
 /* Gate descriptors for interrupts and traps */
 struct gatedesc {
    unsigned gd_off_15_0 : 16;        // low 16 bits of offset in segment
    unsigned gd_ss : 16;            // segment selector
    unsigned gd_args : 5;            // # args, 0 for interrupt/trap gates
    unsigned gd_rsv1 : 3;            // reserved(should be zero I guess)
    unsigned gd_type : 4;            // type(STS_{TG,IG32,TG32})
    unsigned gd_s : 1;                // must be 0 (system)
    unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
    unsigned gd_p : 1;                // Present
    unsigned gd_off_31_16 : 16;        // high bits of offset in segment
 };
``` 
答：  
表示该域的二进制位数。

- 对于如下的代码段，

```
#define SETGATE(gate, istrap, sel, off, dpl) {            \
    (gate).gd_off_15_0 = (uint32_t)(off) & 0xffff;        \
    (gate).gd_ss = (sel);                                \
    (gate).gd_args = 0;                                    \
    (gate).gd_rsv1 = 0;                                    \
    (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;    \
    (gate).gd_s = 0;                                    \
    (gate).gd_dpl = (dpl);                                \
    (gate).gd_p = 1;                                    \
    (gate).gd_off_31_16 = (uint32_t)(off) >> 16;        \
}
```
如果在其他代码段中有如下语句，
```
unsigned intr;
intr=8;
SETGATE(intr, 1,2,3,0);
```
请问执行上述指令后， intr的值是多少？  
答：  
3 & 0xffff = **00000011**  
gd\_off\_15\_0为16位 **00000000**  
sel = **00000010**  
gd\_ss为16位 **00000000**  
gd\_args为5位 **00000**  
gd\_rsv1为3位 **000**  
（istrap） ？ STS\_TG32 : STS\_IG32 = **1111**  
gd\_s为1位 **0**  
gd\_dpl = (dpl) = **00**  
gd\_p为1位 **1**  
(uint32\_t)（off） >> 16 = **00000000 00000000**  
最终结果内存单元从低地址起为： 00000011 00000000 00000010 00000000 00000000 11110001 00000000 00000000

### 课堂实践练习

#### 练习一

1. 请在ucore中找一段你认为难度适当的AT&T格式X86汇编代码，尝试解释其含义。  
如下代码：  

```
	.include "defines.h"
    .data
    hello:
      .string "hello world\n"

    .globl	main
    main:
      movl	$SYS_write,%eax
      movl	$STDOUT,%ebx
      movl	$hello,%ecx
      movl	$12,%edx
      int	$0x80

      ret
```  

  答：首先包含了头文件defines.h，内含各种常量定义；数据段定义了hello字符串值为"hello world\n"。代码段主函数将系统调用$SYS_write放置于%eax寄存器，将输出目标$STDOUT放置在%ebx寄存器，将输出对象$hello字符串放置在%ecx寄存器，将$hello字符串长度12放置在%edx寄存器，接着执行系统调用$0x80，将hello字符串内容输出到stdout，最后返回。

练习二
2. (option)请在rcore中找一段你认为难度适当的RV汇编代码，尝试解释其含义。  


#### 练习二

宏定义和引用在内核代码中很常用。请枚举ucore或rcore中宏定义的用途，并举例描述其含义。  
答：利用宏进行复杂数据结构中的数据访问； 利用宏进行数据类型转换；如 to_struct; 常用功能的代码片段优化；如 ROUNDDOWN。

#### reference
 - [Intel格式和AT&T格式汇编区别](http://www.cnblogs.com/hdk1993/p/4820353.html)
 - [x86汇编指令集  ](http://hiyyp1234.blog.163.com/blog/static/67786373200981811422948/)
 - [PC Assembly Language, Paul A. Carter, November 2003.](https://pdos.csail.mit.edu/6.828/2016/readings/pcasm-book.pdf)
 - [*Intel 80386 Programmer's Reference Manual*, 1987](https://pdos.csail.mit.edu/6.828/2016/readings/i386/toc.htm)
 - [IA-32 Intel Architecture Software Developer's Manuals](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
