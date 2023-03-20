## OS lab1
**建议配合指导书同时使用**
***
### 知识点一： ELF文件
#### 知识点理解

<font color=grey>（如果这部分已经理解，可以直接点击跳到</font>[题目部分](#题目解析11)）

 **ELF**(Executable and Linkable Format)是一种文件格式，看名字就知道是可执行文件和可链接文件的格式，关于ELF格式的详细介绍移步指导书或者stfw。  

 一个还行的[ELF文件讲解](https://blog.csdn.net/lin___/article/details/104077452?utm_source=app&app_version=5.15.0&code=app_1562916241&uLinkId=usr1mkqgl919blen)

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
        Elf32_Word sh_entsize;   /* 某些节区中包含固定大小的项目，如符号表。对于这类节区，此成员给出每个表项的长度字节数。*/
} Elf32_Shdr;
```

#### 题目解析（1.1）
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

学完了上面的ELF初步知识，会发现我们要完成的任务是很简单的：


1.获取节头表地址：即 节头表地址=ELF头地址+节头表偏移动`sh_table = binary + ehdr->e_shoff`

2.获取节头表项数：即ehdr结构体中的e_shnum成员

3.获取节头表表项大小：即ehdr结构体中的e_shentsize成员

4.按地址顺序输出每一节的节地址，注意**循环增加的变量**是const void*类型的地址值，表示一个表项的地址，按字节寻址，因此每轮的**循环增量**是节头表表项大小

>第四点注意节头表表项大小和节大小的区别。节头表项大小就是一个上述一个`Elf32_Shdr`结构体的大小（该结构体记录了一个节的有关信息），是ELF头成员`e_shentsize`的值，而节大小指节头表项结构体所记录信息的一个具体的节的大小（单位字节），是`Elf32_Shdr`中成员`sh_size`的值。另外，也要注意区分`ehdr->e_shentsize`和`shdr->sh_entsize`的区别，在上面的各头表结构代码注释中解释了区别。

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

<font color=grey>（如果这部分已经理解，可以直接点击跳到</font>[题目部分](#题目解析1213)）

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

##### （三）内核入口

Makefile文件大致结构：

首先找到第一个目标：all，其依赖文件为`$(targets)`，找到变量`$(targets)`的定义语句：`targets := $(mos-elf)`，继续顺藤摸瓜找
`$(mos_elf)`的定义语句：`mos_elf := $(target_dir)/mos`，`$(target_dir) := target`，翻译一下就是，`all:target/mos`，整个项目的依赖文件就是内核mos的ELF可执行文件。

进一步细看内核的依赖和命令：
```Makefile
 $(mos_elf): $(modules) $(target_dir)
        $(LD) $(LDFLAGS) -o $(mos_elf) -N -T $(link_script) $(objects)
```
寻找命令中的各变量，容易发现在Makefile第45 46行：
```Makefile
 include mk/tests.mk mk/profiles.mk
 export CC CFLAGS LD LDFLAGS lab
```
在相应文件中可以找到`$(LD)`的定义`mips-linux-gnu-ld`：这是一条交叉编译工具的链接命令，上面提到，使用`-T`选项可以指定使用的链接脚本，这条命令指定了我们下面需要修改的`kernel.lds`链接脚本。打开链接脚本，第九行的`ENTRY(_start)`给出了程序的入口地址。

关于这一点，指导书给出了说明：关于链接后的程序从何处开始执行。程序执行的第一条指令的地址称为入口地址（entrypoint）。我们的实验就在 kernel.lds 中通过 ENTRY(_start) 来设置程序入口为_start。

#### 题目解析(1.2&1.3)

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
>指导书exercise1.3题目描述：

>完成 init/start.S 中空缺的部分。设置栈指针，跳转到 mips_init 函数。执行 gxemul -E testmips -C R3000 -M 64 target/mos 运行仿真，其中 target/mos是我们构建生成的内核 ELF 镜像文件的路径。在我们的实验中，也可以使用命令 make run 来运行 GXemul，或使用命令 make dbg在调试模式下运行 GXemul。

给出的代码框架：
```c
#include <asm/asm.h>
#include <mmu.h>

.text
EXPORT(_start)
.set at
.set reorder
        /* 禁止外部中断 */
        mtc0    zero, CP0_STATUS

        /* hint: 可以参考include/mmu.h中的内存布局 */
        /* 设置内核栈指针 */
        /* Exercise 1.3: Your code here. (1/2) */


        /* 跳转到mips_init */
        /* Exercise 1.3: Your code here. (2/2) */

```

两个任务：
1.将sp寄存器设置到内存栈空间上（mips汇编`la` 或`li` 命令）
2.跳转到mips_init函数（mips汇编`j`命令）

给出需要补全的代码：
```c
#1
la sp, 0x80400000
#2
j mips_init
```

### 知识点三：实战printk

<font color=grey>（点击直接跳到[题目解析](#题目解析14)）</font>

这个部分我们先分析一下lab1分支下的代码结构。

1.首先是分支目录下的几个非目录文件：

* **`Makefile`** 构建内核的总Makefile文件，前面我们已经大致分析过

* **`kernel.lds`** Linker Script链接脚本，前面我们用来控制链接和装载mos_elf相应节在内存中的位置

* **`include.mk`** Makefile里面一些用到的变量（如`$(CC)` `$(LD)`）在这里进行了定义

2.再看比较熟悉的目录：

* **`target`** mos可执行文件所在目录，也就是执行`make`命令后生成的目标文件

* **`tools`** 工具目录，例如我们第一题完成的`readelf`

* **`init`** 顾名思义，是和内核初始化有关的目录。进入目录，发现Makefile构建的目标`init.o`和`start.o`，不难猜出这就是和初始化有关的目标文件。`start.o`的依赖文件`start.S`正是第三题中补全过的汇编文件，它是整个内核的程序入口。而init.c中的`mips_init()`函数就是`start.S`中我们填写的`j`指令跳转的函数了。

3.最后逐个分析其他目录：

* **`kern`** 先看其Makefile，总目标是`include.mk`中定义的三个目标文件`console.o` `printk.o` `panic.o`，我们可以从其依赖文件`printk.c`和`console.c`尝试阅读，当然，这里暂时读不懂也无大碍，介绍完目录后我们再来分析这些`.c`文件中函数调用关系

* **`lib`** 库目录，Makefile的目标是`elfloader.o` `print.o` `string.o`，大致猜测这是一些库函数的实现。

* **`mk`** 内含两个暂时用不着细看的`.mk`文件

现在着眼于`printk.c`，分析相关c文件中的函数调用。

在`printk.c`文件中，我们找到`printk`函数所在处：
```c
 1 void printk(const char *fmt, ...) {
 2        va_list ap;
 3        va_start(ap, fmt);
 4        vprintfmt(outputk, NULL, fmt, ap);
 5        va_end(ap);
 6 }
```
这是一个**变长参数**的函数。我们暂时不考虑变长参数的代码细节，看到第3、4行，显然它对可变参数做了初步处理然后传给了这个`vprintfmt()`函数。也就是说，`printk()`函数中没有实现其功能的逻辑代码，而是调用了这个`vprintfmt`函数来实现逻辑代码。

那么我们来到包含这个函数的文件中（找不到可以使用grep -r在目录下递归寻找字符串），发现在`lib`目录中的`print.c`文件中。找到`vprintfmt`函数中和输出有关的部分，发现它实际上调用了`print_num`等函数，进一步找到`print_num`等函数，发现这些函数的参数表中第一个参数都是一个叫`out`的函数，实际输出也是由`out`函数来完成的。

那么，回到`kern/prink.c`调用`vprintfmt`处，发现这个传入`'out'`处的函数叫做`outputk`，其定义就在`printk.c`上方。这个`outputk`函数又调用了一个名为`printcharc`的函数。再次查找，发现这个函数原型在`kern/console.c`中定义（如下）。至此，我们从上层至下层顺藤摸瓜，终于找到了`prink`函数的底层原理：往控制台输出字符，原来最终就是这个`printcharc`往内存中的某一块位置放置字符。

```c
//console.c 
//只给出了printcharc函数代码
#include <drivers/dev_cons.h>
#include <mmu.h>

void printcharc(char ch) {
        *((volatile char *)(KSEG1 + DEV_CONS_ADDRESS + DEV_CONS_PUTGETCHAR)) = ch; 
        //往内存中某块位置放了一个字符ch，就达到了向控制台输出的目的
}
```

这些代码框架基本已经为我们搭建好，最后一题只需要补全`vprintfmt`中的一点格式化输出的逻辑即可。

#### 题目解析（1.4）

>指导书exercise1.4题目描述：

>阅读相关代码和下面对于函数规格的说明，补全 lib/print.c 中 vprintfmt()函数中两处缺失的部分来实现字符输出。第一处缺失部分：找到% 并分析输出格式; 第二处缺失部分：取出参数，输出格式串为 %[flags][width][length]<specifier> 的情况。具体格式详见printk 格式具体说明。

`vprintfmt`函数代码框架：

```c
void vprintfmt(fmt_callback_t out, void *data, const char *fmt, va_list ap) {
        char c;
        const char *s;
        long num;

        int width;
        int long_flag; // 输出为long则为1，否则int，为0
        int neg_flag;  // 输出是负数
        int ladjust;   // 输出是左对齐的则为1（width>输出长度时默认是右对齐）
        char padc;     // width大于输出长度时空格填充字符（默认为空格）

        for (;;) {
                /* scan for the next '%' */
                /* Exercise 1.4: Your code here. (1/8) */
                
                /* flush the string found so far */
                /* Exercise 1.4: Your code here. (2/8) */

                /* check "are we hitting the end?" */
                /* Exercise 1.4: Your code here. (3/8) */

                /* we found a '%' */
                /* Exercise 1.4: Your code here. (4/8) */

                /* check format flag */
                /* Exercise 1.4: Your code here. (5/8) */
                
                /* get width */
                /* Exercise 1.4: Your code here. (6/8) */
                
                /* check for long */
                /* Exercise 1.4: Your code here. (7/8) */
                
                neg_flag = 0;
                switch (*fmt) {
                case 'b':
                        if (long_flag) {
                                num = va_arg(ap, long int);
                        } else {
                                num = va_arg(ap, int);
                        }
                        print_num(out, data, num, 2, 0, width, ladjust, padc, 0);
                        break;

                case 'd':
                case 'D':
                        if (long_flag) {
                                num = va_arg(ap, long int);
                        } else {
                                num = va_arg(ap, int);
                        }

                        /*
                         * Refer to other parts (case 'b', case 'o', etc.) and func 'print_num' to
                         * complete this part. Think the differences between case 'd' and the
                         * others. (hint: 'neg_flag').
                         */
                        /* Exercise 1.4: Your code here. (8/8) */
                        
                        break;

                case 'o':
                case 'O':
                        if (long_flag) {
                                num = va_arg(ap, long int);
                        } else {
                                num = va_arg(ap, int);
                        }
                        print_num(out, data, num, 8, 0, width, ladjust, padc, 0);
                        break;

                case 'u':
                case 'U':
                        if (long_flag) {
                                num = va_arg(ap, long int);
                        } else {
                                num = va_arg(ap, int);
                        }
                        print_num(out, data, num, 10, 0, width, ladjust, padc, 0);
                        break;

                case 'x':
                        if (long_flag) {
                                num = va_arg(ap, long int);
                        } else {
                                num = va_arg(ap, int);
                        }
                        print_num(out, data, num, 16, 0, width, ladjust, padc, 0);
                        break;

                case 'X':
                        if (long_flag) {
                                num = va_arg(ap, long int);
                        } else {
                                num = va_arg(ap, int);
                        }
                        print_num(out, data, num, 16, 0, width, ladjust, padc, 1);
                        break;

                case 'c':
                        c = (char)va_arg(ap, int);
                        print_char(out, data, c, width, ladjust);
                        break;

                case 's':
                        s = (char *)va_arg(ap, char *);
                        print_str(out, data, s, width, ladjust);
                        break;

                case '\0':
                        fmt--;
                        break;

                default:
                        /* output this char as it is */
                        out(data, fmt, 1);
                }
                fmt++;
        }
}
```

基本逻辑也很简单：

非格式字符由`switch-case`语句之前的逻辑来控制，注意没有到达格式字符（`'%'`后的部分）是不会进入`case`语块的，**不要被`case '\0'`和`default`中的语句误导**。遇到`'%'`后，需要输出格式字符，这个时候才能交由`switch-case`语句来控制输出逻辑。至于格式输出的实现细节不需要我们关注，因此工作难度也就大大降低了。

第八处补全实际上就是要我们来考虑负数情况的输出，这里很容易出现测试的时候发现本来应该是个正常的负数，自己却输出了一个很大的正数，不过学过计组的我们一看就知道这是因为负数直接用补码传入`print_num`函数，而它却不认识补码（完全按二进制位处理的），我们只能手动转换一次，把负数转换成其绝对值，然后由`print_num`等函数在判断是负数的语块中给它前面补个负号。这也是变量`neg_flag`的必须性。

下面是参考代码：

```c
//1
if(*fmt != '%' && *fmt != '\0'){ //既未结束也不是%
        out(data,fmt,1);
        fmt ++;
        continue;
}
//2

//3
if(*fmt == '\0'){
        fmt --;
        break;
}
//4
if(*fmt == '%'){
        fmt ++;
}
//5
ladjust = 0;
padc = ' ';
if(*fmt == '-'){
        ladjust = 1;
        fmt ++;
}else if(*fmt == '0'){
        padc = '0';
        fmt ++;
}
//6
width = 0;
while(*fmt >= '0' && *fmt <= '9'){
        width = width * 10 + (*fmt-48);
        fmt ++;
}
//7
long_flag = 0;
if(*fmt == 'l'){
        long_flag = 1;
        fmt ++;
}
//8
if(num < 0){
        num --;
        num = ~num;
        neg_flag = 1;
}
print_num(out, data, num, 10, neg_flag, width, ladjust, padc, 0);
```
第二处笔者提交时就空着，猜测和缓冲区有关，但是想不出缺少了它有什么逻辑漏洞，而且测评也过了。后续弄清楚后笔者会回来详细解释。

至此lab1全部题解和相应知识点结束了，加油。







