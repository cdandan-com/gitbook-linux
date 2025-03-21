# 03-进程的内存消耗和泄漏

### 进程的内存消耗和泄漏

* 进程的VMA
* 进程内存消耗的4个概念： vss、rss、pss和uss
* page fault的几种可能性， major 和 minor
* 应用内存泄漏的界定方法
* 应用内存泄漏的检测方法：valgrind 和 addresssanitizer

本节重点阐述 Linux的应用程序究竟消耗了多少内存？

一是，看到的内存消耗，并不是一定是真的消耗。

二是，Linux存在大量的内存共享的情况。\
动态链接库的特点：代码段共享内存，数据段写时拷贝。\
把一个应用程序跑两个进程，这两个进程的代码段也是共享的。\


当我们评估进程消耗多少内存时，就是指在用户空间消耗的内存，即虚拟地址在0～3G的部分，对应的物理地址内存。内核空间的内存消耗属于内核，系统调用申请了很多内存，这些内存是不属于进程消耗的。

#### 进程的虚拟地址空间VMA

![image](https://user-images.githubusercontent.com/87457873/127090169-f784d99c-da06-40cb-b500-ed48b1be83ac.png)

task\_struct里面有个mm\_struct指针， 它代表进程的内存资源。pgd，代表 页表的地址； mmap 指向vm\_area\_struct 链表。 vm\_area\_struct 的每一段代表进程的一个虚拟地址空间。vma的每一段，都可能是可执行程序的某个数据段、某个代码段，堆、或栈。一个进程的虚拟地址，是在0～3G之间任意分布的。

![image](https://user-images.githubusercontent.com/87457873/127090189-b6bc651a-d82d-444d-bff6-cdf6a7141069.png)

上图 提供三种方式，看到进程的VMA空间。

pmap 3474

基地址，size, 权限，

通过以上的方式，可以看到进程的虚拟地址空间，分布在0～3G，任意一小段一小段分布的。

应用程序运行起来，就是一堆各种各样的VMA。VMA对应着 堆、栈、代码段、数据段、等，不在任何段里的虚拟地址空间，被认为是非法的。

![image](https://user-images.githubusercontent.com/87457873/127090214-f0f9dfd2-7899-43ae-816e-34403c36ac1d.png)

当指针访问地址时，落在一个非法的地址，即不在任何一个VMA区域。相当于访问一个非法的地址，这些虚拟地址没有对应的物理地址。应用程序收到page fault，查看原因，访问非法位置，返回segv。

在VMA的东西，不等于在内存。调malloc申请了100M内存，立马会多出一个100M的 VMA，代表这段vma区域有r+w权限。

**应用程序访问内存，必须落在一个VMA里。其次，落在一个VMA里也不一定对。把100M的堆申请出来，100M内存页全部映射为0页。页表里每一页写的只读，页表和硬件对应，MMU只查页表。而在页表项中指向物理地址的权限是只读，所以在任何时候，去写其中任何一页，硬件都会发生缺页中断。**

**Linux 内核在缺页中断的处理程序，通过MMU寄存器读出发生page fault的地址和原因。发现此时page fault的原因是写一个页表里记录只读的物理地址，而vma记录的虚拟地址又是r+w，此时，linux会申请一页内存。同时把页表中的权限改为r+w。**

总结：\
Linux 内核通过VMA管理进程每一段虚拟地址空间和权限。一旦发生page fault，如果没有落在任何一个vma区域，会干掉。

VMA的起始地址+size，用来限定程序访问的地址是否合法。VMA中每一段的权限，是来界定访问这段地址是否使用正确的方式访问。

把所有的vma加起来，构成进程的虚拟地址空间，但这并不代表进程真实耗费的内存。拿到之后才是真实耗费的内存，RSS。耗费的虚拟内存，是VSS。

#### page fault的几种可能性

![image](https://user-images.githubusercontent.com/87457873/127090458-dafe02a1-0fc1-4e2c-959e-746b169d2487.png)

1、申请堆内存vma，第一次写，页表里的权限是R ，发生page fault，linux会去申请一页内存，此时把页表权限设置为 R+W。\
2、内存访问落在空白非法区域，程序收到segv段错误。\
3、代码段在VMA记录是R+X，此时如果对代码段执行写，程序会收到segv段错误。\


**minor 和major 缺页**

缺页，分为两种情况：主缺页 和次缺页。

主缺页 和次缺页，区别就是 申请内存时，是否需要读硬盘。前者需要。

如上图第4种情况，在代码段里执行时，出现缺页。linux申请一页内存，而且要从硬盘中读取代码段的内容，此时产生了IO，称为 major缺页。

无论是代码段还是堆，都是边执行边产生缺页中断，申请实际的内存给代码段，且从硬盘中读取代码段的内容到内存。这个过程时间比较长。

minor： malloc的内存，产生缺页中断。去申请一页内存，没有产生IO的行为。major缺页处理时间，远大于minor。

![image](https://user-images.githubusercontent.com/87457873/127090530-49373bbd-8d2e-4170-a46d-80b4b462548b.png)

#### vss、rss、pss和uss的区别

![image](https://user-images.githubusercontent.com/87457873/127090546-05a8c508-77cb-4a43-a792-7109461eeed0.png)

```
VSS - Virtual Set Size
RSS - Resident Set Size
PSS - Proportional Set Size
USS - Unique Set Size
ASAN - AddressSanitizer
LSAN - LeakSanitizer
```

如上图，中间是一根内存条。三个进程分别是1044，1045，1054, 每一个进程对应一个page table，页表项记录虚拟地址如何往物理地址转换。硬件里的寄存器，记录页表的物理地址。当linux做进程上下文切换时，页表也跟着一起切换。

![image](https://user-images.githubusercontent.com/87457873/127090569-cc475c78-4c51-4492-b228-80f25b5b6f02.png)

三个进程都需要使用libc的代码段。\
VSS ＝ 1 +2 +3\
RSS = 4 +5 +6\
PSS= 4/3 + 5/2 + 6 比例化的\
USS＝ 6 独占且驻留的

工具：smem ，查看进程使用内存的情况。\
一般来讲，进程使用的内存量，还是看PSS，强调公平性。看内存泄漏看USS 就好了。

#### 内存泄漏 界定和检测方法

界定：连续多点采样法，随着时间越久，进程耗费内存越多。

主要由内存申请和释放不是成对引起。RSS/USS曲线，

观察方法：使用smem工具查看多次进程使用内存，USS使用量。

检查工具：\
1、valgrind ，会跑一个虚拟机，运行时检查进程的内存行为。会放慢程序的速度。不需要重新编译程序。\
2、addressanitizer，需要重新编译程序。编译时加参数，-fsanitize\
gcc 4.9才支持，只会放慢程序速度2～3倍。
