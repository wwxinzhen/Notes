#### 1、32位内存布局

<img src="https://img-blog.csdn.net/2018041513242953?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pfcnlhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70" alt="img"  />

​	栈至顶向下扩展，堆至底向上扩展，`mmap`映射区域至顶向下扩展。`mmap`映射区域和堆相对扩展，直至耗尽虚拟地址空间中的剩余区域。

#### 2、brk() & sbrk() & mmap()

```c++
int brk(const void *addr)
void* sbrk(intptr_t incr)
两者的作用都是扩展heap的上界
brk()的参数设置为新的brk上界地址，成功返回1，失败返回0；
sbrk()的参数为申请内存的大小，返回heap新的上界brk的地址
void* mmap(void* addr, size_t length, int prot, int flags, int fd, off_t offset)
int munmap(void& addr, size_t length)
mmap第一种用法是映射此盘文件到内存中
mmap第二种用法是匿名映射，不映射此盘文件，而向映射区申请一块内存。malloc使用的是第二种用法
munmap用于释放内存
```

#### 3、allocator

​	allocator，内存分配器，处在用户程序和内核之间。响应用户的分配请求，向操作系统申请内存，然后将其返回给用户程序。为了保持高效的分配，allocator一般都会预先分配一块大于用户请求的内存，并通过算法管理内存，以达到用户的内存分配需求。用户free掉的内存，也不是立即返还给操作系统，而是由allocator管理。

#### 4、malloc实现原理

​	因为`brk`,`sbrk`,`mmap`都属于系统调用，如果每次申请内存都调用他们，会产生系统调用，影响性能；其次，这样申请的内存容易产生碎片，因为堆是从低地址到高地址，如果高地址的内存没有被释放，低地址的内存就不能被回收。

​	所以`malloc`采用的是内存池的管理方式（`ptmalloc`），`ptmalloc`采用**边界标记法**将内存划分为很多块，从而对内存的分配与回收进行管理。为了内存分配函数`malloc`的高效性，`ptmalloc`会预先向操作系统申请一块内存供用户使用，当我们申请和释放内存时，`ptmalloc`会将这些内存管理起来，并通过一些策略来判断是否将其回收给操作系统。这样做的最大好处是，用户申请和释放内存的时候更高效，避免产生过多的内存碎片。

#### 5、chunk内存块的基本组织单元

```c++
//ptmalloc源码中定义结构体malloc_chunk来描述这些块
struct malloc_chunk
{
  INTERNAL_SIZE_T      prev_size;    /* Size of previous chunk (if free).  */  
  INTERNAL_SIZE_T      size;         /* Size in bytes, including overhead. */  
  
  struct malloc_chunk* fd;           /* double links -- used only if free. */  
  struct malloc_chunk* bk;  
  
  /* Only used for large blocks: pointer to next larger size.  */  
  struct malloc_chunk* fd_nextsize;      /* double links -- used only if free. */  
  struct malloc_chunk* bk_nextsize; 
};
	prev_size:如果前一个chunk是空闲的，该域表示前一个chunk的大小，如果chunk不空闲，该域无意义。（知道当前chunk地址，减去prev_size，便得到前一个chunk的地址，prev_size主要用于相邻空闲的chunk合并）
	size：当前chunk的大小，并且记录了当前chunk和前一个chunk的一些属性，包括前一个chunk是否在使用中，当前chunk是否通过mmap获得的内存，当前chunk是否属于非主分配区。
    fd & bk：只有该chunk空闲时才会存在，其作用是用于将对应的空闲chunk块加入到空闲chunk块链表中统一管理，如果该chunk块被分配给应用程序使用，那么这两个指针也就没有用，所以也被当作应用程序的使用空间。
    fd_nextsize & bk_nextsize：当前chunk存在于large bins中，large bins中的空闲chunk是按照大小排序，如果存在多个同一大小的chunk，增加这两个字段可以加快遍历空闲chunk，并查找满足需要的空闲chunk，fd_nextsize指向下一个比当前chunk大的第一个空闲chunk，bk_nextsize指向前一个比当前chunk小的第一个空闲chunk。如果该chunk块被分配给应用程序使用，那么这两个指针也就没有用，所以也被当作应用程序的使用空间。
```

#### 6、chunk的结构

##### 	a、使用中的chunk

​	<img src="https://img-blog.csdn.net/2018041513381475?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pfcnlhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70" alt="img" style="zoom: 67%;" />

​	chunk指针指向chunk开始的地址；`mem`指针指向用户内存块开始的地址。

