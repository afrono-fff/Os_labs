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

对于mos操作系统，我们采用的管理物理内存的方式是**链表法**。即将物理内存分成一些大小固定的页面，作为内存调度的基本单位，再用链表将物理内存中空闲的页组织起来，需要使用时从链表中取出进行分配，使用完成后再插入链表。

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