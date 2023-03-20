## OS lab2
**建议配合指导书同时使用**

***

lab1回顾：学过计组我们知道，程序中使用的地址都是虚拟地址，在lab1内核实验中我们了解到，由于MMU是受OS管理的，必须有在内核启动前就能使用的内存空间，用以放置内核，并建立内存管理体系。MIPS内存布局中，kseg0和kseg1正是这样两个内存空间，它们和物理内存之间的映射方式是直接映射，只需要简单地对高位进行赋0即可直接对应到相应的物理内存。

lab2实现了mos操作系统的内存管理。

### 知识点一：链表法管理物理内存

<font color=grey>（可以直接点击跳到</font>[题目部分](#题目解析)）

![mips-memory](/img/mips%E8%99%9A%E6%8B%9F%E5%9C%B0%E5%9D%80%E7%A9%BA%E9%97%B4.png)

上图是lab1中分析过的mips内存布局，需要明白这是32位体系结构的虚拟地址空间（2^32=4GB,按字节寻址）。在实验中，尤其需要注意区分地址是**虚拟地址**还是**物理地址**，虚拟地址布局可以参照上图，通过某些映射可以得到对应的物理地址，具体的映射函数根据规则的不同（例如直接连续映射还是通过MMU转换）会在下面一一介绍。

我们先接lab1关于此内存布局的解释进一步做一些解释：

* **kseg0**：`0x80000000`到`0x9fffffff`，在lab1中我们用来放置内核，它连续映射到物理地址的低512MB，只需要将**最高位**置0即可得到对应的物理地址（即`0x00000000`到`0x1fffffff`）。

* **kseg1**：`0xa0000000`到`0xbfffffff`，它用于访问外设，也是连续映射到物理地址的低512MB，需要将**高三位**置零得到相应物理地址，也是`0x00000000`到`0x1fffffff`。

对于mos操作系统，我们采用的管理物理内存的方式是**链表法**。即将物理内存分成一些大小固定的页面，作为内存调度的基本单位，再用链表将物理内存中空闲的页组织起来，需要使用时从链表中取出进行分配，使用完成后再插入链表。

在lab1题解中我们提到，启动跳转到的`mips_init`函数会在不同的lab中被替换，以实现不同的评测需求。看`test`目录中`lab2_1`下的`init.c`文件中的`mips-init()`函数，可以首先调用了`mips_detect_memory()`，`mips_vm_init()`，`page_init()`三个函数，也就是说，内核启动跳转到`mips-init()`函数后，首先就执行了内存管理初始化的相关函数。我们从这里开始。

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

现在知道了用什么数据结构来管理物理页面，那么，一个特定的`struct Page`是如何和和一个特定的物理页的内存地址对应起来的呢？也就是说，我们如何通过Page结构体的信息知道它管理的是哪个物理页面呢？实验中，`include/pmap.h`文件里为我们提供了**Page结构体地址**和**物理页首地址**之间相互转换的函数：

* `static inline u_long page2pa(struct Page *pp)`：传入Page结构体地址，返回物理页首地址（物理地址）。

* `static inline struct Page *pa2page(u_long pa)`：传入物理页首地址（物理地址），返回Page结构体首地址。

* `static inline u_long page2kva(struct Page *pp)`：传入Page结构体地址，返回物理页首地址对应的虚拟地址。

<font color=red>注：`page2pa`和`page2kva`区别只在于后者在前者基础上调用了一次`KADDR`，此宏定义于`include/mmu.h`，作用是将**kseg0**区的物理地址转换成对应的虚拟地址，由于是连续映射，它的操作只是简单地将物理地址延展至32位并在最高位设置为1，和它对应的是宏`PADDR`，它将**kseg0**虚拟地址转换成物理地址，只是将最高位设置位0。这就能和我们之前所说的**kseg0**区的虚拟地址和物理地址映射解释通了。</font>

所以，假如我们现在能操作一个`struct Page`类型的结构体变量`manage_page`，就可以通过调用`page2pa(&manage_page)`得到它管理的物理页的首地址，或通过调用`page2kva(&manage_page)`得到它管理的物理页首地址对应的虚拟地址。



### 题目解析
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
        npage = memsize / 4096;
```