​	P = 0时，表示前一个chunk为空闲，`prev_size`才有效

​	P = 1时，表示前一个chunk正在被使用，`prev_size`无效。p主要用于内存块的合并操作。`ptmalloc`分配的第一个块总是将p设为1，以防程序引用到不存在的区域。

​	M = 1为`mmap`映射区域分配；M = 0为heap区域分配

​	A = 0为主分配区分配；A = 1为非主分配区分配

##### 	b、空闲的chunk

​	<img src="https://img-blog.csdn.net/20180415134021151?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pfcnlhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70" alt="img" style="zoom:50%;" />

​	chunk空闲时，M状态不存在，只有AP状态。因为M表示是由`brk`还是`mmap`分配的内存，而`mmap`分配的内存free时直接`munmap`，不会放到空闲链表。

​	原本是用户数据区的地方存储了四个指针。指针`fd`指向后一个空闲chunk，而bk指向前一个空闲的chunk，`malloc`通过这两个指针将大小相近的chunk连成一个双向链表。   

##### 	c、chunk中的空间复用

​	为了使chunk所占用的空间最小，`ptmalloc`使用了空间复用，一个chunk在不同状态下，某些区域表现出来不同的意义，以此达到复用。空闲时，一个chunk至少需要4个`size_t`大小的空间，用来存储`prev_size`，`size`，`fd`和`bk`，也就是`16bytes`。chunk的大小要对齐到`8bytes`。当一个chunk处于使用状态时，它的下一个chunk的`prev_size`域肯定是无效的。所以实际上，这个空间也可以被当前chunk使用。所以，一个使用中的chunk的大小的计算公式为：

​					`in_use_size = （用户请求大小 + 8 - 4）`

这里加`8bytes`是因为存储`prev_size`和`size`，但又因为向下一个chunk借了`4bytes`，所以减去4。最后，因为空闲的chunk和使用中的chunk使用的是同一块空间，所以要取最大值作为实际的分配空间，即最终的分配空间为`chunk_size = max(in_use_size, 16)`。

#### 7、空闲链表bins

​	当用户free掉内存，`ptmalloc`并不会马上交还给操作系统，而是被`ptmalloc`本身的空闲链表bins管理起来，这样当下次进程需要`malloc`一块内存时，`ptmalloc`就会从空闲的bins上寻找一块合适的内存块分配给用户使用。这样可以避免频繁的系统调用，降低内存分配的开销。

​	`malloc`将相似大小的chunk用双向链表连接起来，这样一个链表被称为一个bin，`ptmalloc`一共维护了128个bin，每个bin都维护了大小相近的双向链表的chunk。基于chunk的大小，有下列几种可用bins：

<img src="D:\wuxinzhen1\Desktop\bins.png" alt="bins" style="zoom:50%;" />



​	在32位平台下，bin[0]和bin[127]不存在。bin[1]为unsorted bins，bin[2]~bin[126]为sorted bins。

##### 	a、unsorted bins

​	unsorted bin的队列位于bins数组的第2个，下标为1，是bins的一个缓冲区，加快分配的速度。当用户释放的内存大于max_fast或者fast bins合并后的chunk都会首先进入unsorted bin上，chunk大小无限制，任何大小chunk都可以添加进入这里。这种途径给予`glibc malloc`第二次机会以重新使用最近free掉的chunk，这样寻找合适的bin的时间开销就抹掉了，因此分配和释放更快。

​	用户`malloc`时，如果fast bins中没有找到合适的chunk，则`malloc`会先在unsorted bin中查找合适的空闲chunk。如果没有有合适的bin，`ptmalloc`会将unsorted bin上的chunk放入bins上，然后bins上查找合适的空闲chunk。

##### 	b、small bins

​	小于`512bytes`的chunk被称为small chunk，而保存small chunks的bin被称为small bin。下标从2开始，到63结束，一共62个。small bins每个bin之间相差`8bytes`，同一个small bin的chunk具有相同大小。

​	每个small bin都包括一个空闲区块的双向循环链表，free掉的chunk添加在链表的前端，而所需chunk则从链表后端摘除。

​	两个毗连的空闲chunk会被合并成一个空闲chunk。合并消除了碎片化的影响但是减慢了free的速度。

​	分配时，当small bin非空后，相应的bin会摘除`binlist`中最后一个chunk并返回用户。在free一个chunk的时候，检查其前或其后的chunk是否空闲，若是则合并，也即把他们从所属的链表中摘除合并成一个新的chunk，新的chunk会添加在unsorted bin链表的前端。

