## OS lab2
**建议配合指导书同时使用**

***

lab1回顾：学过计组我们知道，程序中使用的地址都是虚拟地址，在lab1内核实验中我们了解到，由于MMU是受OS管理的，必须有在内核启动前就能使用的内存空间，用以放置内核，并建立内存管理体系。MIPS内存布局中，kseg0和kseg1正是这样两个内存空间，它们和物理内存之间的映射方式是直接映射，只需要简单地对高位进行赋0即可直接对应到相应的物理内存。

lab2实现了mos操作系统的内存管理。

### 知识点一：链表法管理物理内存

<font color=grey>（可以直接点击跳到</font>[题目部分](#题目解析21-25)）

#### 1.内存布局回顾

![mips-memory](/img/mips%E8%99%9A%E6%8B%9F%E5%9C%B0%E5%9D%80%E7%A9%BA%E9%97%B4.png)

上图是lab1中分析过的mips内存布局，需要明白这是32位体系结构的虚拟地址空间（2^32=4GB,按字节寻址）。在实验中，尤其需要注意区分地址是**虚拟地址**还是**物理地址**，虚拟地址布局可以参照上图，通过某些映射可以得到对应的物理地址，具体的映射函数根据规则的不同（例如直接连续映射还是通过MMU转换）会在下面一一介绍。

我们先接lab1关于此内存布局的解释进一步做一些解释：

* **kseg0**：`0x80000000`到`0x9fffffff`，在lab1中我们用来放置内核，它连续映射到物理地址的低512MB，只需要将**最高位**置0即可得到对应的物理地址（即`0x00000000`到`0x1fffffff`）。

* **kseg1**：`0xa0000000`到`0xbfffffff`，它用于访问外设，也是连续映射到物理地址的低512MB，需要将**高三位**置零得到相应物理地址，也是`0x00000000`到`0x1fffffff`。

对于mos操作系统，我们采用的管理物理内存的方式是**空闲链表**。即将物理内存分成一些大小固定的页面，作为内存调度的基本单位，再用链表将物理内存中空闲的页组织起来，需要使用时从链表中取出进行分配，使用完成后再插入链表。

*需要指出，和一般的空闲链表不同，实验中简化了空闲链表，以页为最小单位直接将**空闲的页**使用链表串联起来。

#### 2.`struck Page`结构：物理内存页管理

在lab1题解中我们提到，启动跳转到的`mips_init`函数会在不同的lab中被替换，以实现不同的评测需求。看`test`目录中`lab2_1`下的`init.c`文件中的`mips-init()`函数，首先调用了`mips_detect_memory()`，`mips_vm_init()`，`page_init()`三个函数，也就是说，内核启动跳转到`mips-init()`函数后，首先就执行了内存管理初始化的相关函数。我们从这里开始。

首先，执行`mips_detect_memory()`函数，该函数探测了当前硬件物理内存的可用大小，由此计算出物理页面数量，分别记录到全局变量`memsize`和`npage`中（记住这两个变量的意义，后面我们会用到），这样我们就对物理内存有了一个大致的掌握。

随后，开始建立内存管理机制。目前，页式管理机制还未建立起来，我们需要一些其他的数据结构来暂时管理内存。在`mips_vm_init()`函数中，我们看到功能代码只有一行：调用了一个叫alloc的类似内存分配函数，赋值给了指向`struct Page`结构的指针类型变量`pages`。这是一个结构数组的首地址，这个结构体也是链式物理内存管理的关键。

前面提到，物理内存被我们分成了许多固定大小的页面，所谓链表法当然不是直接将这些页面串联起来，**而是创建了这个结构体数组`pages`，每一个成员都是一个`struct Page`类型，它们和所有的物理页面一一对应，一个`struct Page`结构就管理一个物理页面**。`mips_vm_init()`函数完成了`struct Page`结构体数组的内存分配：

```c
void mips_vm_init() {
        pages = (struct Page *)alloc(npage * sizeof(struct Page), BY2PG, 1);
        printk("to memory %x for struct Pages.\n", freemem);
        printk("pmap.c:\t mips vm init success\n");
}
```

`alloc`是分配函数，返回分配空间的首地址（虚拟地址），`pages`于是成为`struct Page`结构数组的首地址，也是**第一个结构体的首地址**。`alloc`函数传入的第一个参数是分配空间的大小，可以看到为`npage * sizeof(struct Page)`，即`页面总数*页管理结构体大小`，也就是所有的`struct Page`所在的内存空间。

<span id="struct_Page"></span>
```c
//`struct Page`结构体具体数据结构：
typedef LIST_ENTRY(Page) Page_LIST_entry_t;
struct Page {
        Page_LIST_entry_t pp_link; 
        u_short pp_ref;
};
//其中，pp_link是包含两个指针成员的结构体，也就是用于串联链表的指针部分
//pp_ref则表示这段物理内存被引用的次数，例如pp_ref为0时表示空闲
//展开来看：
struct Page{
        struct Page_LIST_entry_t{
                struct Page *le_next;
                struct Page **le_prev;
        } pp_link;
        u_short pp_ref;
}
//为什么le_prev指针多一个'*'?参看'4.链表宏'部分
```


#### 3.从`struct Page`到物理页的地址

现在知道了用什么数据结构来管理物理页面，那么，一个特定的`struct Page`是如何和和一个特定的物理页的内存地址对应起来的呢？也就是说，我们如何通过Page结构体的信息知道它管理的是哪个物理页面呢？实验中，`include/pmap.h`文件里为我们提供了**Page结构体地址**和**物理页首地址**之间相互转换的函数：

* `static inline u_long page2pa(struct Page *pp)`：传入Page结构体地址，返回物理页首地址（物理地址）。

* `static inline struct Page *pa2page(u_long pa)`：传入物理页首地址（物理地址），返回Page结构体首地址。

* `static inline u_long page2kva(struct Page *pp)`：传入Page结构体地址，返回物理页首地址对应的虚拟地址。

<font color=red>注：`page2pa`和`page2kva`区别只在于后者在前者基础上调用了一次`KADDR`，此宏定义于`include/mmu.h`，作用是将**kseg0**区的物理地址转换成对应的虚拟地址，由于是连续映射，它的操作只是简单地将物理地址延展至32位并在最高位设置为1，和它对应的是宏`PADDR`，它将**kseg0**虚拟地址转换成物理地址，只是将最高位设置位0。这就能和我们之前所说的**kseg0**区的虚拟地址和物理地址映射解释通了。</font>

所以，假如我们现在能操作一个`struct Page`类型的结构体变量`manage_page`，就可以通过调用`page2pa(&manage_page)`得到它管理的物理页的首地址，或通过调用`page2kva(&manage_page)`得到它管理的物理页首地址对应的虚拟地址。请注意，上述我们提到的所有物理地址与虚拟地址之间的转换都是在**kseg0**段的简单连续映射，`KADDR`和`PADDR`也仅适用于kseg0区，可以前往`include/mmu.h`查看这两个宏的定义源码。

如果对于Page结构体到物理页面这一部分仍不理解，可以参考[tips](#tips)部分，一段段地详细分析了内存各部分。另外，实验中多次提到**对齐**的概念，可以参考[这篇文章](https://zhuanlan.zhihu.com/p/140063999)，在实验中使用宏`ROUND`和`ROUNDDOWN`来对齐，`alignment`值为宏`BY2PG`，其值为4096（字节），也就是一个页的大小，因此凡是按照`BY2PG`对齐的地址值其**十六进制低三位**一定都是0，即为`0x1000`的整数倍。

#### 4.链表宏

现在，我们已经有了物理页面和管理它们的结构，只需要用链表将这些结构串起来，就完成了链式物理内存的管理。为了方便地使用链表，实验将链表写成宏放在了`include/queue.h`文件中，它和我们平时使用的链表没有本质区别，且实现了双向链表功能。唯一要注意的一点就是前向指针`le_prev`并不向后向指针`le_next`那样直接指向一个Page结构体，而是指向前一个Page结构体的`pp_link`成员中的`le_next`成员，也就是说，它是一个**指向了前一个结构体的后向指针**的指针，所以它是一个二重指针。指导书中给出了不错的图示：

![link_list](/img/page_link_list.jpg)

`include/queue.h`提供了一系列好用的链表宏，建议仔细阅读，了解每一个宏的作用，好开展下面的工作。在`pmap.c`中，已经给出了全局结构变量`struct Page_list page_free_list;`，我们可以找到`struct Page_list`结构的定义处对其进行展开观察，这个藏得有点深，是使用`LIST_HEAD`宏来定义的，其成员就是一个指向Page结构体的指针`lh_head`。`LIST_HEAD`宏如下，而在`include/pmap.h`中，调用宏定义了Page_list结构体。

```c
//include/queue.h宏定义
 #define LIST_HEAD(name,type)                               \
        struct name {                                       \
                struct type *lh_first; /* first element */  \
        }
//include/pmap.c调用宏来定义Page_list结构
LIST_HEAD(Page_list, Page);
```

结合[2中Page结构的分析](#struct_Page)，可以解答指导书Thinking2.3的问题：展开Page_list结构：

```c
struct Page_list{
        struct {  //Page结构
                struct {  //Page_LIST_entry_t结构
                        struct Page *le_next; //指向Page结构的指针
                        struct Page **le_prev; //指向'指向Page结构的指针'的指针
                } pp_link;
                u_short pp_ref;
        }* lh_first;
}
```

这样，我们就可以直接使用全局变量`page_free_list`，进行一系列链表宏的操作来管理物理内存。至此第一部分就结束了，[题目](#题目解析21-25)就很简单了。

>**<span id="tips">tips</span>**：kseg0从0x80000000开始到0x9fffffff结束，其中：
内核elf文件从0x80010000开始，到0x80400000结束；（kernel.lds中的end）
pages首地址从0x80400000开始，到0x80430000，总占用196608（16进制30000）字节，共16384个结构体（即对应16384个物理页面），每个结构体12字节。
物理内存大小为65536KB=64MB=2^26B，按字节寻址，物理地址为0x00000000到0x03ffffff。可以发现物理内存其实并没有一个kseg0或kseg1那么大，连续映射的结果是有部分地址肯定超出了。
我们可以自己写一个测试函数，看看这些Page结构体和物理页地址之间的关系：如下，选择一个test中的`init.c`放入测试用函数，运行得到如下结果。其中，`sP_addr`表示**Page结构体地址**（虚拟地址），也就是`pages`每个元素的地址；`pa`表示**该结构体本身在物理内存中的物理地址**，`pp_addr`表示该结构体**对应管理的物理页的首地址**。可以看到，这些结构体所占的物理内存是从`0x400000`到`0x430000`的48个页大小的空间。

```c
//测试函数
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

```c
//运行结果
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

### 题目解析(2.1-2.5)
>指导书中exercise2.1的题目描述：

>请参考代码注释，补全 mips_detect_memory 函数。在实验中，从外设中获取了硬件可用内存大小 memsize，请你用内存大小 memsize 完成总物理页数 npage 的初始化。 

`mips_detect_memory`函数代码框架：
```c
void mips_detect_memory() {
        /* Step 1: Initialize memsize. */
        memsize = *(volatile u_int *)(KSEG1 | DEV_MP_ADDRESS | DEV_MP_MEMORY);

        /* Step 2: Calculate the corresponding 'npage' value. */
        /* Exercise 2.1: Your code here. */

        printk("Memory size: %lu KiB, number of pages: %lu\n", memsize / 1024, npage);
}
```

memsize的初始化已经由代码框架给出，我们只需要根据物理内存总量换算得到物理页数即可，即`物理内存总量/页大小=物理页数`,注意单位为字节即可。

参考代码：
```c
        npage = memsize / BY2PG;  //或4096
```

>指导书中exercise2.2的题目描述：

>完成 include/queue.h 中空缺的函数 LIST_INSERT_AFTER。其功能是将一个元素插入到已有元素之后，可以仿照 LIST_INSERT_BEFORE 函数来实现。 

照抄`LIST_INSERT_BEFORE`即可，只需注意一点点的`le_prev`二重指针问题，然后注意这是宏，结尾记得加上'\\'。

参考代码：
```c
do{                                                                                 \
        (elm)->field.le_next = (listelm)->field.le_next;                            \
        if(LIST_NEXT((listelm),field) != NULL){                                     \
                LIST_NEXT((listelm),field)->field.le_prev = &((elm)->field.le_next);\
        }                                                                           \
        LIST_NEXT((listelm),field) = (elm);                                         \
        (elm)->field.le_prev = &LIST_NEXT((listelm),field);                         \
}while(0)
```

>指导书中exercise2.3的题目描述：

>完成 page_init 函数。请按照函数中的注释提示，完成上述三个功能。此外，这里也给出一些提示：
1.使用链表初始化宏 LIST_INIT。
2.将 freemem 按照 BY2PG 进行对齐（使用 ROUND 宏为 freemem 赋值）。
3.将 freemem 以下页面对应的页控制块中的 pp_ref 标为 1。
4.将其它页面对应的页控制块中的 pp_ref 标为 0 并使用 LIST_INSERT_HEAD 将其插入空闲链表。

参考代码：
```c
void page_init(void) {
        /* Step 1: Initialize page_free_list. */
        /* Hint: Use macro `LIST_INIT` defined in include/queue.h. */
        /* Exercise 2.3: Your code here. (1/4) */
        LIST_INIT(&page_free_list);
        /* Step 2: Align `freemem` up to multiple of BY2PG. */
        /* Exercise 2.3: Your code here. (2/4) */
        freemem = ROUND(freemem,BY2PG);
        /* Step 3: Mark all memory below `freemem` as used (set `pp_ref` to 1) */
        /* Exercise 2.3: Your code here. (3/4) */
        for(int i=0;i<npage;i++){
                if(page2kva(&pages[i]) < freemem){
                        pages[i].pp_ref = 1;
                }else{
                        pages[i].pp_ref = 0;
                        LIST_INSERT_HEAD(&page_free_list,&pages[i],pp_link);
                }
        }
        /* Step 4: Mark the other memory as free. */
        /* Exercise 2.3: Your code here. (4/4) */

}
```

>指导书中exercise2.4的题目描述：

>完成 page_alloc 函数。在 page_init 函数运行完毕后，在 MOS 中如果想申请存储空间，都是通过这个函数来申请分配。该函数的逻辑简单来说，可以表述为：
1.如果空闲链表没有可用页了，返回异常返回值。
2.如果空闲链表有可用的页，取出第一页；初始化后，将该页对应的页控制块的地址放到调用者指定的地方。
填空时，你可能需要使用链表宏 LIST_EMPTY 或函数 page2kva。

比较坑的地方就是这个返回异常值。我就疑惑这里直接告诉大家异常值就是`-E_NO_MEM`宏有什么不行的吗，笔者开始写这个看他指导书没告诉以为默认-1就行，debug几十分钟，最后摸到`pmap.c`本地测评函数那里发现原来检查异常值的时候用的这个玩意儿。纯纯谜语浪费大伙时间吧。。。

参考代码：
```c
int page_alloc(struct Page **new) {
        /* Step 1: Get a page from free memory. If fails, return the error code.*/
        struct Page *pp;
        /* Exercise 2.4: Your code here. (1/2) */
        if(LIST_EMPTY(&page_free_list)){
                return -E_NO_MEM;
        }
        pp = LIST_FIRST(&page_free_list);
        LIST_REMOVE(pp, pp_link);

        /* Step 2: Initialize this page with zero.
         * Hint: use `memset`. */
        /* Exercise 2.4: Your code here. (2/2) */

        memset((void *)page2kva(pp), 0, BY2PG);
        *new = pp;
        return 0;
}
```

>指导书中exercise2.5的题目描述：

>完成 page_free 函数。
提示：使用链表宏 LIST_INSERT_HEAD，将页结构体插入空闲页结构体链表。

参考代码：
```c
void page_free(struct Page *pp) {
        assert(pp->pp_ref == 0);
        /* Just insert it into 'page_free_list'. */
        /* Exercise 2.5: Your code here. */
        LIST_INSERT_HEAD(&page_free_list,pp,pp_link);
}
```

### 知识点二：采用两级页表的虚存管理

#### 1.从一级页表到二级页表

如果你对二级页表的基本思想还不太了解，可以顺着看一下，如果已经了解，建议跳过，直接到[这里](#lab_2level)

学过计组我们大概知道，每个进程中有一个页表，用来将虚拟地址按照其格式映射成物理地址。具体来说，我们将物理内存和虚拟内存空间划分成一些相同大小的页，在本实验中，页大小是宏定义`BY2PG`，其大小为4KB，即物理地址和虚拟地址的低12位为页内偏移。对虚拟地址而言，其高20位为页索引（页表项偏移）。给定一个基址为`pg_table`的页表，这样我们就能通过页索引找到对应的页表项，该页表项内存的就是物理地址值。

现在，我们以此为基础进入主题：二级页表。详细的二级页表和多级页表理论在这里不再详述，我们仅以实验内容为例。上面我们提到，采用一级页表时，页索引（虚页号）为高20位，即有$2^{20}$个页表项。每个页表项32位4字节（参考指导书给出的设定），每个进程中装一个页表（**每个进程都有自己的地址空间**），极大地浪费了内存空间。引入二级页表的核心思想实际上就是把这些页表项再进行分页，没错，就是按照我们此前划分的页面大小4KB来给这些页表项分页，用一个“页表项的页表”来索引这些页面。也就是说，现在这些页表项被划分成很多页后就好像我们之前要访问的物理页面，我们用一个高一级的页表来以和之前相同的形式来找其中的页，进而找到具体的页表项，这样我们就只需要在原页表项所在的这一页中寻找，而不是在所有的页表项中，节省了空间并提高了映射的效率。

在实验中，有$2^{20}$个页表项，每个4字节，即页表项共占4MB大小，我们每个页4KB，恰好可以分成$(2^{22}/2^{12})=2^{10}$个页面（1K个），需要10位来进行索引，因此我们上述提到的“更高一级的页表”，即用来索引这个页表项页面的页表需要10位作为索引位，这也就是第一级页表的索引位数。再继续看这些分成1K个页面的页表项：每个页表项4B，页大小4KB，因此一页恰好是1K=1024个项，次高10位用作为页内偏移来寻找具体页表项，这也就是第二级页表的索引位数。

![2_level_pagetable](/img/2_level_pagetable.jpg)

<a id="lab_2level"></a>
指导书中的这个图就较清晰地反映了我们实验用到的二级页表结构。一级页表和二级页表所占用的空间都恰好是一页即4KB，有1024个表项。注意，图中一级页表和二级页表每一个表项都标注的是由“物理页号”和“权限位”组成，但是只有二级页表表项是我们需要从虚拟地址映射到物理地址的“物理页号”，而一级页表表项的“物理页号”，指的是它需要索引的对象——某个二级页表的物理页号。他们的共同点是，它们都确实是物理内存中一个页面的物理基址，区别是一级页表索引这1024个二级页表所在页，二级页表索引映射关系的物理页。

#### 2.映射过程

经过上面的解释，有没有发现一级页表就好像专门用来寻找二级页表的“目录”，因此为了便于区分，和指导书一样，下文中用**页目录**来指代一级页表，用**页表**来指代二级页表，可能出现的英文变量名分别为**pg_dir**和**pg_tab**。

如果一个虚拟地址`va`和一个物理地址`pa`已经建立好了映射关系，且已知页目录的基址`pg_dir`，那么我们映射的过程就应该表示为如下的步骤：

* 找到`va`的高10位，即页目录项索引，记为`pgdir_off`，则`pg_dir`+`pgdir_off`即得到具体的页目录项。我们将其记为`pgdir_entry`。

* `pgdir_entry`是一个页目录项，本质是一个32位的无符号二进制数，在实验中我们用`u_long`无符号长整形通过`typedef`定义了一个叫`Pde`的类型。第二步就是找出它的高二十位得到物理页号，这个物理页号就是页表的页号，是我们要寻找的具体页表项所在的页表基址，将其记为`pg_tab`。

* 与第一步类似，我们需要找到`va`次高10位（21~12），即页表项索引，记为`pgtab_off`，则`pg_tab`+`pgtab_off`即得到具体的页表项。我们将其记为`pgtab_entry`。

* `pgtab_entry`的物理页号（高20位），即为映射的物理页号，也就是我们需要寻找到的物理地址所在物理页的基址。通过`va`低12位页内偏OS lab2
建议配合指导书同时使用

lab1回顾：学过计组我们知道，程序中使用的地址都是虚拟地址，在lab1内核实验中我们了解到，由于MMU是受OS管理的，必须有在内核启动前就能使用的内存空间，用以放置内核，并建立内存管理体系。MIPS内存布局中，kseg0和kseg1正是这样两个内存空间，它们和物理内存之间的映射方式是直接映射，只需要简单地对高位进行赋0即可直接对应到相应的物理内存。

lab2实现了mos操作系统的内存管理。

知识点一：链表法管理物理内存
（可以直接点击跳到题目部分）

1.内存布局回顾
mips-memory

上图是lab1中分析过的mips内存布局，需要明白这是32位体系结构的虚拟地址空间（2^32=4GB,按字节寻址）。在实验中，尤其需要注意区分地址是虚拟地址还是物理地址，虚拟地址布局可以参照上图，通过某些映射可以得到对应的物理地址，具体的映射函数根据规则的不同（例如直接连续映射还是通过MMU转换）会在下面一一介绍。

我们先接lab1关于此内存布局的解释进一步做一些解释：

kseg0：0x80000000到0x9fffffff，在lab1中我们用来放置内核，它连续映射到物理地址的低512MB，只需要将最高位置0即可得到对应的物理地址（即0x00000000到0x1fffffff）。

kseg1：0xa0000000到0xbfffffff，它用于访问外设，也是连续映射到物理地址的低512MB，需要将高三位置零得到相应物理地址，也是0x00000000到0x1fffffff。

对于mos操作系统，我们采用的管理物理内存的方式是空闲链表。即将物理内存分成一些大小固定的页面，作为内存调度的基本单位，再用链表将物理内存中空闲的页组织起来，需要使用时从链表中取出进行分配，使用完成后再插入链表。

*需要指出，和一般的空闲链表不同，实验中简化了空闲链表，以页为最小单位直接将空闲的页使用链表串联起来。

2.struck Page结构：物理内存页管理
在lab1题解中我们提到，启动跳转到的mips_init函数会在不同的lab中被替换，以实现不同的评测需求。看test目录中lab2_1下的init.c文件中的mips-init()函数，首先调用了mips_detect_memory()，mips_vm_init()，page_init()三个函数，也就是说，内核启动跳转到mips-init()函数后，首先就执行了内存管理初始化的相关函数。我们从这里开始。

首先，执行mips_detect_memory()函数，该函数探测了当前硬件物理内存的可用大小，由此计算出物理页面数量，分别记录到全局变量memsize和npage中（记住这两个变量的意义，后面我们会用到），这样我们就对物理内存有了一个大致的掌握。

随后，开始建立内存管理机制。目前，页式管理机制还未建立起来，我们需要一些其他的数据结构来暂时管理内存。在mips_vm_init()函数中，我们看到功能代码只有一行：调用了一个叫alloc的类似内存分配函数，赋值给了指向struct Page结构的指针类型变量pages。这是一个结构数组的首地址，这个结构体也是链式物理内存管理的关键。

前面提到，物理内存被我们分成了许多固定大小的页面，所谓链表法当然不是直接将这些页面串联起来，而是创建了这个结构体数组pages，每一个成员都是一个struct Page类型，它们和所有的物理页面一一对应，一个struct Page结构就管理一个物理页面。mips_vm_init()函数完成了struct Page结构体数组的内存分配：

void mips_vm_init() {
        pages = (struct Page *)alloc(npage * sizeof(struct Page), BY2PG, 1);
        printk("to memory %x for struct Pages.\n", freemem);
        printk("pmap.c:\t mips vm init success\n");
}
alloc是分配函数，返回分配空间的首地址（虚拟地址），pages于是成为struct Page结构数组的首地址，也是第一个结构体的首地址。alloc函数传入的第一个参数是分配空间的大小，可以看到为npage * sizeof(struct Page)，即页面总数*页管理结构体大小，也就是所有的struct Page所在的内存空间。


//`struct Page`结构体具体数据结构：
typedef LIST_ENTRY(Page) Page_LIST_entry_t;
struct Page {
        Page_LIST_entry_t pp_link; 
        u_short pp_ref;
};
//其中，pp_link是包含两个指针成员的结构体，也就是用于串联链表的指针部分
//pp_ref则表示这段物理内存被引用的次数，例如pp_ref为0时表示空闲
//展开来看：
struct Page{
        struct Page_LIST_entry_t{
                struct Page *le_next;
                struct Page **le_prev;
        } pp_link;
        u_short pp_ref;
}
//为什么le_prev指针多一个'*'?参看'4.链表宏'部分
3.从struct Page到物理页的地址
现在知道了用什么数据结构来管理物理页面，那么，一个特定的struct Page是如何和和一个特定的物理页的内存地址对应起来的呢？也就是说，我们如何通过Page结构体的信息知道它管理的是哪个物理页面呢？实验中，include/pmap.h文件里为我们提供了Page结构体地址和物理页首地址之间相互转换的函数：

static inline u_long page2pa(struct Page *pp)：传入Page结构体地址，返回物理页首地址（物理地址）。

static inline struct Page *pa2page(u_long pa)：传入物理页首地址（物理地址），返回Page结构体首地址。

static inline u_long page2kva(struct Page *pp)：传入Page结构体地址，返回物理页首地址对应的虚拟地址。

注：page2pa和page2kva区别只在于后者在前者基础上调用了一次KADDR，此宏定义于include/mmu.h，作用是将kseg0区的物理地址转换成对应的虚拟地址，由于是连续映射，它的操作只是简单地将物理地址延展至32位并在最高位设置为1，和它对应的是宏PADDR，它将kseg0虚拟地址转换成物理地址，只是将最高位设置位0。这就能和我们之前所说的kseg0区的虚拟地址和物理地址映射解释通了。

所以，假如我们现在能操作一个struct Page类型的结构体变量manage_page，就可以通过调用page2pa(&manage_page)得到它管理的物理页的首地址，或通过调用page2kva(&manage_page)得到它管理的物理页首地址对应的虚拟地址。请注意，上述我们提到的所有物理地址与虚拟地址之间的转换都是在kseg0段的简单连续映射，KADDR和PADDR也仅适用于kseg0区，可以前往include/mmu.h查看这两个宏的定义源码。

如果对于Page结构体到物理页面这一部分仍不理解，可以参考tips部分，一段段地详细分析了内存各部分。另外，实验中多次提到对齐的概念，可以参考这篇文章，在实验中使用宏ROUND和ROUNDDOWN来对齐，alignment值为宏BY2PG，其值为4096（字节），也就是一个页的大小，因此凡是按照BY2PG对齐的地址值其十六进制低三位一定都是0，即为0x1000的整数倍。

4.链表宏
现在，我们已经有了物理页面和管理它们的结构，只需要用链表将这些结构串起来，就完成了链式物理内存的管理。为了方便地使用链表，实验将链表写成宏放在了include/queue.h文件中，它和我们平时使用的链表没有本质区别，且实现了双向链表功能。唯一要注意的一点就是前向指针le_prev并不向后向指针le_next那样直接指向一个Page结构体，而是指向前一个Page结构体的pp_link成员中的le_next成员，也就是说，它是一个指向了前一个结构体的后向指针的指针，所以它是一个二重指针。指导书中给出了不错的图示：

link_list

include/queue.h提供了一系列好用的链表宏，建议仔细阅读，了解每一个宏的作用，好开展下面的工作。在pmap.c中，已经给出了全局结构变量struct Page_list page_free_list;，我们可以找到struct Page_list结构的定义处对其进行展开观察，这个藏得有点深，是使用LIST_HEAD宏来定义的，其成员就是一个指向Page结构体的指针lh_head。LIST_HEAD宏如下，而在include/pmap.h中，调用宏定义了Page_list结构体。

//include/queue.h宏定义
 #define LIST_HEAD(name,type)                               \
        struct name {                                       \
                struct type *lh_first; /* first element */  \
        }
//include/pmap.c调用宏来定义Page_list结构
LIST_HEAD(Page_list, Page);
结合2中Page结构的分析，可以解答指导书Thinking2.3的问题：展开Page_list结构：

struct Page_list{
        struct {  //Page结构
                struct {  //Page_LIST_entry_t结构
                        struct Page *le_next; //指向Page结构的指针
                        struct Page **le_prev; //指向'指向Page结构的指针'的指针
                } pp_link;
                u_short pp_ref;
        }* lh_first;
}
这样，我们就可以直接使用全局变量page_free_list，进行一系列链表宏的操作来管理物理内存。至此第一部分就结束了，题目就很简单了。

tips：kseg0从0x80000000开始到0x9fffffff结束，其中：
内核elf文件从0x80010000开始，到0x80400000结束；（kernel.lds中的end）
pages首地址从0x80400000开始，到0x80430000，总占用196608（16进制30000）字节，共16384个结构体（即对应16384个物理页面），每个结构体12字节。
物理内存大小为65536KB=64MB=2^26B，按字节寻址，物理地址为0x00000000到0x03ffffff。可以发现物理内存其实并没有一个kseg0或kseg1那么大，连续映射的结果是有部分地址肯定超出了。
我们可以自己写一个测试函数，看看这些Page结构体和物理页地址之间的关系：如下，选择一个test中的init.c放入测试用函数，运行得到如下结果。其中，sP_addr表示Page结构体地址（虚拟地址），也就是pages每个元素的地址；pa表示该结构体本身在物理内存中的物理地址，pp_addr表示该结构体对应管理的物理页的首地址。可以看到，这些结构体所占的物理内存是从0x400000到0x430000的48个页大小的空间。

//测试函数
void my_test_PageToPP(){
        for(int i=0;i<npage;i++){
                printk("page index %d: sP_addr=0x%x in pa 0x%x,pp_addr=0x%x\n",i,pages+i,PADDR(pages+i),page2pa(pages+i));
                if(i==9){
                        printk("...\n...\n...\n");
                        i=npage-11;
                }
        }
}
//运行结果
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
题目解析(2.1-2.5)
指导书中exercise2.1的题目描述：

请参考代码注释，补全 mips_detect_memory 函数。在实验中，从外设中获取了硬件可用内存大小 memsize，请你用内存大小 memsize 完成总物理页数 npage 的初始化。

mips_detect_memory函数代码框架：

void mips_detect_memory() {
        /* Step 1: Initialize memsize. */
        memsize = *(volatile u_int *)(KSEG1 | DEV_MP_ADDRESS | DEV_MP_MEMORY);

        /* Step 2: Calculate the corresponding 'npage' value. */
        /* Exercise 2.1: Your code here. */

        printk("Memory size: %lu KiB, number of pages: %lu\n", memsize / 1024, npage);
}
memsize的初始化已经由代码框架给出，我们只需要根据物理内存总量换算得到物理页数即可，即物理内存总量/页大小=物理页数,注意单位为字节即可。

参考代码：

        npage = memsize / BY2PG;  //或4096
指导书中exercise2.2的题目描述：

完成 include/queue.h 中空缺的函数 LIST_INSERT_AFTER。其功能是将一个元素插入到已有元素之后，可以仿照 LIST_INSERT_BEFORE 函数来实现。

照抄LIST_INSERT_BEFORE即可，只需注意一点点的le_prev二重指针问题，然后注意这是宏，结尾记得加上'\'。

参考代码：

do{                                                                                 \
        (elm)->field.le_next = (listelm)->field.le_next;                            \
        if(LIST_NEXT((listelm),field) != NULL){                                     \
                LIST_NEXT((listelm),field)->field.le_prev = &((elm)->field.le_next);\
        }                                                                           \
        LIST_NEXT((listelm),field) = (elm);                                         \
        (elm)->field.le_prev = &LIST_NEXT((listelm),field);                         \
}while(0)
指导书中exercise2.3的题目描述：

完成 page_init 函数。请按照函数中的注释提示，完成上述三个功能。此外，这里也给出一些提示：
1.使用链表初始化宏 LIST_INIT。
2.将 freemem 按照 BY2PG 进行对齐（使用 ROUND 宏为 freemem 赋值）。
3.将 freemem 以下页面对应的页控制块中的 pp_ref 标为 1。
4.将其它页面对应的页控制块中的 pp_ref 标为 0 并使用 LIST_INSERT_HEAD 将其插入空闲链表。

参考代码：

void page_init(void) {
        /* Step 1: Initialize page_free_list. */
        /* Hint: Use macro `LIST_INIT` defined in include/queue.h. */
        /* Exercise 2.3: Your code here. (1/4) */
        LIST_INIT(&page_free_list);
        /* Step 2: Align `freemem` up to multiple of BY2PG. */
        /* Exercise 2.3: Your code here. (2/4) */
        freemem = ROUND(freemem,BY2PG);
        /* Step 3: Mark all memory below `freemem` as used (set `pp_ref` to 1) */
        /* Exercise 2.3: Your code here. (3/4) */
        for(int i=0;i<npage;i++){
                if(page2kva(&pages[i]) < freemem){
                        pages[i].pp_ref = 1;
                }else{
                        pages[i].pp_ref = 0;
                        LIST_INSERT_HEAD(&page_free_list,&pages[i],pp_link);
                }
        }
        /* Step 4: Mark the other memory as free. */
        /* Exercise 2.3: Your code here. (4/4) */

}
指导书中exercise2.4的题目描述：

完成 page_alloc 函数。在 page_init 函数运行完毕后，在 MOS 中如果想申请存储空间，都是通过这个函数来申请分配。该函数的逻辑简单来说，可以表述为：
1.如果空闲链表没有可用页了，返回异常返回值。
2.如果空闲链表有可用的页，取出第一页；初始化后，将该页对应的页控制块的地址放到调用者指定的地方。
填空时，你可能需要使用链表宏 LIST_EMPTY 或函数 page2kva。

比较坑的地方就是这个返回异常值。我就疑惑这里直接告诉大家异常值就是-E_NO_MEM宏有什么不行的吗，笔者开始写这个看他指导书没告诉以为默认-1就行，debug几十分钟，最后摸到pmap.c本地测评函数那里发现原来检查异常值的时候用的这个玩意儿。纯纯谜语浪费大伙时间吧。。。

参考代码：

int page_alloc(struct Page **new) {
        /* Step 1: Get a page from free memory. If fails, return the error code.*/
        struct Page *pp;
        /* Exercise 2.4: Your code here. (1/2) */
        if(LIST_EMPTY(&page_free_list)){
                return -E_NO_MEM;
        }
        pp = LIST_FIRST(&page_free_list);
        LIST_REMOVE(pp, pp_link);

        /* Step 2: Initialize this page with zero.
         * Hint: use `memset`. */
        /* Exercise 2.4: Your code here. (2/2) */

        memset((void *)page2kva(pp), 0, BY2PG);
        *new = pp;
        return 0;
}
指导书中exercise2.5的题目描述：

完成 page_free 函数。
提示：使用链表宏 LIST_INSERT_HEAD，将页结构体插入空闲页结构体链表。

参考代码：

void page_free(struct Page *pp) {
        assert(pp->pp_ref == 0);
        /* Just insert it into 'page_free_list'. */
        /* Exercise 2.5: Your code here. */
        LIST_INSERT_HEAD(&page_free_list,pp,pp_link);
}
知识点二：采用两级页表的虚存管理
1.从一级页表到二级页表
如果你对二级页表的基本思想还不太了解，可以顺着看一下，如果已经了解，建议跳过，直接到这里

学过计组我们大概知道，每个进程中有一个页表，用来将虚拟地址按照其格式映射成物理地址。具体来说，我们将物理内存和虚拟内存空间划分成一些相同大小的页，在本实验中，页大小是宏定义BY2PG，其大小为4KB，即物理地址和虚拟地址的低12位为页内偏移。对虚拟地址而言，其高20位为页索引（页表项偏移）。给定一个基址为pg_table的页表，这样我们就能通过页索引找到对应的页表项，该页表项内存的就是物理地址值。

现在，我们以此为基础进入主题：二级页表。详细的二级页表和多级页表理论在这里不再详述，我们仅以实验内容为例。上面我们提到，采用一级页表时，页索引（虚页号）为高20位，即有2^{20}2 
20
 个页表项。每个页表项32位4字节（参考指导书给出的设定），每个进程中装一个页表（每个进程都有自己的地址空间），极大地浪费了内存空间。引入二级页表的核心思想实际上就是把这些页表项再进行分页，没错，就是按照我们此前划分的页面大小4KB来给这些页表项分页，用一个“页表项的页表”来索引这些页面。也就是说，现在这些页表项被划分成很多页后就好像我们之前要访问的物理页面，我们用一个高一级的页表来以和之前相同的形式来找其中的页，进而找到具体的页表项，这样我们就只需要在原页表项所在的这一页中寻找，而不是在所有的页表项中，节省了空间并提高了映射的效率。

在实验中，有2^{20}2 
20
 个页表项，每个4字节，即页表项共占4MB大小，我们每个页4KB，恰好可以分成(2^{22}/2^{12})=2^{10}(2 
22
 /2 
12
 )=2 
10
 个页面（1K个），需要10位来进行索引，因此我们上述提到的“更高一级的页表”，即用来索引这个页表项页面的页表需要10位作为索引位，这也就是第一级页表的索引位数。再继续看这些分成1K个页面的页表项：每个页表项4B，页大小4KB，因此一页恰好是1K=1024个项，次高10位用作为页内偏移来寻找具体页表项，这也就是第二级页表的索引位数。

2_level_pagetable


指导书中的这个图就较清晰地反映了我们实验用到的二级页表结构。一级页表和二级页表所占用的空间都恰好是一页即4KB，有1024个表项。注意，图中一级页表和二级页表每一个表项都标注的是由“物理页号”和“权限位”组成，但是只有二级页表表项是我们需要从虚拟地址映射到物理地址的“物理页号”，而一级页表表项的“物理页号”，指的是它需要索引的对象——某个二级页表的物理页号。他们的共同点是，它们都确实是物理内存中一个页面的物理基址，区别是一级页表索引这1024个二级页表所在页，二级页表索引映射关系的物理页。

2.映射过程
经过上面的解释，有没有发现一级页表就好像专门用来寻找二级页表的“目录”，因此为了便于区分，和指导书一样，下文中用页目录来指代一级页表，用页表来指代二级页表，可能出现的英文变量名分别为pg_dir和pg_tab。

如果一个虚拟地址va和一个物理地址pa已经建立好了映射关系，且已知页目录的基址pg_dir，那么我们映射的过程就应该表示为如下的步骤：

找到va的高10位，即页目录项索引，记为pgdir_off，则pg_dir+pgdir_off即得到具体的页目录项。我们将其记为pgdir_entry。

pgdir_entry是一个页目录项，本质是一个32位的无符号二进制数，在实验中我们用u_long无符号长整形通过typedef定义了一个叫Pde的类型。第二步就是找出它的高二十位得到物理页号，这个物理页号就是页表的页号，是我们要寻找的具体页表项所在的页表基址，将其记为pg_tab。

与第一步类似，我们需要找到va次高10位（21~12），即页表项索引，记为pgtab_off，则pg_tab+pgtab_off即得到具体的页表项。我们将其记为pgtab_entry。

pgtab_entry的物理页号（高20位），即为映射的物理页号，也就是我们需要寻找到的物理地址所在物理页的基址。通过va低12位页内偏移，就可以找到对应的物理地址了。

这个过程很简单，但遗憾的是我们并不是一开始就有这移，就可以找到对应的物理地址了。

这个过程很简单，但遗憾的是我们并不是一开始就有这些页表给我们映射的，映射关系建立之前显然是无法完成如此简单的映射的，我们在lab2第二部分需要做的就是建立这样的映射关系。

#### 3.实验解释

反应上述映射过程的函数是`page_lookup`：

`struct Page * page_lookup(Pde *pgdir, u_long va, Pte **ppte)`

给定虚拟地址`va`和页目录基址`pgdir`，该函数返回虚拟地址映射的物理地址所在页的管理结构Page结构体的指针。

建立映射的函数是`pgdir_walk`和`page_insert`。具体来说，`pgdir_walk`检查页目录项物理页号对应的页表是否存在，并根据传入参数*选择性*（参数`create`）创建页表。`page_insert`给定虚拟地址和物理地址所在页的管理结构Page的指针，通过调用`pgdir_walk`来建立这二者之间的映射。也就是说，`page_insert`就是实现从`va`到`pp`（Page Pointer）的映射的函数，其中调用了`pgdir_walk`来完成检索页目录项和创建页表的工作。

我们要完成的就是这两个用于建立映射的函数。

<a id="function1"></a>

**（1）**`int pgdir_walk(Pde *pgdir, u_long va, int create, Pte **ppte)`

参数解释：`pgdir`为页目录基址，即指向页目录第一项的指针；`va`为给定的虚拟地址；`create`指定在没有检索到页表时是否需要创建；`ppte`为指向**找到的（或创建的）页表项地址**的指针，其表现为二重指针（因为页表项地址就是一个指针），事实上在**调用函数**传参时一般将页表项指针取地址后传入，目的是在此函数中改变这个指针值。

函数的逻辑流是，首先根据页目录偏移（索引）得到页目录项，检查页目录项有效位（宏`#define PTE_V 0x0200`和页目录项做按位'与'运算）是否为1：

* 若为1，说明页目录检索成功，页表存在；

* 若为0，检索失败，不存在页表，判断参数`create`的值：
  
  * 若`create`为1，需要创建一个页表与页目录联系起来。

  * 若`create`为0，将`ppte`指向的值，即`*ppte`，页表项的地址置为`NULL`或0。（这里可以查看关于C语言中NULL值的定义，可以看[这篇](https://blog.csdn.net/weixin_44006265/article/details/109656285)参考文章），然后正常返回`return 0`。

分支结束后，将页表项地址取地址交由`ppte`，在上面的分支中，页表项地址不存在的情况只有一种，且已正常返回0，这样才能保证这一步不发生访问位置地址或者空指针的错误。

<a id="function2"></a>

**(2)**`int page_insert(Pde *pgdir, u_int asid, struct Page *pp, u_long va, u_int perm)`

参数解释：涉及核心功能的参数是`pgdir`、`pp`、`va`，表示给定页目录基址`pgdir`，将虚拟地址`va`和结构体指针`pp`所管理的物理页建立起映射关系。

函数的逻辑流是，首先调用一次`pgdir_walk`，`create`暂为0，检索页目录是否已经有页表目标：

* 若已经有对应的页表项，检查这个页表项的物理页是否与我们传入的`pp`管理的物理页相同：
 
  * 若相同，说明我们需要建立的映射本身存在，只需要将权限位和有效位置1正常返回；

  * 若不同，说明本身的映射和我们需要建立的不相同，需要删去这个映射。

* 若没有对应的页表项，需要建立从页目录到页表的对应关系，具体来说就是再次调用`pgdir_walk`，这一次需要将`create`参数传入1，以此来创建页表，并将其对应到页目录项。最后将分配的物理页页号填入页表项，设置页表项权限位即可。


### 题目解析（2.6&2.7）

>指导书exercise2.6题目描述：

>完成 pgdir_walk 函数。
该函数的作用是：给定一个虚拟地址，在给定的页目录中查找这个虚拟地址对应的物理地址，如果存在这一虚拟地址对应的页表项，则返回这一页表项的地址；如果不存在这一虚拟地址对应的页表项（不存在这一虚拟地址对应的二级页表、即这一虚拟地址对应的页目录项为空或无效），则根据传入的参数进行创建二级页表，或返回空指针。注意，这里可能会在页目录表项无效且 create 为真时，使用page_alloc 创建一个页表，此时应维护申请得到的物理页的 pp_ref 字段。 

思路参考[上面](#function1)函数逻辑分析。

参考代码：

```c
static int pgdir_walk(Pde *pgdir, u_long va, int create, Pte **ppte) {
	Pde *pgdir_entryp;
	struct Page *pp;

	/* Step 1: Get the corresponding page directory entry. */
	/* Exercise 2.6: Your code here. (1/3) */
	
	pgdir_entryp = pgdir + PDX(va);

	/* Step 2: If the corresponding page table is not existent (valid) and parameter `create`
	 * is set, create one. Set the permission bits 'PTE_D | PTE_V' for this new page in the
	 * page directory.
	 * If failed to allocate a new page (out of memory), return the error. */
	/* Exercise 2.6: Your code here. (2/3) */

	if( ((*pgdir_entryp) & ((Pde)PTE_V)) == 0 ){ //若此页目录有效位为0，即不对应页表
	        if(create == 1){
			if(page_alloc(&pp) == 0){ //分配，此后pp管理分配的一页内存
                        	pp->pp_ref = 1;
				*pgdir_entryp = page2pa(pp) | (Pde)(PTE_D | PTE_V);
                	}else{
                        	return -E_NO_MEM;
                	}
		}else{
			*ppte = NULL;
			return 0;
		}
	}

	/* Step 3: Assign the kernel virtual address of the page table entry to '*ppte'. */
	/* Exercise 2.6: Your code here. (3/3) */
	
	*ppte = (Pte *)(KADDR(PTE_ADDR(*pgdir_entryp)))+PTX(va);
	return 0;
}
```

对于可能出现的疑惑：

* 最后一步为什么要对`PTE_ADDR(*pgdir_entryp)`再调用一次`KADDR`？

> 因为`PTE_ADDR()`的作用是取出页目录（或者页表）项的物理页号，这个地址是kseg0中的真实物理地址，而我们要用来寻找页表（即二级页表页号）时，使用的页表基址是虚拟地址。因此需要将这个物理页号转换成虚拟页号，这样就得到了页表基址，再加上`PTX(va)`来得到具体的页表项地址（也是个虚拟地址）。

* 最后一步赋值给`*ppte`时，为什么不直接用`page2kva(pp)`得到页表项的虚拟地址，而是要从页目录项的物理页号转换成虚拟基址再加上页表项偏移得到呢？

> 根据上述逻辑流程，并不是所有的条件分支中页控制块`pp`都起了作用。首先如果这个`va`是有效的，最外层的`if`根本不会进入，其次，参数`create`为0时，也不会进行页表的空间分配。


>指导书exercise2.6题目描述：

>Exercise 2.7 完成 page_insert 函数（补全 TODO 部分）。 

思路参考[上面](#function2)函数逻辑分析。

参考代码：

```c
int page_insert(Pde *pgdir, u_int asid, struct Page *pp, u_long va, u_int perm) {
	Pte *pte;
	
	/* Step 1: Get corresponding page table entry. */
	pgdir_walk(pgdir, va, 0, &pte);
	
	if (pte && (*pte & PTE_V)) {
		if (pa2page(*pte) != pp) {
			page_remove(pgdir, asid, va);
		} else {
			tlb_invalidate(asid, va);
			*pte = page2pa(pp) | perm | PTE_V;
			return 0;
		}
	}

	/* Step 2: Flush TLB with 'tlb_invalidate'. */
	/* Exercise 2.7: Your code here. (1/3) */
	
	tlb_invalidate(asid, va);

	/* Step 3: Re-get or create the page table entry. */
	/* If failed to create, return the error. */
	/* Exercise 2.7: Your code here. (2/3) */


	if(pgdir_walk(pgdir, va, 1, &pte)!=0){
		return -E_NO_MEM;
	}

	/* Step 4: Insert the page to the page table entry with 'perm | PTE_V' and increase its
	 * 'pp_ref'. */
	/* Exercise 2.7: Your code here. (3/3) */
	
	*pte = page2pa(pp) | perm | PTE_V;
	pp->pp_ref++;
	return 0;
}
```

在step1中（题目已给出的代码部分），很显然它进行的就是思路中第一个条件分支：判断是否已经存在映射，然后针对映射是否符合要求再分支处理。对于下面待补充代码里面一些还未提及的函数，我们也可以暂时仿照给出的代码来完成（比如那个`tlb_invalidate`）。

另外，我们再介绍一下`page_lookup`函数，这个函数不需要我们补全，但是后续会使用到，所以需要简单了解一下它的功能。
```c
struct Page *page_lookup(Pde *pgdir, u_long va, Pte **ppte) {
	struct Page *pp;
	Pte *pte;

	/* Step 1: Get the page table entry. */
	pgdir_walk(pgdir, va, 0, &pte);

	/* Hint: Check if the page table entry doesn't exist or is not valid. */
	if (pte == NULL || (*pte & PTE_V) == 0) {
		return NULL;
	}

	/* Step 2: Get the corresponding Page struct. */
	/* Hint: Use function `pa2page`, defined in include/pmap.h . */
	pp = pa2page(*pte);
	if (ppte) {
		*ppte = pte;
	}

	return pp;
}
```

它传入了页目录基址、虚拟地址和页表项地址的指针，并检查虚拟地址是否映射到了某个页表项，若没有，返回空，若有，返回页表项物理页号对应物理页的管理块`pp`，并将页表项地址值交给传入的指针。

### 知识点三：TLB

点击跳转到[题解](#题目解析28-210)

#### 1. TLB简介

学过计组的我们知道（计组真是个好东西），上述我们介绍的所有的地址转换过程，其实并不一定会发生，在此之前我们会先用快表TLB（Translation Lookaside Buffer，地址转换后援缓冲器）来尝试直接找到从虚拟地址到页表项的联系，就和cache一样，我们不直接从内存去访问数据，而是先尝试在cache中寻找，若能直接命中cache，则可以让访问速度变快很多，这是由计算机存储体系结构所决定的。TLB实际上也是一个小型cache，因为页表项本身是放在内存中的，进行内存转换时，我们不直接通过两级页表去内存中寻页表项，而是先看看TLB看看能不能直接命中，如果命中，就省去了从主存中访问页表项的过程。简化地类比：**cache是访问数据时的缓存，而TLB是转换地址时的缓存。**

![TLB](/img/TLB.jpg)

在实验中，一个TLB表项共64位，由`key`和`Data`两个部分组成，每个部分32位。`EntryHi`和`EntryLo`是两个寄存器，它们分别对应TLB的这两个部分（`Key`是高32位，`Data`是低32位），顾名思义，这两个寄存器就是用来访问TLB Entry的寄存器，因为TLB一项有64位所以要两个寄存器来完成这个工作。下面所有TLB的读写操作都要通过这两个寄存器。

TLB的高32位可以理解为它的索引位或键（Key），其高20位为VPN（Virtual Page Number）虚页号，即`va`的高20位，在前面我们将其用作页索引。低12位中6-11位是ASID（Address Space IDentifier），简单地说就是，因为每个进程都有其地址空间，那么相同的虚拟地址也就可以映射到不同的物理地址（在之前不会发生这种情况是因为我们是提供了具体的页目录），所以需要有一个识别码来区分不同的地址空间。

指导书中两次提及，页表项的标志位（低12位）规范与EntryLo寄存器的规范相同，实际上也就是说TLB一项的`Data`部分就是存放的一个页表项。低12位是标志位，高20位为PFN（Page Frame Number），也就是物理页号（页帧号）。由此我们可知TLB中的Data部分相当于一个页表项的副本，若操作系统对页表项进行了更改，而对应的TLB表项低32位没有随之更新的话，在通过TLB访问物理地址时就会发生错误，因此，我们需要对TLB的表项进行维护。

![TLB_Entry_Rigister](/img/TLB_Entry.jpg)

这样一来，我们的地址映射功能就闭环了：

在用户地址空间访存时，虚拟地址到物理地址的转换均通过 TLB 进行。只有当前虚拟地址的页号和当前进程 ASID 组成的Key 在TLB 中存在对应的 TLB 表项（或虚页号在 TLB 中存在对应的 TLB 表项且表项权限位中的 G 位为 1）时，才能找到对应的物理地址；若没有找到，则产生TLB Miss异常，跳转到异常处理程序中，在页表中找到对应的物理地址（我们上面的二级页表映射过程），对 TLB 进行重填。

#### 2.函数实现

关于内存操作的细节，涉及到几个寄存器：

![Register](/img/Register.jpg)

现在来看一个在`page_insert`和`page_remove`中都用到的一个函数：`tlb_invalidate`，表示将TLB表项无效化。在`page_insert`中，建立映射需要将旧的TLB表项先删除，因此用到了这个函数。它的功能实现依赖于函数`tlb_out`，其代码在`kern/tlb_asm.S`中，通过汇编实现了TLB表项的清空。具体来说：

当 `tlb_out` 被` tlb_invalidate` 调用时, `a0` 寄存器中存放着传入的参数，寄存器值为旧表项的 Key(由虚拟页号和 ASID 组成)。我们使用 `mtc0 `将 Key写入` EntryHi`，随后使用 `tlbp` 指令，根据 `EntryHi `中的 Key 查找对应的旧表项，将表项的索引存入` Index`。如果索引值大于等于 0（即 TLB 中存在 Key 对应的表项），我们向 `EntryHi` 和`EntryLo `中写入 0, 随后使用 tlbwi 指令，将 `EntryHi` 和 `EntryLo `中的值写入索引指定的表项。此时旧表项的 Key 和 Data 被清零，实现将其无效化。

TLB的重填是靠`kern/tlb_asm.S`中的`do_tlb_refill`和`kern/tlbex.c`中的`_do_tlb_refill`来实现的。具体来说：

* 从 `BadVAddr` 中取出引发 TLB 缺失的虚拟地址。

* 从 `EntryHi` 的 6 – 11 位取出当前进程的 ASID。在 Lab3 的代码中，会在进程切换时修改`EntryHi` 中的 ASID，以标识访存所在的地址空间。

* 以虚拟地址和 ASID 为参数，调用 `_do_tlb_refill` 函数。该函数是 TLB 重填过程的核心，其功能是根据虚拟地址和 ASID 查找页表，返回包含物理地址的页表项。为了保存汇编函数现场中的返回地址，在调用函数前，我们将 `ra` 寄存器的值保存在一个“全局变量”中。

* 将物理地址存入 `EntryLo` , 并执行 `tlbwr` 将此时的 `EntryHi` 与 `EntryLo` 写入到 TLB 中（在发生 TLB 缺失时，`EntryHi` 中保留了虚拟地址相关信息）。


### 题目解析（2.8-2.10）

> 2.8 和 2.10分别插入`tlbp` `tlbwi` 和`tlbwr`即可。

> 2.9：完成 kern/tlbex.c 中的 _do_tlb_refill 函数。 

参考代码：

```c
Pte _do_tlb_refill(u_long va, u_int asid) {
	Pte *pte;
	/* 提示:
                尝试在循环中调用'page_lookup'以查找虚拟地址 va
                在当前进程页表中对应的页表项'pte'
                如果'page_lookup'返回'NULL'，表明'pte'找不到，使用'passive_alloc'
                为 va 所在的虚拟页面分配物理页面，
                直至'page_lookup'返回不为'NULL'则退出循环。

                你可以在调用函数时，使用全局变量 cur_pgdir 作为其中一个实参
	 */

	/* Exercise 2.9: Your code here. */
	
	struct Page *pp = 0;
	while (!pp){
		if((pp = page_lookup(cur_pgdir, va, &pte)) != 0 ) break;
		passive_alloc(va, cur_pgdir, asid);
	}
	return *pte;
}
```