# 02-内存的动态申请和释放

### 内存的动态申请和释放

内核空间 和用户空间申请的内存最终和buddy怎么交互？以及在页表映射上的区别？虚拟地址到物理地址，什么时候开始映射？

#### Buddy的问题

分配的粒度太大\
buddy算法把空闲页面分成1，2，4页，buddy算法会明确知道哪一页内存空闲还是被占用？

4k，8k，16k

无论是在应用还是内核，都需要申请很小的内存。

从buddy要到的内存，会进行slab切割。

#### slab原理：

比如在内核中申请8字节的内存，buddy分配4K，分成很多个小的8个字节，每个都是一个object。

slab，slub，slob 是slab机制的三种不同实现算法。

Linux 会针对一些常规的小的内存申请，数据结构，会做slab申请。

cat /proc/slabinfo 可以看到内核空间小块内存的申请情况，也是slab分配的情况。

\<num\_objs>：每个slab一共可以分出多少个obj，\
\<active\_objs> ：还可以分配多少个obj，\
< pagesperslab>：每个slab对应多少个pages，\
< objperslab>：每个slab可以分出多少个object，\
< objsize>：每个obj多大，

slab主要分为两类：

一、常用数据结构像 nfsd\_drc， UDPv6，TCPv6 ，这些经常申请和释放的数据结构。比如，存在TCPv6的slab，之后申请 TCPv6 数据结构时，会通过这个slab来申请。