##### 	c、large bins

​	大于`512bytes`的chunk被称为large chunk，而保存为large chunks的bin被称为large bin，位于small bins后面。下标从64开始，到126结束，一共63（32+16+8+4+2+1）个。large bins中的每个bin分别包含了一个给定范围内的chunk，其中的chunk按大小递减排序，大小相同则按照最近使用时间排列。

​	两个毗邻的空闲chunk会被合并成一个空闲chunk。

​	分配时，遵循“smallest-first，best-fit"，从顶部遍历到底部以找到一个大小最接近用户需求的chunk。一旦找到，相应chunk就会分成两块user chunk（用户请求大小）返回给用户。remainder chunk剩余部分添加到unsorted bin。free和small bin类似。

##### 	d、fast bins

​	程序在运行时会经常需要申请和释放一些较小的内存空间。当分配器合并了相邻的几个小的chunk之后，也许马上就会有另一个小块内存的请求，这样分配器又需要从大的空闲内存中切分出一块，比较低效。故，引入fast bins

<img src="https://img-blog.csdn.net/20180415152501972?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pfcnlhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70" alt="img" style="zoom:50%;" />

​	fast bins是bins的高速缓冲区，大约有10个定长队列(bin)。每个fast bin都记录着一条free chunk的单链表（`binlist`，采用单链表是因为fast bin中链表的chunk不会被摘除的特点），增删chunk都发生在链表的前端。

​	当用户释放一块不大于max_fast（默认值为`64B`）的chunk时，会默认放到fast bins上。当需要给用户分配的chunk小于等于max_fast时，`malloc`首先会到fast bins上寻找是否有合适的chunk。一定大小内的chunk无论是分配还是释放，都会在fast bin中过一遍。

​	分配时，`binlist`中被检索的第一个chunk将被摘除并返回给用户，free掉的chunk将被添加在索引到的`binlist`前端。

#### 8、三种特殊的chunk

##### 	a、top chunk

​	top chunk相当于分配区的顶部空闲内存（可能就是由`brk`调用控制的`brk`指针），当bins都不能满足内存分配要求时，就会来top chunk上分配。

​	当top chunk大小比用户所请求大小还大的时候，top chunk会分为两个部分，user chunk和remainder chunk（剩余大小）。其中remainder chunk成为新的top chunk。

​	当top chunk大小小于用户所请求的大小时，top chunk就通过`sbrk`（main arena)或`mmap`（thread arena)系统调用来扩容。

##### 	b、mmaped chunk

​	当分配的内存非常大（大于分配阈值，默认`128k`）的时候，需要被`mmap`映射，则会放到`mmaped chunk`上，当释放`mmaped chunk`上的内存的时候会直接交还给操作系统。（chunk中M标志位为1）

##### 	c、last remainder chunk

​	last remainder chunk是另外一种特殊的chunk，就像top chunk和`mmaped chunk`一样，不会在任何bins中找到这种chunk，当需要分配一个small chunk，但在small bins中找不到合适的chunk，如果last remainder chunk的大小小于所需要的small chunk大小，last remainder chunk被分裂成两个chunk，其中一个chunk返回给用户，另一个chunk变成新的last remainder chunk。

#### 9、sbrk & mmap

​	在堆区中，`start_brk`指向heap的开始，而`brk`指向heap的顶部。可以使用`brk()&sbrk()`来增加分配给用户的heap空间。在使用`malloc`前，`brk`的值等于start_`brk`，也就是说heap大小=0。

​	`ptmalloc`在开始时，若请求的空间小于`mmap`分配阈值（默认为`128KB`）时，主分配区会调用`sbrk`()增加一块大小为`（128KB+chunk_size）`的空间作为heap。非主分配区会调用`mmap`映射一块大小为HEAP_MAX_SIZE(32位系统默认为`1MB`，64位系统默认为`64MB`)的空间作为sub-heap。

​	当用户请求内存分配时，首先会在这个区域找一块合适的chunk给用户，当用户释放了heap中的chunk时，`ptmalloc`又会使用`fastbin`和bins来组织空闲chunk。

​	若需要分配的chunk大小小于`mmap`分配阈值，而heap空间又不够，则此时主分配区会通过`sbrk()`调用来增加heap大小，非主分配区会调用`mmap`映射一块新的sub-heap，也就是增加top-chunk的大小，每次heap增加的值都会对齐到`4KB`。当用户的请求超过`mmap`分配阈值，并且主分配区使用`sbrk()`分配失败的时候，或是非主分配区在top chunk中不能分配到需要的内存时，`ptmalloc`会尝试使用`mmap()`直接映射一块内存到进程内存空间。使用`mmap()`直接映射的chunk在释放时直接接触映射，而不再属于进程的内存空间。任何对该内存的访问都会产生段错误。而在heap中或是sub-heap中分配的空间则可能会留在进程内存空间内，开可以再次引用。

