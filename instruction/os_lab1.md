## OS lab1
**建议配合指导书同时使用**
***
### 知识点一： ELF文件
#### 知识点理解
 **ELF**(Executable and Linkable Format)是一种文件格式，看名字就知道是可执行文件和可链接文件的格式，关于ELF格式的详细介绍移步指导书或者stfw。  

 一个还行的[ELF文件初步讲解](https://blog.csdn.net/daide2012/article/details/73065204)

 >可以使用file filename命令来查看文件类型

 ELF的内容：和程序相关的所有必要内容。
 下图展示了ELF格式的内容：
 ![graph1](/img/ELF.png)
 从左边看，是“节”（section）的视角，从右边看，是“段”（segment）的视角。节和段并不是ELF中不同的两块区域，它们只是对ELF内容不同尺度的划分，或者说不同视角的分类。

 **链接视图**：从左边看，它为汇编器和链接器提供了解析程序的视角：汇编器和链接器从节的角度来解析文件，链接视图可以没有段表。

 **执行视图**：从右边看，它为加载器（动态链接器）提供了视角：加载器从段的角度来解析文件，执行视图可以没有节表。

关于ELF头、段头表（或程序头表Program Header Table）、节头表（Section Header Table）：

<font color=red>ELF头</font> 一个结构体，包含了程序的基本信息，比如体系结构和操作系统，还包含了**段头表和节头表**的相对偏移offset（ELF头位于文件头部，相对于ELF头的偏移）
```c
//ELF头的内容：
typedef struct {
        unsigned char e_ident[EI_NIDENT]; /* 魔数Magic Number和其他信息*/
        Elf32_Half e_type;                /* 文件类型*/
        Elf32_Half e_machine;             /* 体系结构*/
        Elf32_Word e_version;             /* 文件版本*/
        Elf32_Addr e_entry;               /* 入口点虚拟地址*/
        Elf32_Off e_phoff;                /* 段头表偏移 */
        Elf32_Off e_shoff;                /* 节头表偏移 */
        Elf32_Word e_flags;               /* Processor-specific flags */
        Elf32_Half e_ehsize;              /* ELF头大小（字节）*/
        Elf32_Half e_phentsize;           /* 段头表表项大小*/
        Elf32_Half e_phnum;               /* 段头表项数（段数）*/
        Elf32_Half e_shentsize;           /* 节头表表项大小*/
        Elf32_Half e_shnum;               /* 节头表项数（节数）*/
        Elf32_Half e_shstrndx;            /* 节头字符串编号 */
} Elf32_Ehdr;
```

<font color=red>段头表</font> 相当于一个结构体数组，每一项就记录一个段的信息。
```c
//段头表一个表项的内容
typedef struct {
        Elf32_Word p_type;   /* 段类型*/
        Elf32_Off p_offset;  /* 段偏移*/
        Elf32_Addr p_vaddr;  /* 段虚拟地址*/
        Elf32_Addr p_paddr;  /* 段物理地址*/
        Elf32_Word p_filesz; /* 在文件中段所占大小*/
        Elf32_Word p_memsz;  /* 内存中段所占大小*/
        Elf32_Word p_flags;  /* Segment flags */
        Elf32_Word p_align;  /* Segment alignment */
} Elf32_Phdr;
```

<font color=red>节头表</font> 相当于一个结构体数组，每一项旧纪录一个节的信息。
```c
//节头表一个表项的内容
typedef struct {
        Elf32_Word sh_name;      /* 节名称*/
        Elf32_Word sh_type;      /* 节类型*/
        Elf32_Word sh_flags;     /* Section flags */
        Elf32_Addr sh_addr;      /* 节地址*/
        Elf32_Off sh_offset;     /* 节偏移*/
        Elf32_Word sh_size;      /* 节大小*/
        Elf32_Word sh_link;      /* Section link */
        Elf32_Word sh_info;      /* Section extra info */
        Elf32_Word sh_addralign; /* Section alignment */
        Elf32_Word sh_entsize;   /* 节头表表项大小*/
} Elf32_Shdr;
```

#### 题目解析
>指导书exercise1.1题目描述：

>阅读 tools/readelf 目录下的 elf.h、readelf.c 和 main.c 文件，并补全readelf.c 中缺少的代码。readelf 函数需要输出 ELF 文件中所有节头中的地址信息，对于每个节头，输出格式为"%d:0x%x\n"，其中的 %d 和 %x 分别代表序号和地址。正确完成 readelf.c 之后,在 tools/readelf 目录下执行 make 命令，即可生成可执行文件readelf，它接受文件名作为参数，对 ELF 文件进行解析。可以执行 make hello生成测试用的 ELF 文件 hello，然后运行 ./readelf hello 来测试 readelf。 

简单地说就是把ELF文件中所有节的地址信息按行打印，即上述节头表中每一项的sh_addr成员。

先看给出的代码框架（删去了英文注释，替换成了中文简洁版）：
```c
//readelf.c
#include "elf.h"
#include <stdio.h>

//此函数用来判断是否是ELF格式
int is_elf_format(const void *binary, size_t size) {
        Elf32_Ehdr *ehdr = (Elf32_Ehdr *)binary;
        return size >= sizeof(Elf32_Ehdr) && ehdr->e_ident[EI_MAG0] == ELFMAG0 &&
               ehdr->e_ident[EI_MAG1] == ELFMAG1 && ehdr->e_ident[EI_MAG2] == ELFMAG2 &&
               ehdr->e_ident[EI_MAG3] == ELFMAG3;
}

//待完成的核心函数：参数binary是ELF文件的首地址，该地址必须有效
int readelf(const void *binary, size_t size) {
        Elf32_Ehdr *ehdr = (Elf32_Ehdr *)binary; //ELF头结构体指针

        // 检查binary为首地址的空间是否是ELF文件
        if (!is_elf_format(binary, size)) {
                fputs("not an elf file\n", stderr);
                return -1;
        }

        //获取节头表首地址，节头表表项数，表项大小
        const void *sh_table; //节头表指针
        Elf32_Half sh_entry_count;
        Elf32_Half sh_entry_size;
        /* Exercise 1.1: Your code here. (1/2) */


        //对每一节，输出索引和节地址。索引从0开始
        for (int i = 0; i < sh_entry_count; i++) {
                const Elf32_Shdr *shdr;
                unsigned int addr;
                /* Exercise 1.1: Your code here. (2/2) */

        }

        return 0;
}
```

学完了上面的ELF初步理论，会发现我们要完成的任务是很简单的：


1.获取节头表地址：即 节头表地址=ELF头地址+节头表偏移动`sh_table = binary + ehdr->e_shoff`

2.获取节头表项数：即ehdr结构体中的e_shnum成员

3.获取节头表表项大小：即ehdr结构体中的e_shentsize成员

4.按地址顺序输出每一节的节地址，注意**循环增加的变量**是const void*类型的地址值，表示一个表项的地址，按字节寻址，因此每轮的**循环增量**是节头表表项大小

>第四点注意节头表表项大小和节大小的区别。

综上，给出两个部分填空代码：
```c
//1
sh_table = binary + ehdr -> e_shoff; //节头表第一项地址 = ELF头地址 + 节头表偏移
sh_entry_count = ehdr -> e_shnum;
sh_entry_size = ehdr -> e_shentsize;
```
```c
//2
shdr = (Elf32_Shdr *)sh_table;
addr = shdr -> sh_addr;
sh_table += sh_entry_size;
printf("%d:0x%x\n", i, addr);
```

保存退出，在目录下执行make命令，得到输出结果与执行readelf -S hello的节头表内容进行比对，发现地址一致。
 
### 知识点二： MIPS内存布局和内核装载
#### 知识点理解
##### （一）MIPS内存布局和内存装载的位置
由计组的知识我们知道，程序中使用的地址和真正访存的地址是不同的，前者是虚拟地址，后者是物理地址，由虚拟地址到物理地址的转换是由处理器中的硬件MMU(Memory Management Unit)来完成的，而MMU是受操作系统控制的（似乎又回到了内核装载的鸡生蛋问题）。对于32位处理器而言，其虚拟地址空间一般为4GB，布局如图：
![mips_vertual_addr](/img/mips%E8%99%9A%E6%8B%9F%E5%9C%B0%E5%9D%80%E7%A9%BA%E9%97%B4.png)

由低到高分别是：

**kuseg** 用户态下唯一可用的内存空间。即mips规定的用户内存空间，需要通过MMU TLB进行虚拟地址到物理地址的变换。对这段地址的存取都会通过cache。

**kseg0** 内核态可用地址，特点是这段虚拟地址直接连续映射到物理地址低512MB，存取通过cache。

**kseg1** 内核态可用地址，特点是这段虚拟地址也直接连续映射到物理地址低512MB，但存取不通过cache，使用MMIO访问外设。

**kseg2** 内核态可用地址，需要通过MMU TLB进行虚拟地址到物理地址的变换，存取通过cache。

<font color=red>简单地说，kuseg是用户态内存，其他都是内核态；kseg0和kseg1不需要用到MMU进行映射内存而另外两个要；kseg1的访存不经过cache而其他都经过。</font>

这里直接粘贴指导书原话（指导书中为数不多的人话）：

>MMU需要操作系统进行配置管理，因此在载入内核时，我们不能选用需要通过 MMU 转换的虚拟地址空间。这样内核就只能放在 kseg0 或 kseg1 了。而 kseg1 是不经过 cache 的，一般来说，利用 MMIO 访问外设时才会使用 kseg1。**因此我们将内核的 .text、.data、.bss 段都放到 kseg0 中**。至于上文提到 kseg0 在 cache 未初始化前不能使用，但这里我们又将内核放在kseg0 段，这是因为在真实的系统中，运行在 kseg1 中的 bootloader 在载入内核前会进行 cache初始化工作。

##### （二）Linker Script链接脚本：修改加载位置

再来一句指导书中的内容：**Linker Script 中记录了各个节应该如何映射到段，以及各个段应该被加载到的位置**。

我们知道，编译器在生成 ELF 文件时就已经记录了各节**所需要**被加载到的位置，最终的可执行文件是由链接器链接生成的，也就是说，一般程序被加载的位置是确定的（或者说默认的）可以通过命令`ld --verbose`来查看默认的链接脚本，不过这个大概率是看不懂的，但看看也无妨：仔细看就是除了一些相关信息，主体是一个`SECTIONS`开头的一个代码块：
```c
SECTIONS
{
        ...
        ...
}
```
事实上也就是这个代码块来决定将ELF文件中的各节装载到内存的哪些位置。链接脚本文件后缀为`.lds`，我们可以在编译时加上`-T`选项来指定连接脚本，以达到最终指定ELF文件特定段被加载到指定程序空间的效果。

这两个部分结合到一起，就是第二题需要做的事情：将内核调整到内存空间中的kseg0。

#### 题目解析

>指导书exercise1.2题目描述：

>填写 kernel.lds 中空缺的部分，在 Lab1 中，只需要填补.text、.data和.bss 节，将内核调整到正确的位置上即可。 

先看代码框架，老样子翻译并简化过注释：
```c
/*
 * 设置为mips体系结构
 */
OUTPUT_ARCH(mips)

/*
 * 程序入口起点设置为 _start
 * 关于这一点，指导书给出了说明：
 * 关于链接后的程序从何处开始执行。
 * 程序执行的第一条指令的地址称为入口地址（entrypoint）。
 * 我们的实验就在 kernel.lds 中通过 ENTRY(_start) 来设置程序入口为_start。
 */
ENTRY(_start)

SECTIONS {
        /* Exercise 3.10的代码，忽略 */

        
        /*将text data bss节填入正确内存地址 */
        /* Exercise 1.2: Your code here. */

        bss_end = .;
        . = 0x80400000;
        end = . ;
}
```

首先我们在 include/mmu.h 里查看MOS操作系统内核完整的内存布局图：

```c
//只展示了mmu.h的内存布局注释内容
/*
 o     4G ----------->  +----------------------------+------------0x100000000
 o                      |       ...                  |  kseg2
 o      KSEG2    -----> +----------------------------+------------0xc000 0000
 o                      |          Devices           |  kseg1
 o      KSEG1    -----> +----------------------------+------------0xa000 0000
 o                      |      Invalid Memory        |   /|\
 o                      +----------------------------+----|-------Physical Memory Max
 o                      |       ...                  |  kseg0
 o      KSTACKTOP-----> +----------------------------+----|-------0x8040 0000-------end
 o                      |       Kernel Stack         |    | KSTKSIZE            /|\
 o                      +----------------------------+----|------                |
 o                      |       Kernel Text          |    |                    PDMAP
 o      KERNBASE -----> +----------------------------+----|-------0x8001 0000    |
 o                      |      Exception Entry       |   \|/                    \|/
 o      ULIM     -----> +----------------------------+------------0x8000 0000-------
 o                      |         User VPT           |     PDMAP                /|\
 o      UVPT     -----> +----------------------------+------------0x7fc0 0000    |
 o                      |           pages            |     PDMAP                 |
 o      UPAGES   -----> +----------------------------+------------0x7f80 0000    |
 o                      |           envs             |     PDMAP                 |
 o  UTOP,UENVS   -----> +----------------------------+------------0x7f40 0000    |
 o  UXSTACKTOP -/       |     user exception stack   |     BY2PG                 |
 o                      +----------------------------+------------0x7f3f f000    |
 o                      |                            |     BY2PG                 |
 o      USTACKTOP ----> +----------------------------+------------0x7f3f e000    |
 o                      |     normal user stack      |     BY2PG                 |
 o                      +----------------------------+------------0x7f3f d000    |
 a                      |                            |                           |
 a                      ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~                           |
 a                      .                            .                           |
 a                      .                            .                         kuseg
 a                      .                            .                           |
 a                      |~~~~~~~~~~~~~~~~~~~~~~~~~~~~|                           |
 a                      |                            |                           |
 o       UTEXT   -----> +----------------------------+------------0x0040 0000    |
 o                      |      reserved for COW      |     BY2PG                 |
 o       UCOW    -----> +----------------------------+------------0x003f f000    |
 o                      |   reversed for temporary   |     BY2PG                 |
 o       UTEMP   -----> +----------------------------+------------0x003f e000    |
 o                      |       invalid memory       |                          \|/
 a     0 ------------>  +----------------------------+ ----------------------------
 o
*/
```

其中 KERNBASE 是内核镜像的起始虚拟地址。由此，我们的任务就是补全`.lds`中的代码，将`.text`节放入以KERNBASE开始的kseg0内存空间。

指导书中提供了`SECTIONS`代码块的基本写法，在其`Note 1.3.5`中指出：修改`.text节`内存位置，`.data`和`.bss`节只需紧随其后即可，这是因为...

给出需要补全部分的代码如下：
```c
. = 0x80010000;
.text : { *(.text) } 
.data : { *(.text) }
.bss : { *(.bss) }
```

***
continuing to update...








