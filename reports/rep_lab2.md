## OS Lab1 实验报告

21373369 熊前锋

***

### 一、思考题（按指导书中顺序）

#### Thinking 2.1

C程序中指针变量存储的是虚拟地址。

MIPS汇编中`lw`和`sw`指令使用的是虚拟地址。

#### Thinking 2.2

（1）用宏实现链表，可以避免重复劳动，也可以提高代码的效率 ，减少出错。C语言中没有多态，那么就会出现以下的情况：如果在一个工程中想要同时使用整型的链表和浮点型的链表，就要定义两个结构体，并且同样的操作节点的代码要写两遍；若在C语言中借助void*指针，利用强制类型转换的方法实现多态，可能会出现无法确定数据存储位置而导致出错的情况。

（2）SLIST为单向无尾链表，删除与插入操作与普通的链表操作相同。进行插入操作时只需要将新插入节点的指针（field.sle_next）指向要插入的位置的下一个节点 ，并将要插入的上一个节点的指针（field.sle_next）指向它即可；删除将上一个节点的指针指向下待删除节点的下一个节点，再释放空间即可。

STAILQ为单向有尾链表。它的删除与插入操作相比SLIST多了对尾部的操作，在处理时多了if是尾部的判断，其余基本相同。

CIRCLEQ为循环链表，其最后一个节点的链表指针指向头节点。

三者相比，单向链表和循环链表在插入和删除节点时操作较双向链表简单，但是不能通过节点直接访问其前驱节点；双向链表在空间上比单向和循环链表多了一个指针，插入删除操作也相对复杂，但是可以直接访问某一节点的前驱节点。

#### Thinking 2.3

正确的展开选项是C。

#### Thinking 2.4

（1）在虚拟内存的实现中，每个进程都有自己的虚拟地址空间，这些地址空间可以通过页表映射到物理内存中的物理页面。ASID的作用是将每个进程的虚拟地址空间与其对应的页表关联起来。当进程执行时，硬件会根据其ASID来查找其对应的页表，从而将进程的虚拟地址转换为物理地址。

（2）R3000处理器中的ASID字段长度为6位，因此可以支持2\^6即（64）个不同的ASID。每个ASID对应一个不同的地址空间，因此最多可以管理64个不同进程的虚拟地址空间，每个进程的最大地址空间大小为2\^32（即4GB）。即R3000可以管理最大为256GB的虚拟地址空间。

#### Thinking 2.5

（1）tlb_out 被 tlb_invalidate 调用。

（2）tlb_invalidate函数用于清空TLB中的某个或某些条目。

（3）如下，在每行用注释解释。

``` c
 .set noreorder
         mfc0    t0, CP0_ENTRYHI #将寄存器CP0_ENTRYHI的内容读取到寄存器t0中
         mtc0    a0, CP0_ENTRYHI #将Key写入CP0_ENTRYHI（Key通过参数a0传入）
         nop #插入一个 nop 以解决数据冒险
         /* Step 1: Use 'tlbp' to probe TLB entry */
         /* Exercise 2.8: Your code here. (1/2) */
         tlbp #根据 EntryHi 中的 Key（包含 VPN 与 ASID），查找 TLB 中与之对应的表项，并将表项的索引存入 Index 寄存器
         nop #插入一个 nop 以解决数据冒险
         /* Step 2: Fetch the probe result from CP0.Index */
         mfc0    t1, CP0_INDEX #将寄存器CP0_INDEX的内容读取到寄存器t1中
 .set reorder
         bltz    t1, NO_SUCH_ENTRY #分支指令，如果t1小于零，则跳转到标签NO_SUCH_ENTRY处
 .set noreorder
         mtc0    zero, CP0_ENTRYHI #将寄存器CP0_ENTRYHI赋0
         mtc0    zero, CP0_ENTRYLO0 #将寄存器CP0_ENTRYLO赋0
         nop #插入一个 nop 以解决数据冒险
         /* Step 3: Use 'tlbwi' to write CP0.EntryHi/Lo into TLB at CP0.Index  */
         /* Exercise 2.8: Your code here. (2/2) */
         tlbwi #使用tlbwi指令将新的TLB条目写入到指定的TLB位置
 .set reorder
 
 NO_SUCH_ENTRY:
         mtc0    t0, CP0_ENTRYHI #将之前保存的EntryHi寄存器的值写入到CP0寄存器中的EntryHi寄存器中  
         j       ra #使用j ra指令返回到调用该函数的位置
 END(tlb_out)
```

#### Thinking 2.6

（2）RISC-V 与 MIPS 在内存管理上的区别：

RISC-V和MIPS都是采用虚拟内存管理机制，其内存管理机制的基本原理类似，都是通过页表和TLB来实现虚拟地址到物理地址的映射。其实现的具体区别包括（但不限于）：

1.虚拟地址的位数不同：RISC-V中虚拟地址的位数为39位，可以支持512TB的虚拟地址空间；而MIPS中虚拟地址的位数为32位，可以支持4GB的虚拟地址空间。

2.页表项大小的不同：在RISC-V中，一个页表项大小为64位；而MIPS中一个页表项大小为32位。