#### 10、主分配区和非主分配区

​	内存分配器中，为了解决多线程锁争夺问题，分为主分配区和main_area（分配区的本质就是内存池，管理着chunk）和非主分配区no_main_area.

​	（1）主分配区和非主分配区形成一个环形链表进行管理

​	（2）每个分配区利用互斥锁使线程对于该分配区的访问互斥

​	（3）每个进程只有一个主分配区，也可以允许由多个非主分配区

​	（4）`ptmalloc`根据系统对分配区的争用动态分配区的大小，分配区的数量一旦增加，则不会减少

​	（5）主分配区可以使用`brk()`和`mmap()`来分配，而非主分配区只能使用`mmap()`来映射内存块

​	（6）申请小内存时会产生很多内存碎片，`ptmalloc`在整理时也需要对分配区做加锁操作

​	当一个线程需要使用`malloc`分配内存的时候，会先查看该线程的私有变量中是否存在一个分配区，若是存在。会尝试对其进行加锁操作。如果加锁成功，就会在使用该分配区分配内存；若是失败，就会便利循环链表中获取一个未加锁的分配区。若是整个链表中都没有未加锁的分配区，则`malloc`会开辟一个新的分配区，将其加入全局的循环链表并加锁，然后使用该分配区进行分配。当释放这块内存时，同样会先获取待释放内存块所在的分配区的锁。若是有其他线程正在使用该分配区，则必须等待其他线程释放该分配区互斥锁后才能进行内存释放。

​	需要注意：

​		（1）主分配区通过`brk()`进行分配，非主分配区通过`mmap()`进行分配

​		（2）从分配区虽然是`mmap()`分配，但是和大于`128K`直接使用`mmap()`分配没有任何关系。大于`128K`的内存使用`mmap()`分配，使用完之后直接用`ummap()`还给系统

​		（3）每个线程在`malloc`会先获取一个area，使用area内存池分配自己的内存，这里存在竞争关系。

​		（4）为了避免竞争，我们可以使用线程局部存储，thread cache(`tcmalloc`中的`tc`正是此意)，改进如下：

​				a、如果需要在一个线程内部的各个函数调用都能访问、但其他线程不能访问的变量（被称为static memory local to a thread 线程局部静态变量），就需要新的机制来实现，这就是`TLS`。

​				b、thread cache本质上是在static区为每一个thread开辟一个独有的空间，因为独有，不再有竞争

​				c、每次`malloc`时，先去线程局部存储空间中找area，用thread cache中的area分配存在thread area中的chunk。当不够时，才去找栈区的area

​				d、C++11提供thread_local方便于线程局部存储

​		`tcmalloc` 和 `jemalloc`都不再使用主分配区，直接使用非主分配区

#### 11、内存分配malloc流程

```c++
1、获取分配区的锁，防止多线程冲突（一个进程有一个malloc管理器，而一个进程中的多个线程共享这一个管理器，有竞争，加锁）
2、计算出实际需要分配的内存的chunk实际大小
3、判断chunk的大小，如果小于max_fast(64B),则尝试去fast bins上取合适的chunk，如果有则分配结束。否则，下一步；
4、判断chunk的大小是否小于512B，如果是，则从small bins上去查找chunk，如果有合适的，则分配结束，否则下一步；
5、ptmalloc首先会遍历fast bins（这里是第二次遍历fast bins）中的chunk，将相邻的chunk进行合并，并链接到unsorted bin中然后遍历unsorted bins。（先遍历fast bin，遍历unsorted bin时一边遍历，一边放到small bin和large bin中）；
	如果unsorted bins只有一个chunk并且大于待分配的chunk，则进行切割，并且剩余的chunk继续扔回unsorted bins；
	如果unsorted bins有大小和待分配chunk相等的，则返回，并从unsorted bins删除
	如果unsorted bins中的某个chunk大小属于small bins的范围，则放入small bins的头部
	如果unsorted bins中的某个chunk大小属于large bins的阀内，则找到合适的位置。若未分配成功，转入下一步；
6、从large bins中查找到合适的chunk之后，然后进行切割，一部分分配给用户，剩下的放入unsorted bin中
7、如果搜索fast bins和bins都没有找到合适的chunk，那么就需要按照top chunk来进行分配了。当top chunk大小比用户所请求大小还大的时候，top chunk会分为两部分：User chunk和remainder chunk。其中remainder chunk成为新的top chunk。当top chunk大小小于用户所请求的大小时，top chunk就通过sbrk()或者mmap()系统调用来扩容。
8、top chunk也不能满足分配要求，就有两个选择：如果是主分配区，调用sbrk()。增加top chunk大小；如果是非主分配区，调用mmap来分配一个新的sub-heap，增加top chunk大小；或者使用mmap()来直接分配。在这里，需要依靠chunk的大小来决定到底使用哪种方法。判断所需分配的chunk大小是否大于等于mmap分配阈值，如果是的话，则转下一步，调用mmap分配；否则调到第10步，增加top chunk的大小。
9、使用mmap系统调用为程序的内存空间映射一块chunk_size align 4KB大小的空间。然后将内存指针返回给用户。
10、判断是否为第一次调用malloc，若是主分配区，则需要进行一次初始化工作，分配一块大小为(chunk_size + 128KB) align 4KB 大小的空间作为初始的heap。若已经初始化过了，主分配区则调用sbrk()增加heap空间，非主分配区则在top chunk中切割出一个chunk，使之满足分配需求，并将内存指针返回给用户。
```

