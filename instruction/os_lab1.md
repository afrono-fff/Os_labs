## OS lab1
**建议配合指导书同时使用**
***
### 知识点一： ELF
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
        Elf32_Word sh_entsize;   /* Section entry size */
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
1.获取节头表地址：即 节头表地址=ELF头地址+节头表偏移 `sh_table = binary + ehdr->e_shoff`
2.获取节头表项数：即ehdr结构体中的e_shnum成员
3.获取节头表表项大小：即ehdr结构体中的e_shentsize成员
4.按地址顺序输出每一节的节地址，注意**循环增加的变量**是const void*类型的地址值，按字节寻址，因此每轮的**循环增量**是节头表表项大小

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
***
continuing to update...