3.MIPS中的TLB具有固定的条目数：MIPS中的TLB只有32个固定的条目，而RISC-V中的TLB大小是可变的，可以根据需要配置大小。

### 二、难点分析

* 第一处难点是`page_alloc`函数。倒不是说有多难，是我觉得第一个“坑点”。思路上很简单，就是调用链表宏和memset函数来弄出一个物理页管理块，给和它挂钩的虚拟地址为起点分配一页大小的空间。坑的是，错误返回值`-E_NO_MEM`指导书中真的好像没提到啊……开始以为默认是-1，debug不决，最后跑到assertion出错的地方发现检查错误返回值用了`-E_NO_MEM`这个宏。可能是我自己不仔细？

* 第二处难点在虚拟页管理部分的`pgdir_walk`。主要是要理解，这个函数不单是检查有没有映射的，实际上传参`create`如果有效，相应的二级页表也是由这个函数来完成的，`page_insert`实际是调用了带参数`create`=1的`pgdir_walk`来完成这一工作。另外我自己开始写的时候，犯了一个错误：最后一步直接用`page2kva(pp)`得到页表项的虚拟地址，赋值给`*ppte`，这样是不对的：据上述逻辑流程，并不是所有的条件分支中页控制块`pp`都起了作用。首先如果这个`va`是有效的，最外层的`if`根本不会进入，其次，参数`create`为0时，也不会进行页表的空间分配。

* 第三处难点是`_do_tlb_refill`函数。主要就是不了解这个`passive_alloc`的调用背景，开始没看到将`cur_pgdir`用作全局变量的页目录基址来调用映射检查函数`page_lookup`。

其他还好，物理页面的空闲链表管理方法经过了简化，好理解多了；二级页表偶尔有点绕，搞清楚各个函数的本职工作是什么以后，理解起来也不难了。

### 三、实验体会

最大的体会就是计组真是个好东西。计组的知识极大地帮助我理解了本次实验设计的知识，包括虚拟地址空间、虚拟地址到物理地址的映射（因为已经对一级页表有一定的了解，再学起二级页表也就不那么难了）、TLB等。

另外，感觉自己也可以尝试写一些测试函数来帮助自己理解这个内核的一些设计，以及完成函数以后的功能实测。比如，在对从物理页面到其管理块的理解还有些迷茫时，为了理解函数`pa2page`、`page2pa`、`page2kva`等，我自己尝试写了一个测试函数并在`tests`的`init.c`中调用：

```c
void my_test_PageToPP(){
        for(int i=0;i<npage;i++){
                printk("page index %d: sP_addr=0x%x in pa 0x%x,pp_addr=0x%x\n",i,pages+i,PADDR(pages+i),page2pa(pages+i));
                if(i==9){
                        printk("...\n...\n...\n");
                        i=npage-11;
                }
        }
}
```

运行：

```c
init.c: mips_init() is called
Memory size: 65536 KiB, number of pages: 16384
to memory 80430000 for struct Pages.
pmap.c:  mips vm init success
page index 0: sP_addr=0x80400000 in pa 0x400000,pp_addr=0x0
page index 1: sP_addr=0x8040000c in pa 0x40000c,pp_addr=0x1000
page index 2: sP_addr=0x80400018 in pa 0x400018,pp_addr=0x2000
page index 3: sP_addr=0x80400024 in pa 0x400024,pp_addr=0x3000
page index 4: sP_addr=0x80400030 in pa 0x400030,pp_addr=0x4000
page index 5: sP_addr=0x8040003c in pa 0x40003c,pp_addr=0x5000
page index 6: sP_addr=0x80400048 in pa 0x400048,pp_addr=0x6000
page index 7: sP_addr=0x80400054 in pa 0x400054,pp_addr=0x7000
page index 8: sP_addr=0x80400060 in pa 0x400060,pp_addr=0x8000
page index 9: sP_addr=0x8040006c in pa 0x40006c,pp_addr=0x9000
...
...
...
page index 16374: sP_addr=0x8042ff88 in pa 0x42ff88,pp_addr=0x3ff6000
page index 16375: sP_addr=0x8042ff94 in pa 0x42ff94,pp_addr=0x3ff7000
page index 16376: sP_addr=0x8042ffa0 in pa 0x42ffa0,pp_addr=0x3ff8000
page index 16377: sP_addr=0x8042ffac in pa 0x42ffac,pp_addr=0x3ff9000
page index 16378: sP_addr=0x8042ffb8 in pa 0x42ffb8,pp_addr=0x3ffa000
page index 16379: sP_addr=0x8042ffc4 in pa 0x42ffc4,pp_addr=0x3ffb000
page index 16380: sP_addr=0x8042ffd0 in pa 0x42ffd0,pp_addr=0x3ffc000
page index 16381: sP_addr=0x8042ffdc in pa 0x42ffdc,pp_addr=0x3ffd000
page index 16382: sP_addr=0x8042ffe8 in pa 0x42ffe8,pp_addr=0x3ffe000
page index 16383: sP_addr=0x8042fff4 in pa 0x42fff4,pp_addr=0x3fff000
```

这样探测出了所有物理页面及其管理块pp和pp所在的页面，从而对整个物理内存的管理有了比较细致的了解。

总之，如果上机顺利的话（），这真是一次不错的lab体验。