#### 12、内存回收流程

```c++
1、获取分配区的锁，保证线程安全
2、如果free的是空指针，则返回，什么都不做
3、判断当前chunk是否是mmap映射区域映射的内存，如果是，则直接munmap()释放这块内存。前面的已使用chunk的数据结构中，我们可以看到有M来表示是否是mmap映射的内存
4、判断chunk是否与top chunk相邻，如果相邻，则直接和top chunk合并(和top chunk相邻，相当于和分配区中的空闲内存块相邻)，转到第8步
5、如果chunk的大小大于max_fast(64B)，则放入unsorted bin，并且检查是否有合并，有合并情况并且和top chunk相邻，则转到步骤8，没有合并情况则free
6、如果chunk的大小小于max_fast(64B)，则直接放入fast bin，fast bin并没有改变chunk的状态。没有合并情况，则free；有合并情况，转到步骤7
7、在fast bin，如果当前chunk的下一个chunk也是空闲的，则将这两个chunk合并，放入unsorted bin上面。合并后的大小如果大于64B，会触发进行fast bins的合并操作，fast bins中的chunk将被遍历，并与相邻的空间chunk进行合并，合并后的chunk会被放到unsorted bin中，fast bin会边为空。合并后的chunk和top chunk相邻，则会合并到top chunk中。转到步骤8
8、判断top chunk的大小是否大于mmap收缩阈值(默认为128KB)，如果是的话，对于主分配区，则会试图归还top chunk中的一部分给操作系统，free结束。
```

#### 13、使用注意事项

```c++
为了避免Glibc内存暴增，需要注意：
	1、后分配的内存先释放，因为ptmalloc收缩内存是从top chunk开始，如果与top chunk相邻的chunk不能释放，top chunk以下的chunk无法释放。
	2、ptmalloc不适合用于管理长生命周期的内存，特别是持续不定期分配和释放长生命周期的内存，这将导致ptmalloc内存暴增。
	3、不要关闭ptmalloc的mmap分配阈值动态调整机制，因为这种机制保证了短生命周期的内存分配尽量从ptmalloc缓存的内存chunk中分配，更高效，浪费更少的内存。
	4、多线程分阶段执行的程序不适合用ptmalloc，这种程序的内存更适合内存池管理（因为同一个进程下的多线程要加锁后才能使用malloc分配器）
	5、尽量减少程序的线程数量和避免频繁分配\释放内存。频繁分配，会导致锁的竞争，最终导致非主分配区增加，内存碎片增高，并且性能降低。
	6、防止内存泄漏，ptmalloc对内存泄漏相当敏感，根据它的内存收缩机制，如果与top chunk相邻的那个chunk没有回收，将导致top chunk以下很多的空闲内存都无法返回给操作系统
	7、防止程序分配过多的内存，或是由于glibc内存暴增，导致系统内存耗尽，程序因为OOM(out of memory)被系统杀死。预估程序可以使用的最大物理内存的大小，配置系统的/proc/sys/vm/overcommit_memory,/proc/sys/vm/overcommit_ratio，以及使用ulimit -v限制程序能使用的虚拟内存大小，防止因OOM被杀死
```

​	