![image](https://user-images.githubusercontent.com/87457873/127087302-cc7eee3c-f9a6-4340-8b20-3663bb338a7c.png)

二、常规的小内存申请，做的slab。例如 kmalloc-32，kmalloc-64， kmalloc-96， kmalloc-128

![image](https://user-images.githubusercontent.com/87457873/127087333-f0f4f6ef-19fb-4383-9606-26471744d5a7.png)

![image](https://user-images.githubusercontent.com/87457873/127087352-9e0e0697-b1f8-45d8-949b-7a7e17b503f8.png)

注意，slab申请和分配的都是只针对内核空间，与用户空间申请分配内存无关。用户空间的malloc和free调用的是libc。

slab和buddy的关系？\
1、slab的内存来自于buddy。slab相当于二级管理器。\
2、slab和buddy在算法上，级别是对等的。

两者都是内存分配器，buddy是把内存条分成多个Zone来管理分配，slab是把从buddy拿到的内存，进行管理分配。

同理，malloc 和free也不找buddy拿内存。 malloc 和free不是系统调用，只是c库中的函数。

#### mallopt

在C库中有一个api是mallopt，可以控制一系列的选项。

![image](https://user-images.githubusercontent.com/87457873/127087421-6f665ad2-7715-4fa8-a1f7-9e855a299551.png)

M\_TRIM\_THRESHOLD：控制c库把内存还给内核的阈值。\
-1UL 代表最大的正整数。

此处代表应用程序把内存还给c库后，c库并不把内存还给内核。

<\do your RT-thing>\
程序在此处申请内存，都不需要再和内核交互了，此时程序的实时性比较高。

#### kmalloc vs. vmalloc/ioremap

内存空间： 内存+寄存器

register --> LDR/STR

所有内存空间的东西，CPU去访问，都要通过虚拟地址。\
CPU --> virt --> mmu --> phys

cpu请求虚拟地址，mmu根据cpu请求的虚拟地址，查页表得物理地址。

buddy算法，管理这一页的使用情况。

两个虚拟地址可以映射到同一个物理地址。

![image](https://user-images.githubusercontent.com/87457873/127087493-e71ab614-ceb1-4a77-9b48-6daad3bf59d9.png)

页表 -> 数组，

任何一个虚拟地址，都可以用地址的高20位，作为页表的行号去读对应的页表项。而低12位，是指页内偏移。（由于一页是4K，2^12 足够描述)

kmalloc 和 vmalloc 申请的内存，有什么区别？\
答：申请之后，是否还要去改页表。一般情况，kmalloc申请内存，不需要再去改页表。同一张页表，几个虚拟地址可以同时映射到同一个物理地址。

寄存器，通过ioremap往vmalloc区域，进行映射。然后改进程的虚拟地址页表。

总结：所有的页，最底层都是用buddy算法进行管理，用虚拟地址找物理地址。理解内存分配和映射的区别，无论是lowmem还是highmem 都可以被vmalloc拿走，也可能被用户拿走，只不过拿走之后，还要把虚拟地址往物理地址再映射一遍。但如果是被kmalloc拿走，一般指低端内存，就不需要再改进程的页表。因为这部分低端内存，已经做好了虚实映射。

```
	cat /proc/vmallocinfo |grep ioremap
```

可以看到寄存器中的哪个区域，被映射到哪个虚拟地址。

vmalloc区域主要用来，vmalloc申请的内存从这里找虚拟地址 和 寄存器的ioremap映射。

#### Linux内存分配的lazy行为

Linux总是以最lazy的方式，给应用程序分配内存。

![image](https://user-images.githubusercontent.com/87457873/127087582-ece701ed-ea7f-448c-bb2b-f5c6e37b99bf.png)

malloc 100M内存成功时，其实并没有真实拿到。只有当100M内存中的任何一页，被写一次的时候，才成功。

vss：虚拟地址空间。 rss：常驻内存空间

malloc 100M内存成功时，Linux把100M内存全部以只读的形式，映射到一个全部清0的页面。

当应用程序写100M中每一页任意字节时，会发出page fault。 linux 内核收到缺页中断后，从硬件寄存器中读取到，包括缺页中断发生的原因和虚拟地址。Linux从内存条申请一页内存，执行cow，把页面重新拷贝到新申请的页表，再把进程页表中的虚拟地址，指向一个新的物理地址，权限也被改成R+W。

调用brk 把8k变成 16k。

针对应用程序的堆、代码、栈、等，会使用lazy分配机制，只有当写内存页时，才会真实请求内存分配页表。但，当内核使用kmalloc申请内存时，就真实的分配相应的内存，不使用lazy机制。

#### 内存OOM

当真实去写内存时，应用程序并不能拿到真实的内存时。Linux启动OOM，linux在运行时，会对每一个进程进行out-of-memory打分。打分主要基于，耗费的内存。耗费的内存越多，打分越高。

```
cat /proc/<pid>/oom_score
```

demo:

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
int main(int argc, char **argv)
{
    int max = -1;
    int mb = 0;
    char *buffer;
    int i;
#define SIZE 2000
    unsigned int *p = malloc(1024 * 1024 * SIZE);
    printf("malloc buffer: %p\n", p);
    for (i = 0; i < 1024 * 1024 * (SIZE/sizeof(int)); i++) {
        p[i] = 123;
        if ((i & 0xFFFFF) == 0) {
            printf("%dMB written\n", i >> 18);
            usleep(100000);
        }
    }
    pause();
    return 0;
}
```

设定条件：

总内存1G\
1、swapoff -a 关掉swap交换\
2、echo 1 > /proc/sys/vm/overcommit\_memory\
3、内核不去评估系统还有多少空闲内存\


Linux进行OOM打分，主要是看耗费内存情况，此外还会参考用户权限，比如root权限，打分会减少30分。

还有OOM打分因子：/proc/pid/oom\_score\_adj （加减）和 /proc/pid/oom\_adj （乘除）。

![image](https://user-images.githubusercontent.com/87457873/127087921-c7f64317-9895-4b4e-93fa-4d9f44543a31.png)

#### 总结：

1、slab的作用，针对在内核空间小内存分配，和常用数据结构的申请。\
2、同样的二次分配器，在用户空间是C库。malloc和free的时候，内存不一定从buddy分配和还给buddy。\
3、kmalloc，vmalloc 和malloc的区别\


* kmalloc：申请内存，一般在低端内存区。申请到时，内存已经映射过了，不需要再去改进程的页表。所以，申请到的物理页是连续的。
* vmalloc：申请内存，申请到就拿到内存，并且已经修改了进程页表的虚拟地址到物理地址的映射。vmalloc()申请的内存并不保证物理地址的连续。
* 用户空间的malloc：申请内存，申请到并没有拿到，写的时候才去拿到。拿到之后，才去改页表。申请成功，页表只读，只有到写时，发生page fault，才去buddy拿内存。
* kmalloc和vmalloc针对内核空间，malloc针对用户空间。这些内存，可以来自任何一个Zone。
* 无论是kmalloc，vmalloc还是用户空间的malloc，都可以使用内存条的不同Zone，无论是highmem zone、lowmem zone 和 DMA zone。

4、如果在从buddy拿不到内存时，会触发Linux对所有进程进行OOM打分。当Linux出现内存耗尽，就kill一个oom score 最高的那个进程。oom\_score，可以根据 oom\_adj （-17～25）。

安卓的程序，不停的调整前台和后台进程oom\_score，当被切换到后台时，oom\_score会被调整的比较大。以保证前台的进程不容易因为oom而kill掉。
