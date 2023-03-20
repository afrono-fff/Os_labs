## OS lab2
**建议配合指导书同时使用**

***

### 知识点一：链表法管理物理内存

<font color=grey>（可以直接点击跳到</font>[题目部分](#题目解析)）

学过计组我们知道，程序中使用的地址都是虚拟地址，对物理地址访存需要通过


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